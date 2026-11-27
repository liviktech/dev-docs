# 2026-11-11 - purchase-invoice-payments-cost-and-edit-approval (frontend)

## Summary

A set of changes to the Purchase / Inventory module, all in the same branch pair.

1. **Duplicate supplier invoice numbers are blocked.** The same supplier can no longer have the same invoice number on two purchases of the same company (checked in the form, in the API and in the database).
2. **LOT numbers are now counted per company.** Add to Stock was taking the next LOT number from a shared counter. Each company now has its own.
3. **Unpaid purchases no longer show as fully paid.** The paid amount and the payment state now come from the server and are the same in the list, the View modal and the Payment modal.
4. **Partial payments.** A purchase can be paid in instalments. The Payment modal was redesigned (summary, products, payment history, Add Payment form). Each payment keeps its amount, date, mode, reference and bank / UPI / cash details. The status is Unpaid, Partially Paid or Paid.
5. **Goods Received Date.** "Purchase Date" is renamed "Goods Received Date" and it cannot be earlier than the Supplier Invoice Date. The date picker disables earlier days once the invoice date is chosen.
6. **Stock lot cost.** The Stock tab now shows Purchase Rate, Cost Unit and Current Stock Value per lot, "N/A" for customer lots and "Cost Not Available" when a lot has no cost.
7. **Correct item matching.** A purchase line now matches the Items page by exact glass type, colour and thickness. The most specific item wins. Two equally good items are refused instead of picking one.
8. **Purchase modal clean-ups.** View modal, Edit modal and Payment modal were reworked (listed below).
9. **Edit request with admin approval (partially completed, not tested in the browser).** A purchase that is in stock, or has a payment, can no longer be edited directly. Edit sends a correction request with a reason. A new **Edit Payment** menu item does the same for one recorded payment. Nothing changes until another admin approves it.

The author checked items 1 to 8 in the browser. **Item 9 was NOT tested in the browser and is only partially complete** - see "Decisions made" and "Test plan". Both projects compile with no type errors. No automated tests were added.

**The mobile app (PWA) edit-bill field, a mobile Edit Request screen and notifications for new edit requests are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

**Duplicate invoice, dates and item matching**

| File | Change |
|---|---|
| `src/features/inventory/components/purchase-dialog.tsx` | Debounced duplicate-invoice check with a 409 error mapped to the field; "Goods Received Date" label with `minDate` set to the invoice date, and the received date is cleared if the invoice date moves later; item matching through `pickMatchingItem`, HSN follows the chosen item; the Invoice / Bill / Receipt field now also shows when editing (saved bill with View Bill, upload a file to replace it) |
| `src/features/inventory/purchase-item-match.ts` (new) | `pickMatchingItem` returns exact, ambiguous or none |
| `src/features/inventory/purchase-api.ts`, `purchase-queries.ts` | `checkSupplierInvoice` call; failed mutations refresh the lists |
| `src/features/pwa/inventory/mobile-purchase-form.tsx` | Received date validation and minimum date; "Goods Received Date" label |

**Payments, status badges and labels**

| File | Change |
|---|---|
| `src/features/inventory/purchase-types.ts` | `PurchasePaymentStatus`, `PurchasePaymentTransaction`, `paymentHistory`, new payment input (amount, date, request id, reference), lot cost fields |
| `src/features/inventory/purchase-tab.tsx` | Three-state payment badges; Payment menu disabled only when Paid; "Goods Received Date" column; **Edit** now sends an edit request when the purchase is in stock or has a payment; new **Edit Payment** menu item; Edit disabled while a request is pending |
| `src/features/inventory/components/purchase-payment-dialog.tsx` | Redesigned Payment modal: Summary, Product Details, Payment History, then Add Payment form (amount, date, reference, mode fields). Stays open after saving. New edit mode used by Edit Payment: an Edit button per payment, a reason box, "Submit for Approval" |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Stock and payment badges moved to the modal header; Supplier Bill moved into the General Details header (compact link); Payment Details built from the real payment history; before the first payment only the "No payment has been recorded yet" message shows; table text style matched to the other sections; old Supplier Bill card commented out |
| `src/features/inventory/components/payment-slip.tsx` | `PaymentSlipLink` gets a `compact` size |
| `src/features/inventory/components/purchase-reference-cards.tsx`, `purchase-sheet.tsx` | "Goods Received Date" label |
| `src/features/pwa/inventory/purchase-payment-sheet.tsx`, `purchase-tab.tsx` | Mobile payment sheet sends amount, date, reference and a request id; mobile payment label and pay button follow the three states |

**Stock lot cost**

