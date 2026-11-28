# 2026-11-28 - historical-snapshots-and-stock-thresholds (backend)

## Summary

Master data edits (party names, addresses, GSTINs, products, items, glass types, colors, suppliers, company details, bank accounts, and user names) previously changed historical transactions because list, detail, PDF, report, GST, and ledger queries read live master tables directly. This PR adds transactional snapshot columns and backfill migrations (`200`, `201`, `202`) along with shared snapshot builders, updates all transactional and reporting repositories to read snapshots first with live-master fallbacks, and simplifies Inventory and Order Booking stock thresholds into a single SQM-based config file with a dedicated `LOW_STOCK` filter tab.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/200_transactional_snapshots.sql` (new) | Adds JSONB and name snapshot columns to core sales, work order, dispatch, final invoice, purchase, and inventory tables and backfills existing rows. |
| `src/database/migrations/201_remaining_snapshots.sql` (new) | Adds remaining snapshot columns for FIR, procurement, expenses, sales visits, and item/product details and backfills existing rows. |
| `src/database/migrations/202_procurement_ledger_snapshots.sql` (new) | Adds snapshot columns for procurement items and ledger entries and backfills existing rows. |
| `src/shared/snapshot/snapshot-builder.ts` (new) | Shared helpers (`buildPartySnapshot`, `buildSupplierSnapshot`, `buildCompanySnapshot`, `buildBankSnapshot`, `buildSignatorySnapshot`, `buildMasterNameMap`, etc.) to capture point-in-time master records. |
| `src/shared/constants/stock-thresholds.ts` (new) | Single source of truth for `STOCK_LOW_WARNING_SQM = 150` and `ORDER_BOOKING_BLOCK_MIN_SQM = 10`. |
| `src/modules/enquiry/enquiry.repository.ts`, `src/modules/order-booking/order-booking.repository.ts`, `src/modules/order-booking/order-booking.service.ts`, `src/modules/order-booking/order-booking.types.ts` | Captures and reads party, address, project, signatory, product, item, glass type, color, and finish snapshots; removes Order Booking lot/stock fields where dropped. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts`, `src/modules/final-invoice/final-invoice.repository.ts`, `src/shared/gst/gst-invoice-source.repository.ts` | Stores and reads frozen party, billing/shipping address, company, bank, signatory, and item snapshots on PI, Final Invoice, and GST/e-Invoice/e-Way Bill sources. |
| `src/modules/work-order/work-order.repository.ts`, `src/modules/work-order/work-order-inventory.service.ts`, `src/modules/work-order/replace-order.repository.ts`, `src/modules/production-scan/production-scan.repository.ts` | Reads and preserves product, item, party, and signatory snapshots across work orders, replacements, and production scans. |
| `src/modules/dispatch/dispatch.repository.ts`, `src/modules/dispatch/dispatch.service.ts`, `src/modules/dispatch/dispatch-shipment.repository.ts`, `src/modules/dispatch/dispatch-shipment.service.ts` | Captures and returns historical party, address, transporter, company, and item snapshots in dispatch and gate pass queries. |
| `src/modules/fir/fir.repository.ts`, `src/modules/fir/fir-ro.repository.ts` | Captures and reads customer, product, item, and signatory snapshots on FIR and rework orders. |
| `src/modules/procurement/procurement.repository.ts`, `src/modules/inventory-purchase/inventory-purchase.repository.ts`, `src/modules/inventory-purchase-return/purchase-return.repository.ts` | Captures supplier, signatory, product, item, glass type, and color snapshots (including conflict resolution in `replaceIndentItems` keeping snapshots and null-safe `colorCode`). |
| `src/modules/expenses/expense.repository.ts`, `src/modules/sales-visit/sales-visit.repository.ts`, `src/modules/accounts/ledgers.adopt.ts`, `src/modules/accounts/ledgers.service.ts`, `src/modules/tally/tally.sources.ts` | Reads snapshot names/GSTINs/parties first with live-master `COALESCE` fallbacks across expenses, sales visits, ledgers, and Tally exports. |
| `src/modules/report/**` (9 report repositories) | Updated enquiry follow-up, FIR, inventory, order status, production, sales income, sales rep, sales visit, and work order progress reports to prefer stored snapshots over live master joins. |
| `src/modules/inventory/inventory.repository.ts`, `src/modules/inventory/inventory.schema.ts`, `src/modules/inventory/inventory.service.ts`, `src/modules/inventory/inventory.types.ts` | Switches stock status and summary counts to SQM-only using `STOCK_LOW_WARNING_SQM` (150), adds `LOW_STOCK` tab filter and count (`0 < sqm < 150`), excludes low-stock lots from `IN_STOCK` (`sqm >= 150`), and exposes `lowStockThresholdSqm` in `InventorySummary`. |
| `src/modules/order-booking/order-booking-stock-check.ts`, `src/modules/order-booking/order-booking-stock-check.test.ts` | Replaces 2D piece-cutting and percentage stock check with SQM-only comparison against `STOCK_LOW_WARNING_SQM` (150) and `ORDER_BOOKING_BLOCK_MIN_SQM` (10), keeping party-supplied glass warn-only. |

