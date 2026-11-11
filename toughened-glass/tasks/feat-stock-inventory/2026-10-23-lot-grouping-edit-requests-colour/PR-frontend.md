# 2026-10-23 - lot-grouping-edit-requests-colour (frontend)

## Summary

Stock showed one row per product, anyone could edit a lot, the View Lot dialog was split across several tables, and order booking had no colour. This PR shows one lot per purchase in Stock, replaces the stock edit with an edit request flow (admins edit directly, others send a request with a mandatory reason, an admin approves or rejects), limits editing of glass that is already used in work orders to quantity only, rebuilds the View Lot history as one table with clickable Work Order and PI badges, adds an optional Colour field (default Clear) to the order booking product form, and makes Select Stock show only lots of the booked colour. It also tidies the Stock table and the two request dialogs.

**None of this was opened in a browser.** Only the type check was run, see "Test plan" below.

## What changed

| File | Change |
|---|---|
| `src/features/inventory/types.ts` | New types for a lot group, its product items and per product history. `inUse` flag on a lot. |
| `src/features/inventory/edit-requests.ts` (new) | Queries and mutations for edit requests, the review roles, and `editErrorMessage` (shows the first field message, for example "Product 2: ..."). |
| `src/features/inventory/components/lot-group-edit-dialog.tsx` (new) | Edit form for one lot with all its products. Unused products can change type, colour, thickness, size and quantity. Used products show only the quantity, with a red warning that lists them and says why. |
| `src/features/inventory/components/lot-edit-reason-dialog.tsx` (new) | Asks for the mandatory reason before a request is sent. |
| `src/features/inventory/components/lot-edit-requests-dialog.tsx` (new) | The Stock Edit Requests list with Pending and All tabs. Admins get Approve and Reject, the requester gets only Cancel Request. |
| `src/features/inventory/components/lot-detail-dialog.tsx` | The history step is now one table: Work Order, PI No, Product, Colour, Customer, Last Used, Used (SQM), Returned (SQM). The two badges open the Work Order preview and the PI preview. The Purchased From and Delivered To Customers tables are removed. |
| `src/features/inventory/components/receive-stock-dialog.tsx` | Locks the fields of a lot that is in use. Errors use `editErrorMessage`. |
| `src/features/inventory/components/purchase-dialog.tsx` | The rows a "Purchase" link hands over are set by one effect that gives the same result when it runs twice (fixes the second New Purchase opening empty). Prefilled rows no longer disable Add Row, only rows you added and have not confirmed do. |
| `src/features/inventory/stock-tab.tsx` | One row per purchase lot. Glass Size is centered. The Location column is hidden. The Edit Request column shows only a clickable "Pending Request" badge (no tick or cross) and the "No pending requests" text matches the other columns. Approve and reject moved out of the table. |
| `src/features/inventory/purchase-tab.tsx`, `purchase-types.ts` | Pass the prefill glasses (`PurchasePrefill`) to the purchase form. |
| `src/routes/inventory.tsx` | The New Stock button is replaced by Edit Requests (with the pending count). Reads the `purchaseItems` search param and opens New Purchase from it. |
| `src/routes/work-orders.tsx`, `src/features/order-booking/stock-shortage.ts` (new) | The shortage warning lists each short glass, and its Purchase button opens New Purchase with a row per glass. |
| `src/features/order-booking/work-order-edit-dialog.tsx`, `work-order-form-fields.tsx` | When approving with a delivery date that has passed, the date is cleared and a warning asks for a new one. |
| `src/features/order-booking/schema.ts`, `api.ts`, `add-product-modal.tsx` | `colorCode` on the item (default `CLEAR` for a new one, empty for items booked before). Sent as null when empty. |
| `src/features/order-booking/product-detail-panel.tsx` | New searchable Colour dropdown in Product Specification with "Any Colour" and "+ Add Color", like the other colour dropdowns. |
| `src/features/order-booking/work-order-stock-api.ts`, `work-order-stock-section.tsx` | The group carries `bookedColorCode`. The Purchase prefill uses the chosen colour, then the booked colour. |
| `src/features/pi-status/components/pi-change-dialogs.tsx` | The Open and All tabs now use the shared FilterTabs. |
| `src/features/pi-status/pi-status-page.tsx` | The Change Requests button uses the same cyan style as the Edit Requests button. |

## Why

Stock should match how glass is bought (one purchase, one lot) and changes to real stock should be reviewed. The in use rule stops edits that would make old movements and work orders wrong. The colour field and filter make a work order take the colour the customer asked for. The tab and table changes keep these screens consistent with the rest of the app.

## How

**Edit flow**: an admin saves straight away. Anyone else gets a reason dialog and a request is created. A used product row is locked to quantity in the form, and the server enforces the same rule, so the form is a help and not the only guard.

**Colour default**: a new product starts with `CLEAR`. If the colour master has no code `CLEAR`, an effect picks the colour named Clear, or none. "Any Colour" is sent as null. Products booked before colours existed open as Any Colour.

**View Lot badges**: the Work Order badge opens the existing Work Order preview by id. The PI badge looks the PI up by number with the PI status list and opens the invoice preview, and shows a toast if it is not found.

**Alignment**: Glass Size uses `meta.align: "center"` on the column, as the table alignment rules ask, not a layout trick inside the cell.

**Kept the same on purpose**: the Select Stock screen itself is unchanged. The colour chips come from the server, which now only returns the booked colour.

## Decisions made

**Approve and reject live only in the Edit Requests dialog.** The Stock table column only shows the pending badge, which opens that dialog, so the same action is not in two places.

**Reject for admins, Cancel for the requester, never both.** An admin reviews, the person who asked can withdraw their own request.

**The Colour field sits alone in its own row** in Product Specification, so that row looks a little uneven. It can be moved if that is not wanted.

## Notes

- Existing order items have no colour (any colour). Only new items default to Clear.
- Needs backend migrations 083 and 084 to be applied first.
- Saved material choices are shared by colour groups of the same product and thickness, see the backend PR.

## Test plan

**Automated tests (all passing):**
- [x] `npx tsc -p tsconfig.app.json --noEmit`: passes after the last change.

**Manual checks (local):**
- [ ] Stock table, Edit Requests dialogs (stock and PI), View Lot table, order booking Colour field and Select Stock colour filter in the browser - not done. Only the type check ran. The server side of the edit flow and the colour filter was checked by script, see the backend PR.
