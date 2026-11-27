# 2026-11-12 - purchase-edit-requests-stock-lock-and-order-pricing-history (backend)

## Summary

A set of changes to the Purchase / Inventory module, the Order Booking module and the Read from photo window, all in the same branch pair.

**Purchase edit requests (follow-up to the 2026-11-11 PR)**
1. **Admins can approve any request, including their own.** The self-approval block is removed. The sender can still cancel their own request.
2. **Edit on an in-stock purchase opens the normal Purchase modal in "request mode".** Before, it opened a separate, smaller form. Now it is the same form used elsewhere, with a required reason, and Save sends a request instead of saving.
3. **Stock fields stay editable until the lot is used.** Quantity, size, glass, product, location, supplier and received date can be corrected as long as no production has touched the lot. Once a work order has issued, selected or cut from the lot, those fields lock. The server checks this again at approval time.
4. **Approve no longer fails with "Internal server error".** The cause was a database check (approved quantity cannot be more than quantity) hit by two separate writes. It is now one write.
5. **Products table in the edit modal is consistent.** The edit icon on a row opens every field of that row, like the other tabs.
6. **"Stock used" notice.** When a lot is already used, a red notice says which fields are locked and why.
7. **Edit request buttons show a loader.** Approve, Reject and Cancel Request show a spinner on the clicked button while the call runs.

**Purchase tab and modals**
8. **New Purchase table columns:** Payment Status, Balance Due, Supplier Invoice No, Products (count), Storage Location and Stock Used. Every column is searchable and sortable on the server. Numbers are right-aligned.
9. **Product name instead of item-master name** in the View and Edit screens. For example a line bought as "Annealed Glass" no longer shows the glass type from the Items master. The "Item" label in the Add Purchase modal is now "Product".
10. **Supplier Description moved to purchase level.** One description per purchase, placed after Remarks and before the bill upload field.
11. **Quick-add dropdown** in the New Purchase modal now looks like the other dropdowns.
12. **Table polish.** Rounded, framed corners on the Payment, View and edit-request tables, and the "#" column is now "No" in Payment History and in the View modal (products and payment history).

**Database**
13. **Migration 130 no longer fails on existing duplicates.** It used to stop when two old purchases already shared a supplier invoice number. It now exempts those older rows and still blocks new duplicates.
14. **Migration 134 (new)** adds the purchase-level supplier description column.

**Order Booking**
15. **Enquiry remarks in the booking header.** After an enquiry is chosen, a small "Remarks" pill appears next to the enquiry date. Hover or click it to read the full remarks.
16. **Pricing History in the Add Product modal.** Beside the Rate (per SQM) label, a "Pricing History (N)" button lists this customer's last confirmed prices for the same product, thickness and type. Clicking a row applies that rate.

**Read from photo**
17. **Model shown under the AI banner.** The window now says "Read by Gemini · gemini-2.5-pro" (the real provider and model of that reading).

All changes were tested in the browser by the author. Both projects compile with no type errors. No automated tests were added.

