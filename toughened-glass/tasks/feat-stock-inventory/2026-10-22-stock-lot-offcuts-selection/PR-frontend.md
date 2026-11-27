# 2026-10-22 - stock-lot-offcuts-selection (frontend)

## Summary

The Inventory page let users add and edit movements by hand and used an Adjust Stock dialog to change quantities, so the movement list could not be trusted. The Work Order approval screen had no way to see which lots or leftover pieces the glass would come from, and no way to choose them. This PR makes Movements a read-only, grouped history, replaces Adjust Stock with the lot edit form, adds a lot view with its history, and adds an optional colour and manual lot selection to Work Order approval, with a Refill stock shortcut and a Purchase shortcut when stock is low. Glass brand is hidden in the UI only (the database keeps it).

**Brand removal from the database is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/routes/inventory.tsx` | Large simplification. The manual movement dialogs are gone and the Movements tab moved to its own file. The search params `editLot`, `newPurchase`, `refillProduct`, `refillThickness` and `refillType` are read with `validateSearch`. They open the edit form of a lot, the Purchase tab with a new purchase form, or the best lot for a refill. The params are cleared once handled. |
| `src/features/inventory/movements-tab.tsx` (new) | Read-only Movements tab. Global search bar, date range, group by lot or by Work Order, expandable groups with the lot code or Work Order as the first column, remark column, and a pagination footer. |
| `src/features/inventory/components/lot-detail-dialog.tsx` (new) | Lot view as a two step stepper (details, then history). |
| `src/features/inventory/lot-format.ts` (new) | Shared formatting helpers for lots and movements. |
| `src/features/inventory/stock-tab.tsx` | Adjust Stock removed from the row menu and its dialog (only Edit and Delete remain). Opens the edit form from `editLot` or from a refill request, and shows an info toast if no lot of that product exists. Brand column hidden. |
| `src/features/inventory/components/receive-stock-dialog.tsx` | New Stock form: colour is required and glass type is added. "+ Add Type" opens the selected product's edit form on the Rates tab. Sheets quantity is editable on purchase lots, with an updated notice. Brand field hidden. |
| `src/features/inventory/components/purchase-dialog.tsx` | Passes glass type through. Brand hidden. |
| `src/features/inventory/components/edit-movement-dialog.tsx` (deleted) | Movements are read only. |
| `src/features/inventory/components/record-movement-dialog.tsx` (deleted) | Movements are read only. |
| `src/features/inventory/api.ts`, `queries.ts`, `types.ts` | Added lot history, grouped movements and ledger calls. Removed the create and update movement calls and the adjust stock hook, call and `AdjustStockInput` type. |
| `src/features/order-booking/work-order-stock-section.tsx` (new) | Raw glass table inside Approve Work Order. Optional colour choice, Select stock dialog for manual lots, Selected / Required text, and a Refill stock button for low stock groups. |
| `src/features/order-booking/work-order-stock-api.ts` (new) | API calls and hooks for stock candidates, material choice and stock selection. |
| `src/features/order-booking/work-order-edit-dialog.tsx`, `work-order-summary-modal.tsx`, `api.ts` | Show the new stock section and send the choices on approval. |
| `src/routes/work-orders.tsx` | Insufficient Stock message is now 12px semibold slate, like the rest of the app. The Purchase button opens `/inventory` with `newPurchase`. The rose reject box further down still uses `text-sm`, left as is. |
| `src/features/products/product-dialog.tsx` | New `initialTab` prop so a caller can open the dialog on a given tab (used for Add Type). |
| `src/components/ui/data-table.tsx` | Small additions for expandable rows and manual pagination used by Movements. |
| `src/features/pwa/inventory/inventory-format.ts`, `mobile-lot-detail.tsx`, `mobile-lot-form.tsx`, `movements-tab.tsx`, `stock-tab.tsx` | Mobile inventory follows the same rules: read-only movements, brand not shown or sent, glass type shown. |

## Why

This follows `info/docs/stock-lot-cutoff-and-lot-selection-plan.md` and the backend PR. Showing the lots a Work Order will use, and letting a person pick them, needed new UI. The movement list is only useful if nothing can change it directly, so the manual dialogs were removed. Quantity corrections now go through the lot edit form, and the backend writes the movement.

## How

**Movements**: grouped by lot or by Work Order and loaded one page at a time. Clicking the chevron or the badge in the first column opens the group.

**Lot edit as the only way to change quantity**: the sheets quantity can be changed for any lot. The backend adds the ADJUSTMENT movement. The form no longer locks the quantity for purchase lots.

**Shortcuts to Inventory**: Refill stock and Purchase navigate to `/inventory` with a search param. The inventory route reads it once, opens the right form, and clears the param so a refresh does not reopen it. Refill picks the lot of the same product, preferring the same thickness, then glass type, then the newest received date.

**Work Order raw glass table**: uses the same fonts as the PI Status drawer (13px bold headers, 13px semibold cells). The Select stock button copies the Photos button style from Enquiry. The Take input has no spinner arrows and browser autofill is turned off.

**Kept the same on purpose**: the brand field stays in the types and in the data sent to the backend where the API needs it. It is only hidden from view.

## Decisions made

**Brand is hidden, not removed.** The request was to hide it in the UI only (Inventory pages and Approve Work Order). The database and API keep the column.

**Refill opens an existing lot, not New Stock.** An early version opened New Stock when no matching lot was found. That was wrong, so it now edits the best matching lot and shows an info toast when the product has no lot at all.

**Refill may open a lot of another glass type.** The lot closest to the shortage is picked. The glass type can be changed in the form unless the lot has history or came from a purchase.

**"No Grouping" in Movements was kept.** This was only a yes or no question, no code changed.

## Notes

- Nothing was checked in a browser in this session.
- Only `tsc -b` was run on the frontend. No frontend test suite was run.
- The backend must be restarted and migration 079 applied before this works (see the backend PR).

## Test plan

**Automated tests (no frontend tests run):**
- [x] `npx tsc -b`: no errors.

**Manual checks (local):**
- [ ] Movements tab: search, date range, group by lot and by Work Order, expand, pagination - not done, not opened in a browser.
- [ ] Edit a purchase lot quantity and see the ADJUSTMENT in Movements - not done, needs the backend and database.
- [ ] Insufficient Stock dialog: Purchase opens the Purchase tab with the new purchase form, Refill stock opens the lot edit form - not done, only type checked.
- [ ] New Stock: colour required, "+ Add Type" opens the product edit form on the Rates tab - not done, only type checked.
- [ ] Approve Work Order raw glass table fonts, Select stock button, Take input with no arrows or autofill popup - not done, only type checked.