## Why

Historical documents, reports, and GST/e-Invoice payloads must remain immutable when master records are edited later. In addition, the previous Order Booking stock check ran full 2D sheet/offcut simulation with `0%` and `20%` thresholds while the Stock page used hardcoded `20 SQM` / `5 pieces` thresholds without a dedicated Low Stock view.

## How

**Snapshot capture and fallback reads**: Write paths build JSONB and name snapshots at create/update/approve time via `snapshot-builder.ts`, and read/report/export queries use `COALESCE(snapshot, live_master)` so pre-migration and new records both resolve cleanly.

**Single SQM threshold file**: `src/shared/constants/stock-thresholds.ts` defines `STOCK_LOW_WARNING_SQM = 150` and `ORDER_BOOKING_BLOCK_MIN_SQM = 10`, imported by both `inventory.repository.ts` and `order-booking-stock-check.ts`, and returned in `InventorySummary.lowStockThresholdSqm` for the frontend.

**Mutually exclusive Stock status tabs**: `IN_STOCK` filters `available_sqm >= 150`, `LOW_STOCK` filters `0 < available_sqm < 150`, and `DEPLETED` filters `available_sqm <= 0`, so `IN_STOCK + LOW_STOCK + DEPLETED = ALL`.

**Order Booking SQM check**: `shortageOf` sums `requiredSqm` and compares it against `stock.availableSqm` for the glass group (`thickness + glassTypeCode + colorCode`). Company stock blocks when `availableSqm < 10 || availableSqm < requiredSqm`, and warns when `availableSqm < 150`; party-supplied glass (`customerPartyCode` present) never blocks and only warns when `availableSqm < 150`.

## Decisions made

**SQM-only thresholds without piece counts.** Piece-count checks (`< 5` pieces) were removed from Stock status and summary counts, and 2D bin-packing was removed from Order Booking submit checks, so both screens follow one predictable SQM rule.

**Backend owns the threshold constants.** Rather than duplicating `150` across separate frontend and backend repositories, the backend returns `lowStockThresholdSqm` in `InventorySummary` and the frontend falls back to `150` only while loading.

## Test plan

**Automated tests (all passing):**
- [x] `src/modules/order-booking/order-booking-stock-check.test.ts`: 7/7 tests passing (covers `>= 150` pass, `10 <= avail < 150` warn, `< 10` block, `avail < required` block, `mergeSameGlass`, and party-supplied warn-only).
- [x] `npx tsc --noEmit`: 0 errors after the final edit (including merge conflict resolution in `procurement.repository.ts`).

**Manual checks (local):**
- [ ] Full runtime end-to-end API/DB test after all final threshold and conflict edits - verified only by `tsc --noEmit` and unit tests above; migrations `200`-`202` were run earlier in the session.