| File | Change |
|---|---|
| `src/features/inventory/stock-tab.tsx`, `src/features/inventory/types.ts` | New columns Purchase Rate, Cost Unit and Current Stock Value, with "N/A" and "Cost Not Available" states |

**Edit request with admin approval (partially completed, not tested)**

| File | Change |
|---|---|
| `src/features/inventory/purchase-edit-requests.ts` (new) | Types, list query and the hooks to raise, approve, reject and cancel a request |
| `src/features/inventory/components/purchase-edit-request-dialog.tsx` (new) | Edit form for a purchase that is in stock or paid: invoice details, charges, per-product rate, discount, HSN and supplier wording, and a required reason |
| `src/features/inventory/components/purchase-edit-requests-dialog.tsx` (new) | Admin list with Pending / Approved / Rejected / All tabs, a before-and-after table for each request, Approve, Reject (reason required) and Cancel |
| `src/routes/inventory.tsx` | **Edit Requests (n)** button on the Purchase tab header and the new list dialog |

## Why

These came from a series of specs on the Purchase module: duplicate invoices, per-company LOT numbers, the wrong paid state, partial payments, date rules, lot cost, wrong item selection, several modal fixes, and finally a correction workflow for purchases that are already in stock or paid. The easy parts were labels and the modal layout. The real work was the payment rules (so every screen shows the same figures), concurrency (two payments at once) and the approval flow.

## How

**Payment rules**: the server decides the paid amount and the status. The purchase row is locked while a payment is recorded, so two payments at once cannot overpay. A payment form carries a request id, so a double click records it once. The older one-shot payment (stock posting) still counts as fully paid and is shown as one payment.

**Item matching**: an item must match glass type, colour and thickness exactly. Among matches, the one with the fewest colours wins. If two are equally specific the save is refused and both item codes are named.

**Lot cost**: each lot is joined to its purchase line. The cost columns are worked out on the server from that line, so no cost is stored twice.

## Decisions made

**What a purchase correction can change.** Invoice number, invoice date, additional charges, remarks, and per product the rate, rate unit, discount, HSN, supplier item code and description. Not supplier, product, glass, colour, size, quantity, location or Goods Received Date, because those make up the stock lots (the received date is copied onto them). Easy to widen later.

**Which payments can be corrected.** Only those recorded through the Payment option. Older one-shot payments show "Older payment" and cannot be corrected.

**Products that cannot be re-priced.** A product received in parts at different rates, or on a purchase paid the old one-shot way, is refused with a message.

**Admins also need a second admin.** The spec says nobody approves their own request, so an admin's own request also goes to another admin. A company with one admin cannot approve its own requests.

**Edit Payment needs the Add right.** It reuses the payment-options call, which needs the inventory Add right. A user with Edit but not Add cannot use it.

**Mobile app.** The saved-bill field is still hidden when editing on mobile, and there is no mobile screen for edit requests. Notifications for new requests are not built.

**Stock value uses gross or net discount?** Left open for the team to decide.

**"Purchase Date" label in Creditors and the mobile Sales-Income screens is unchanged.** Only the Purchase module was renamed.

## Bugs found while testing (already there before, not fixed here)

**1. View modal showed no partial payments.** It only read the older one-shot payment. Fixed here by building it from the payment history.

**2. Lint error in `purchase-payment-dialog.tsx` (state set inside an effect).** The same pattern exists in many other files (for example `src/routes/inventory.tsx`). It was not touched here.

## Notes

- Purchase PUR-20261029-114 needs its bill uploaded again.

## Next steps

- Test the whole edit request flow in the browser (raise, approve, reject, cancel, the Edit Requests list, Edit Payment) and finish whatever it shows.
- Decide on gross or net discount in stock value.
- Add the mobile bill field and mobile edit requests, and notifications for new requests, if wanted.
- Add automated tests for the payment rules and the approval flow.

## Test plan

**Automated tests:**
- [ ] No test files were added or changed. The checks below are not automated.
- [x] Backend and frontend type checks (`tsc --noEmit`): no errors.
- [x] Lint: one new finding in the new edit-request form was fixed; the other errors are in older code.

**Manual checks (local):**
- [x] Duplicate invoice, per-company LOT numbers, paid state, partial payments, Payment modal, Goods Received Date rule, stock lot cost columns, item matching, and the View / Edit / Payment modal changes: checked in the browser by the author.
- [ ] Edit request screens (Edit form, Edit Payment, Edit Requests list, Approve / Reject / Cancel) - **not tested in the browser**; this part is partially completed.
- [ ] Bank transfer and cheque payments end to end - the test supplier had no bank accounts, so only the checks were covered.
- [ ] Mobile edit of the saved bill - the field is hidden there.
