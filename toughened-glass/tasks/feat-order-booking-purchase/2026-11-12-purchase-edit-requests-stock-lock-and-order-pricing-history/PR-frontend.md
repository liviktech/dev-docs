# 2026-11-12 - purchase-edit-requests-stock-lock-and-order-pricing-history (frontend)

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

**Purchase edit requests**

| File | Change |
|---|---|
| `src/features/inventory/components/purchase-dialog.tsx` | Request mode (`requestMode`, `onSubmitRequest`) with a required reason; per-line and header stock locks from the server's used-lot flags; red "stock used" notice; products table edit icon opens all fields of the row; "Item" label is now "Product"; Supplier Description is now one field for the purchase (after Remarks, before the bill upload); quick-add dropdown styling fixed |
| `src/features/inventory/purchase-tab.tsx` | Edit on an in-stock purchase opens the normal modal in request mode; new columns (Payment Status, Balance Due, Supplier Invoice No, Products, Storage Location, Stock Used) with search and sort; numbers right-aligned |
| `src/features/inventory/purchase-edit-requests.ts`, `purchase-types.ts` | Request types and hooks follow the new payload (stock edits, supplier description); new list fields on the purchase |
| `src/features/inventory/components/purchase-edit-requests-dialog.tsx` | Admins can approve their own requests; Approve / Reject / Cancel Request show a spinner and a "...ing" label while running, the other buttons stay disabled |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Shows the product name, not the item-master name; "#" is now "No" in products and payment history; framed table corners |
| `src/features/inventory/components/purchase-payment-dialog.tsx` | "#" is now "No" in Payment History; framed table corners |
| `src/features/inventory/components/purchase-reference-cards.tsx` | `FRAMED_TABLE` style shared by the framed tables |
| `src/features/inventory/components/purchase-sheet.tsx` | Follows the new purchase fields |
| `src/features/pwa/inventory/purchase-tab.tsx` | Follows the new purchase fields |

**Order Booking**

| File | Change |
|---|---|
| `src/features/order-booking/booking-header-form.tsx` | Enquiry remarks pill with a hover/click popover next to the enquiry date |
| `src/features/order-booking/pricing-history-popover.tsx` (new) | The "Pricing History (N)" button and its popup: Order No, Order Date, Rate (per SQM); click or Enter on a row applies the rate; the row matching the current rate is ticked; hidden when there is no history |
| `src/features/order-booking/product-detail-panel.tsx` | New optional `pricingHistory` prop; the button sits on the Rate (per SQM) label row |
| `src/features/order-booking/add-product-modal.tsx` | Loads the history for the chosen customer, product, thickness and type; `applyHistoricRate` sets and locks the rate and skips the Previous / Master choice dialog; the choice dialog itself is unchanged |
| `src/features/order-booking/api.ts`, `queries.ts` | `getPricingHistory` and `pricingHistoryQueryOptions`; `remarks` added to the local Enquiry type |

**Read from photo**

| File | Change |
|---|---|
| `src/features/ai-extraction/photo-review-panel.tsx` | "Read by <provider> · <model>" line under the "This is AI generated" banner, with friendly provider names |

## Why

The edit request flow from the previous PR was too strict and too separate: admins could not approve their own corrections, in-stock purchases opened a different form, and fields that were safe to correct were locked from the start. The rule now follows the real risk, which is a lot that production has already used. The Purchase table needed more columns for day-to-day follow-up (payments, balance, location, usage). In Order Booking, sales staff asked to see past prices for a customer while quoting, and to see enquiry remarks without leaving the booking. The AI model label answers the question of which model read a photo.

The easy part was the screens. The real work was the used-lot rule (it must be identical in the form, the API and at approval time), the approve error, and making migration 130 safe on data that already has duplicates.

## How

**Used-lot rule**: a lot counts as used when it has a stock movement other than the receipt, an offcut, or a work order stock selection. A line is locked when its lot is used or was received in parts. The purchase shows `stockUsed` and `lotsInUse` from the same check, and approval runs it again before changing anything.

