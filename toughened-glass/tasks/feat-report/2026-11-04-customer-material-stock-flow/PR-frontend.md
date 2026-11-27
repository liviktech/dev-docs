# 2026-11-04 - customer-material-stock-flow (frontend)

## Summary

Customer glass for Without Material orders was typed on the order booking product form, and Work Order approval locked it to one lot with no choice. This PR moves the customer glass to the Customer Material tab, where it is added, edited, deleted and returned like stock, and lets Approve Work Order pick from that customer's lots with the normal Select stock picker. A Without Material product with no stock now offers "Add Customer Material" in place of Purchase. The Approve button stays off until stock is picked for every raw glass row.

`tsc --noEmit -p tsconfig.app.json` passes. I did not run any of the screens in a browser myself, so every screen below is checked by type check and code reading only.

**The Job Type dropdown on the order booking form is not removed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/product-detail-panel.tsx` | Customer Glass Details block removed (raw sheet width, height, pieces, Add Pieces). Without Material still sets the material rate to 0. |
| `src/features/order-booking/schema.ts` | Raw sheet fields are no longer required for Without Material. The `extraRawSheets` field stays in the form type. |
| `src/features/order-booking/add-product-modal.tsx` | Sizes are no longer checked against a raw sheet. Loads `extraRawSheets` for old items. |
| `src/features/order-booking/size-details-panel.tsx` | The size check takes a list of raw sheets (`rawSheetsMm`). The booking now passes an empty list, so no size is blocked. |
| `src/features/order-booking/api.ts` | Item type and `itemBody` carry `extraRawSheets` through. |
| `src/features/order-booking/job-type-badge.tsx` | `customerGlassText` joins the extra sheet sizes with "+" for old items. |
| `src/features/order-booking/work-order-stock-api.ts` | `customerPartyCode` on a stock group, next to `customerSupplied`. |
| `src/features/order-booking/work-order-stock-section.tsx` | A customer glass row shows a "Customer Stock" tag and the Select stock button works. When the picker shows low stock for a customer glass, the button is "Add Customer Material" (opens the Customer Material tab with the form filled in). With Material rows keep Purchase. Reports `allPicked` (every group has a pick). |
| `src/features/order-booking/work-order-edit-dialog.tsx` | The Approve button is disabled until `allPicked` is true, with a tooltip saying why. The normal Edit dialog is unchanged. |
| `src/features/inventory/customer-material.ts` | New list filters (customer, start and end date), and hooks to add, update and delete a lot. Row type gets product code, colour, glass type, remark and party description. |
| `src/features/inventory/components/add-customer-material-dialog.tsx` (new) | The New / Edit Party Material form. Customer, Product, Type, Color, Thickness, Location are required. A Sheets table in the same style as the Purchase dialog: Add Row, then Width, Height, Qty, Area (SQM, read only), Party Description, with confirm, edit and delete buttons per row. Accepts a prefill for the hand off from Approve Work Order. |
| `src/features/inventory/components/customer-material-view-dialog.tsx` (new) | Customer dialog opened from the Party ID: customer details and a table of that customer's lots grouped by product, with the Stock style lot code cell (first code, "+N", hover to list). |
| `src/features/inventory/customer-material-tab.tsx` | Add button wiring, Start Date and End Date pickers, customer dropdown and Reset button in the filter bar. Columns are now Party ID, Lot, Party, Date, Location and Actions. Stored At is plain text. Actions is a menu with Edit, Mark Returned and Delete (delete has a confirm dialog). |
| `src/features/inventory/stock-tab.tsx` | `PurchaseNumbersCell` is now exported so the customer dialog can reuse it. No other change. |
| `src/routes/inventory.tsx` | "Add Customer Material" header button on the Customer Material tab. New search params (`newCustomerMaterial`, `cmParty`, `cmProduct`, `cmThickness`, `cmType`, `cmColor`) open the form pre-filled, and are cleared when it closes. |
| `src/features/pi-status/components/approve-party-modal.tsx` | no real change. Only editor reformatting (quotes and semicolons). Do not review line by line. |

## Why

Entering the customer's sheet on the booking tied the glass to one order item, so it could not be received once and used by several orders, edited, or deleted. Treating it as stock held per customer lets the store enter what arrives, and lets approval choose which lots to cut from, the same way it works for Rubine's own glass.

## How

**Where the glass is entered**: Add Customer Material creates one lot per sheet row. Edit changes one lot. The sheet size and quantity can only change before cutting starts, the server enforces that and the toast shows its message.

**Approve Work Order**: The customer glass row uses the same picker as other glass. The lots offered are that customer's lots with the same product and thickness. If the picker shows low stock or no matching lot, the button becomes Add Customer Material. It opens `/inventory` on the Customer Material tab with the customer, product, thickness, type and colour already filled in.

**Approve button**: `WorkOrderStockSection` now reports `allPicked` along with the selections. It is false while the stock list loads, and true when every group has at least one pick. If the stock list fails to load it reports true so approval is not locked by a load error.

**Customer Material table**: One row per lot. The Lot column and the Actions menu work on that lot. The filters call the same list endpoint, so the tab counts follow the customer and date filters but the three KPI cards still count everything.

**Submit fix**: The form's submit handler now calls `preventDefault()`. Without it the browser submitted the form itself and reloaded the page, which showed a white page for about a second after Save.

## Decisions made

**The Job Type dropdown stays.** Removing it would change how the rest of the booking works (rate 0, hidden material charges, the work order flow). Only the sheet entry was removed from the booking.

**Approve needs a pick for every glass, not only customer glass.** This was asked for in general ("without select stock approve button disable"), so the old "leave it on Auto" approval no longer works for any glass row. If this should apply only to customer glass, it is one condition in `work-order-stock-section.tsx`.

**Area (SQM) is shown as a whole number in the form.** It is rounded for display only. The server works out the stored area from the exact width, height and quantity.

**Size check on bookings is switched off, not deleted.** `SizeDetailsPanel` still supports a raw sheet list, the booking just passes none. Kept so it can come back if needed.

## Notes

- The two commits also contain the earlier "Add Pieces" work (`extraRawSheets` in `schema.ts`, `api.ts`, `add-product-modal.tsx`, `size-details-panel.tsx`, `job-type-badge.tsx`). It was not committed before. The booking form no longer shows it, but old items that have extra sheets still load.
- The Add Customer Material form is also used for Edit. An older lot saved without a type, colour or location must have them filled in before an edit can be saved.
- When Select stock saves a pick, it is stored by product and thickness only. If one work order has a With Material and a Without Material row of the same product and thickness, a pick on one could be offered for the other. Not seen in testing, because no browser run was done.

## Next steps

- Open each screen in the browser and check: add, edit, delete and Mark Returned, the filters, the Approve Work Order flow with a customer glass that has no stock, and the Approve button states.
- Decide whether the Approve rule should apply to customer glass only.
- Hide the Material label on the PI status drawer, proforma sheet and final invoice screens for Without Material (carried over from the earlier job work PR).

## Test plan

**Automated tests (no test cases were added or run):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes with no errors.
- [ ] Unit or component tests: none exist for these screens, so none were added.

**Manual checks (local):**
- [ ] Add Customer Material, Customer Material table, Edit, Delete, Mark Returned: not run in a browser. Covered by type check only.
- [ ] Approve Work Order with a customer glass and no stock, then Add Customer Material hand off: not run in a browser.
- [ ] Approve button disabled until every glass has a pick: not run in a browser.
- [ ] Save in the form no longer reloads the page: the cause was found by reading the form code (no `preventDefault`) and fixed. Not seen live.
