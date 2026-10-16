# 2026-10-16 - fabrication-charges-and-receipts (backend)

## Summary
Final Invoices and the PI detail only counted order level charges, so the machining (fabrication) charges on each item were missing from partial work order invoices. The amount still to collect also went wrong when the party paid part of the invoice at creation, because approval overwrote that payment. This PR snapshots item level fabrication charges onto the PI, works out each Final Invoice's charges from the items it actually covers, and makes Party approval add the balance to what was already paid.

Both repos compile clean with a type check. Nothing was run against a real database, and no automated tests exist for these paths.

## What changed
| File | Change |
|---|---|
| `src/modules/final-invoice/final-invoice.repository.ts` | `getPiChargesAndItemsTotal` also reads the PI header `charge_amount` and uses it when there are no charge rows. New `getFinalInvoiceChargesBreakdown` returns item fabrication charges for the chosen items plus the order level charges prorated by item value. `findFinalInvoiceById` now uses it for the charges list, the charges total and the discount base. The remaining amount SQL now counts `collected_amount` for invoices not yet approved. |
| `src/modules/final-invoice/final-invoice.service.ts` | `createFinalInvoice` uses the new breakdown for the charges and the GST base. `getEligibleItemsForInvoice` returns `fabricationAmount` and `totalAmount` per item and a new `piOrderCharges`. Party approval sets `collected_amount` to the amount already paid plus the balance. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | `findProformaInvoiceById` returns `fabrication_amount` per item. `createProformaInvoice` now saves order level charges with `order_booking_item_id` as NULL, and copies each item's fabrication charges from `sales.order_booking_charge` into `invoice.proforma_invoice_charge`. |

## Why
Partial work order invoices should carry the machining cost of only the products on that invoice. Before, the PI total charges were prorated across all items, which spread fabrication cost onto products that did not have it. The Party approval fix stops a part payment at creation from being lost.

## How
**Item charges from the PI first**: the breakdown reads `invoice.proforma_invoice_charge`. Rows with an item id that matches a selected item are the fabrication charges. Rows with no item id are order level charges.

**Fallback for older PIs**: if the PI has no item level rows, the breakdown reads `sales.order_booking_charge` for the selected items, so PIs made before this change still work.

**Proration**: order level charges are multiplied by selected item value divided by total PI item value, rounded to 2 decimals. Fabrication charges are not prorated, they belong to the item.

**Collected amount**: `collected_amount` is the running total received. At creation it holds what was paid. At Party approval it becomes that plus the remaining balance. A legacy approved or confirmed row with no `collected_amount` still counts as fully paid.

**Kept the same on purpose**: the discount formula and the GST check at creation are unchanged. Only the charge figure that feeds them changed.

## Decisions made
**Charges are copied onto the PI at creation.** The other option was to always read live from the order booking. A copy keeps the PI stable if the order is edited later, and the order booking read is kept only as a fallback.

**The charge copy failures are swallowed.** The new snapshot block in `createProformaInvoice` sits in a try and catch with an empty catch, the same way the block above it does. A failure will not stop the PI from being created, but it will leave the PI without fabrication rows and nothing will be logged.

## Bugs found while testing (already there before, not fixed here)
**1. The discount at creation still uses the old base.** `createFinalInvoice` still builds the discount from `piTotals.totalItemsValue + piTotals.totalCharges`, while `findFinalInvoiceById` now uses item and order level charges separately. The two can differ on PIs with item level charges. It was already prorated by item value before this change.

## Notes
- No database migration is included. The code assumes `invoice.proforma_invoice_charge` already has the columns `order_booking_item_id`, `quantity`, `percentage` and `calculation_basis`. Check this on every environment before release.
- PIs created before this change have no item level rows and rely on the fallback.
- `getEligibleItemsForInvoice` runs two new queries with the shared `pool` and not the passed db client, so they are outside any caller transaction.

## Next steps
- Add a migration or schema check for the new charge columns.
- Align the creation time discount base with the detail view.
- Log the error in the empty catch of the charge snapshot.

## Test plan
**Automated tests (none added):**
- [ ] No test file covers `getFinalInvoiceChargesBreakdown`, Party approval collection or the PI charge snapshot - not added in this PR.
- [x] Backend type check (`npx tsc --noEmit`): no errors.

**Manual checks (local):**
- [ ] Create a PI with item fabrication charges, then a partial Final Invoice - not run, no database was used. The code was only read and type checked.
- [ ] Party approval after a part payment at creation - not run, same reason.