**Request mode**: the normal Purchase modal is reused. In request mode Save asks for a reason and sends a request. The server compares it with the saved purchase and only the changed fields go into the request.

**Approve fix**: the database rejects an approved quantity above the quantity, even for an instant between two writes. Quantity and approved quantity are now set in one statement.

**Purchase list**: search and sort run in SQL over the new columns, so they cover all pages, not just the one on screen. Status words are matched by prefix (for example "part" finds Partially Paid).

**Pricing History**: the button appears once customer, product, thickness and type are chosen, and not in read-only mode or for Without Material products. A picked rate is locked the same way "Previous Order Pricing" is, so changing the glass type does not overwrite it and the "Which pricing should be used?" dialog does not ask afterwards. That dialog still uses only the most recent order. The history is hidden when empty.

**Read by line**: it uses the provider and model stored on the reading. A photo whose saved rows were reused shows the model of the original read.

## Decisions made

**Admins can approve their own requests.** The earlier rule blocked this. Requested by the team. The sender can still cancel.

**Lock on use, not on stock.** Editing is allowed until the lot is used in production (work order issue, selection or cutoffs). The other options were to lock all stock edits or to lock only after dispatch. Locking on use protects production data without blocking harmless corrections.

**"Stock used" notice only informs.** It does not block the form. The locked fields are what enforce the rule.

**One Supplier Description per purchase**, not per product line. Chosen by the author over a per-line description.

**Pricing History shows only this customer's confirmed orders**, the last 15 priced lines, same currency, and nothing from Master Data rates. Lines with no rate or a rate of 0 are hidden, because 0 is not a price. If one order has two lines of the same specification, each line is a row. History rows are read-only except for the click-to-apply action.

**Supplier credit / overpayment handling is not included.** The team is still deciding how it should work, so nothing was built.

**Remove means comment out.** Where a section was removed from a screen, it is commented out, not deleted.

## Bugs found while testing (already there before, not fixed here)

**1. Database constraint errors reach the user as "Internal server error".** The approve failure showed this because errors from database checks are not turned into readable messages. It was already like this. Any screen that breaks a database check can show it. Only the approve case was fixed here.

## Notes

- Pricing History was checked against real data: the busiest specification (ANNE-GLS, 8mm, Clear Saint Gobain Glass) returned 14 rows, newest first; an unknown product returned none; the previous-pricing lookup returned the same result as before.
- The used-lot lock was checked on real purchases (PUR-20261112-103, PUR-20261112-141, PUR-20261030-120, PUR-20261104-135).
- Existing center and left aligned number or status columns in the Purchase table were not re-aligned, only the new ones.

## Next steps

- Decide how supplier credit and overpayment should work, then build it.
- Turn database constraint failures into readable messages.
- Apply the framed table style to the other tables (edit-request dialogs, Order Booking details modal).
- Fix the same quick-add dropdown styling in the receive-stock dialog.
- Rename the "Product Code" column in Add Purchase to "Supplier Item Code".

## Test plan

**Automated tests:**
- [x] Frontend type check (`tsc --noEmit -p tsconfig.app.json`): no errors.

**Manual checks (local):**
- [x] Admin approves their own edit request; the sender cancels a request; spinners show on Approve, Reject and Cancel Request.
- [x] Edit on an in-stock purchase opens the normal modal in request mode and asks for a reason.
- [x] Stock fields are editable on an unused lot and locked on a used lot; the red "stock used" notice shows.
- [x] Products table edit icon opens every field of the row.
- [x] Purchase table shows the new columns; search and sort work on each; numbers are right-aligned.
- [x] View and Edit screens show the product name; the Add Purchase label reads "Product".
- [x] Quick-add dropdown looks like the other dropdowns; table corners and the "No" column look right in the Payment and View modals.
- [x] Order Booking: the enquiry remarks pill and popover show after choosing an enquiry.
- [x] Add Product: Pricing History lists the past prices; clicking a row applies and locks the rate; the choice dialog does not ask after that.
- [x] Read from photo: the "Read by <provider> · <model>" line shows under the banner.