**Supplier credit / overpayment handling is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/inventory-purchase/purchase-edit-request.service.ts` | Admins may approve their own request; approval re-checks used lots and refuses stock edits on a used lot; stock edits rewrite the lot under a lock |
| `src/modules/inventory-purchase/purchase-edit-request.repository.ts`, `purchase-edit-request.schema.ts` | Stock fields accepted in a request; `updateItemStock` writes quantity and approved quantity in one statement (fixes the approve error); `updateItemFields` no longer writes quantity |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | Server-side search and sort for the new list columns; "used lot" detection (non-receipt stock movement, offcuts, work order stock selection); supplier description at purchase level |
| `src/modules/inventory-purchase/inventory-purchase.service.ts`, `inventory-purchase.types.ts`, `inventory-purchase.schema.ts` | Payment status, balance due, products count, storage location and stock used on the list; supplier description field; lock flags on each line |
| `src/database/migrations/130_inventory_purchase_unique_supplier_invoice.sql` | Rewritten: the unique index is created while leaving out older duplicate rows, so the migration no longer fails on existing data |
| `src/database/migrations/134_inventory_purchase_supplier_description.sql` (new) | Adds the purchase-level supplier description column |
| `src/modules/order-booking/order-booking.repository.ts` | `findPricingHistory`: this customer's last 15 confirmed lines for the product, thickness and type, same currency, rate above 0, newest first, optionally leaving out one booking |
| `src/modules/order-booking/order-booking.service.ts`, `order-booking.controller.ts`, `order-booking.routes.ts` | `getPricingHistory` and `GET /order-bookings/pricing-history` (view permission, placed before the `/:id` routes) |

## Why

The edit request flow from the previous PR was too strict and too separate: admins could not approve their own corrections, in-stock purchases opened a different form, and fields that were safe to correct were locked from the start. The rule now follows the real risk, which is a lot that production has already used. The Purchase table needed more columns for day-to-day follow-up (payments, balance, location, usage). In Order Booking, sales staff asked to see past prices for a customer while quoting, and to see enquiry remarks without leaving the booking. The AI model label answers the question of which model read a photo.

The easy part was the screens. The real work was the used-lot rule (it must be identical in the form, the API and at approval time), the approve error, and making migration 130 safe on data that already has duplicates.

## How

**Used-lot rule**: a lot counts as used when it has a stock movement other than the receipt, an offcut, or a work order stock selection. A line is locked when its lot is used or was received in parts. The purchase shows `stockUsed` and `lotsInUse` from the same check, and approval runs it again before changing anything.

**Request mode**: the normal Purchase modal is reused. In request mode Save asks for a reason and sends a request. The server compares it with the saved purchase and only the changed fields go into the request.

**Approve fix**: the database rejects an approved quantity above the quantity, even for an instant between two writes. Quantity and approved quantity are now set in one statement.

**Read by line**: it uses the provider and model stored on the reading. A photo whose saved rows were reused shows the model of the original read.

## Decisions made

**Admins can approve their own requests.** The earlier rule blocked this. Requested by the team. The sender can still cancel.

**Lock on use, not on stock.** Editing is allowed until the lot is used in production (work order issue, selection or cutoffs). The other options were to lock all stock edits or to lock only after dispatch. Locking on use protects production data without blocking harmless corrections.

**One Supplier Description per purchase**, not per product line. Chosen by the author over a per-line description.

**Migration 130 exempts older duplicates.** The other option was to clean or delete the old duplicate rows, which would change real purchase data. New duplicates are still blocked.

**Pricing History shows only this customer's confirmed orders**, the last 15 priced lines, same currency, and nothing from Master Data rates. Lines with no rate or a rate of 0 are hidden, because 0 is not a price. If one order has two lines of the same specification, each line is a row. History rows are read-only except for the click-to-apply action.

## Bugs found while testing (already there before, not fixed here)

**1. Database constraint errors reach the user as "Internal server error".** The approve failure showed this because errors from database checks are not turned into readable messages. It was already like this. Any screen that breaks a database check can show it. Only the approve case was fixed here.

**2. The backend `.env` database username was mistyped**, so `npm run migrate` failed with an authentication error. This is local setup, not code, and I did not edit `.env`. It affects anyone who copies that `.env`.

## Notes

- `npm run migrate` must be run so migration 130 (fixed) and 134 (new) are applied. Saving Supplier Description does not work until 134 is applied.
- The backend must be restarted for the new `GET /order-bookings/pricing-history` endpoint.
- Pricing History was checked against real data: the busiest specification (ANNE-GLS, 8mm, Clear Saint Gobain Glass) returned 14 rows, newest first; an unknown product returned none; the previous-pricing lookup returned the same result as before.
- The used-lot lock was checked on real purchases (PUR-20261112-103, PUR-20261112-141, PUR-20261030-120, PUR-20261104-135).

## Next steps

- Turn database constraint failures into readable messages.

## Test plan

**Automated tests:**
- [x] Backend type check: no errors.

**Manual checks (local):**
- [x] Admin approves their own edit request; the sender cancels a request; spinners show on Approve, Reject and Cancel Request.
- [x] Approve works without the internal server error.
- [x] Supplier Description appears after Remarks and before the bill upload, and saves (after migration 134).
- [x] Migration 130 runs on data with existing duplicates; migration 134 applies.
- [x] Read from photo: the "Read by <provider> · <model>" line shows under the banner.
