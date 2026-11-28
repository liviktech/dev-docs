# 2026-11-28 - historical-snapshots-and-stock-thresholds (frontend)

## Summary

Updates frontend types, detail modals, PDF/print views, and e-Invoice builders to display historical snapshot fields returned by the backend instead of relying on live master lookups, removes Lot and Stock display from the Order Booking details view, and updates the Inventory Stock page and Order Booking stock warning dialog to use the unified SQM-only thresholds (`150 SQM` warning, `10 SQM` block) with a new **Low Stock** tab.

## What changed

| File | Change |
|---|---|
| `src/features/pi-status/types.ts`, `src/features/pi-status/api.ts`, `src/features/pi-status/components/proforma-pdf-data.ts` | Adds and maps historical snapshot fields for Proforma Invoice details and PDF generation. |
| `src/features/final-invoices/types.ts`, `src/features/final-invoices/api.ts`, `src/features/final-invoices/components/final-invoice-detail-modal.tsx`, `src/features/final-invoices/final-invoice-as-proforma-data.ts`, `src/features/pwa/final-invoices/use-invoice-bill-to.ts` | Adds and renders frozen party, billing/shipping, company, bank, and item snapshot details on Final Invoices (desktop and PWA). |
| `src/features/e-invoice/components/einvoice-sheet-data.ts` | Uses snapshot party, address, GSTIN, and item details in e-Invoice sheet data. |
| `src/features/dispatch/components/gate-pass-modal.tsx`, `src/features/dispatch/components/pi-dispatch-detail-modal.tsx`, `src/features/dispatch/piece-stage.ts` | Uses snapshot fields in dispatch detail and gate pass modals. |
| `src/features/inventory/components/purchase-sheet.tsx` | Displays snapshot supplier/product details on purchase sheet. |
| `src/features/order-booking/api.ts`, `src/features/order-booking/booking-preview-modal.tsx`, `src/features/order-booking/order-booking-pdf.ts`, `src/features/order-booking/order-details-modal.tsx` | Maps snapshot fields for Order Booking previews/PDFs and removes Lot and Stock rows from the Order Booking details modal. |
| `src/features/order-booking/stock-warning.ts`, `src/features/order-booking/stock-warning-dialog.tsx` | Adds `availableSqm` to `OrderStockShortage` and updates the warning/block dialog to show `Available <n> SQM` and a `Low Stock` label when `shortSqm <= 0`. |
| `src/features/inventory/types.ts`, `src/features/inventory/stock-tab.tsx` | Adds `lowStockThresholdSqm` to `InventorySummary`, switches row Status badge to SQM-only (`<= 0` Out of Stock, `< 150` Low Stock, `>= 150` In Stock), adds the **Low Stock** tab after **In Stock**, and updates the **All** tab count to `IN_STOCK + LOW_STOCK + DEPLETED`. |
| `src/features/pwa/inventory/inventory-format.ts`, `src/features/pwa/inventory/stock-tab.tsx` | Mirrors the SQM-only `lotStockStatus` / `lotHasStock` rules and adds the **Low Stock** status chip on mobile PWA. |

## Why

Frontend modals, PDFs, and mobile views needed to render frozen historical snapshot data consistently, hide Lot/Stock from Order Booking details as requested, and reflect the backend's unified SQM-only stock thresholds and mutually exclusive **In Stock** / **Low Stock** / **Out of Stock** tabs.

## How

**Snapshot display**: Detail modals, PDF mappers, and e-Invoice data helpers prefer snapshot properties returned by the API when present.

**Stock status and Low Stock tab**: Desktop (`stock-tab.tsx`) and mobile (`pwa/inventory/stock-tab.tsx`) read `summary?.lowStockThresholdSqm` (defaulting to `150` while loading) so row badges match the backend's `IN_STOCK` (`>= 150 SQM`), `LOW_STOCK` (`0 < SQM < 150`), and `DEPLETED` (`<= 0 SQM`) tabs.

**Order Booking warning dialog**: `StockWarningDialog` handles both shortfall cases (`shortSqm > 0`) and low-buffer warnings (`availableSqm < 150` with `shortSqm = 0`) by showing `Available SQM` alongside `Required SQM`.

## Test plan

**Automated tests:**
- [x] `npx tsc --noEmit`: 0 errors after the final edit.

**Manual checks (local):**
- [ ] Live browser check of the new **Low Stock** tab and updated Order Booking warning dialog after the final edits - not re-run in Chrome after the final threshold and tab edits; covered by `tsc --noEmit`.
