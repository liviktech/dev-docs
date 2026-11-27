# 2026-11-13 - payments-tabs-proof-stock-paid-and-ui-fixes (frontend)

## Summary

A set of changes across Order Booking, Payments (Party Approval, Purchase, Final Invoice), Inventory (Purchase, Stock, Payments tabs), Employee and Work Order. Branches: `feat/ui-changes` (frontend) and `feat/order-booking-backend` (backend).

**Order Booking**
1. **Rate is required in the Add Product modal.** The Rate (per SQM) label has a red asterisk and Save shows "Rate is required" when it is empty. Without Material products (rate set to 0, field hidden) still save.
2. **New Booking button reads "Save".** On the New Booking page the header button said "Update"; it now says "Save". Editing a saved booking still says "Update".
3. **Products table columns.** New Chargeable SQM and Material Charges columns per product. Order is now: Product, Thickness, Chargeable SQM, Rate (per sqm), Material Charges, Charges, Qty, Sizes, Photos, Total, Actions. The same table is used in the New PI flow.

**Payments: Party Approval & Advance**
4. **Quick add on Sender and Receiver Account.** Sender Account "+ Add Bank Account" opens the party's Edit Party modal (party bank accounts are kept in the Party master). Receiver Account opens "Add Company Bank Account". After saving, the list reloads and the new account is selected. The same fields are used by the Final Invoice payment panel and its mobile sheet, so they get it too.

**Inventory: Purchase Payment dialog**
5. **Sender Account defaults to the company's primary account.** The user can still pick another one.
6. **Optional Payment Proof upload** on each new payment (JPEG, PNG, WebP or PDF, up to 5 MB, same as the supplier bill). A correction request keeps the payment's proof.
7. **Add Payment layout.** First row: Payment Mode, Payment Amount, Payment Date, Payment Proof. The fields of the chosen mode follow below.
8. **Purchase Payment Summary** fields now spread across the full card width, with no empty space after the right column.

**Inventory: Purchase and Stock tabs**
9. **Purchase tab: Paid (₹) column** restored between Purchase Total and Balance Due.
10. **Stock tab: Purchase Total, Paid and Balance Due columns,** showing the full figures of the purchase(s) the product came from, exactly as the Purchase tab shows them. Purchase Rate and Cost Unit columns are hidden.

**Payments tabs (Inventory > Payments and Final Invoice > Payments)**
11. **New list columns.** Both lists: No. of Payments, Last Payment Date, Last Payment Mode. Invoice Payments also: Advance (PI Share), so Total Amount − Advance − Collected = Remaining Balance adds up on screen.
12. **View modals.** New header fields (Purchase: Supplier Invoice No, Goods Received Date, Last Payment Date; Invoice: Invoice Date, Work Order Number, Last Payment Date). New Proof column with a View button in the payment table. Fields spread across the full card width. Both modals are wider (1400px) so the payment table does not scroll sideways.
13. **Invoice Payments modal.** Payments listed oldest first, like Purchase Payments. Payment Mode moved onto the Recorded On / Recorded By line in the PI Payment Details card.
14. **Invoice Payments column names match the main Final Invoice table:** FI Number, Party, PI Number, Work Order Number, Total Amount, Remaining Balance.

**Employee**
15. **One mobile number per employee.** Add and Edit Employee reject a mobile number already used by another employee, with a message naming that employee.
16. **No more hard delete.** The Delete action is removed from the Employees list and the mobile employee screen. The server's delete request now deactivates the employee and their login instead of removing the row, so employee codes never get gaps and attendance and payroll history stay.

**Work Order**
17. **Progress and Delivery Date columns swapped** in the Work Order table.

**Database**
18. **Migration 137 (new)** adds `proof_url` and `proof_file_name` to `public.inventory_purchase_payment_txn`.

All changes were tested in the browser by the author. Both projects type-check with no errors. No automated tests were added.

**Fixing older payment records is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

**Order Booking**

| File | Change |
|---|---|
| `src/features/order-booking/schema.ts` | `orderBookingItemSchema`: adds "Rate is required" when rate is empty (Without Material's "0" passes) |
| `src/features/order-booking/product-detail-panel.tsx` | Red asterisk on the Rate (per SQM) label |
| `src/features/order-booking/booking-header-form.tsx` | New booking submit button label "Update" changed to "Save" |
| `src/features/order-booking/products-tab.tsx` | New Chargeable SQM and Material Charges columns (summed from each product's sizes); columns reordered; Qty and Sizes placed before Photos |

**Payment details (Party Approval & Advance, Final Invoice payment panel)**

| File | Change |
|---|---|
| `src/features/payment-details/payment-details-fields.tsx` | "+ Add Bank Account" on every Sender and Receiver Account dropdown; Sender opens `PartyDialog` (edit mode) and saves through `useUpdateParty`, Receiver opens `QuickAddBankAccountDialog` (company); new account selected after reload; `isAddModeOpen` also covers these dialogs so the parent modal hides; `ApprovalPaymentView` gets an optional `hideMode` prop and skips its line list when empty |
| `src/features/payment-details/types.ts` | `partyCode` and `partyName` on `ApprovalPaymentOptions` |
| `src/features/bank-accounts/quick-add-bank-account-dialog.tsx` | New `party` owner kind (title, holder name prefill, `POST /master/bank-accounts/party/:code`); currently not used by any screen, see Notes |

**Inventory: Purchase Payment dialog**

| File | Change |
|---|---|
| `src/features/inventory/components/purchase-payment-dialog.tsx` | Sender defaults to the company's primary account (kept if the user already chose one); optional Payment Proof upload sent as `proofBase64` / `proofFileName`, cleared when the form reopens, hidden for corrections; Add Payment first row is Mode, Amount, Date, Proof; summary grid spread to full width |
| `src/features/inventory/purchase-types.ts` | `hasProof` / `proofFileName` on payment history entries; `proofBase64` / `proofFileName` on `RecordPurchasePaymentInput`; `paymentCount`, `lastPaymentDate`, `lastPaymentMode` on list rows |

**Inventory: Purchase and Stock tabs**

| File | Change |
|---|---|
| `src/features/inventory/purchase-tab.tsx` | Paid (₹) column switched back on (it was commented out) |
| `src/features/inventory/stock-tab.tsx` | Purchase Total, Paid, Balance Due columns; Purchase Rate and Cost Unit columns commented out |
| `src/features/inventory/types.ts` | `paidAmount`, `purchaseTotal`, `balanceDue` on the Stock row |

**Payments tabs**

| File | Change |
|---|---|
| `src/components/payments/payment-columns.tsx` (new) | `lastPaymentColumns()`: shared No. of Payments, Last Payment Date and Last Payment Mode columns |
| `src/components/payments/payments-shared.tsx` | `PaymentsModal` gets an optional `width="wide"` (1400px); `PaymentHistoryTable` gets a Proof column |
| `src/features/inventory/purchase-payments-tab.tsx` | New list columns; modal header fields; View proof per payment (`PaymentSlipLink`); wide modal; summary spread to full width |
| `src/features/final-invoices/components/final-invoice-payments-tab.tsx` | Advance (PI Share) and the three payment columns; renamed columns and modal labels; header fields; View proof per payment (`ProofButton` + `PaymentSlipViewer`); payments sorted oldest first; Payment Mode on the Recorded On / Recorded By line; wide modal; summary spread to full width |
| `src/features/final-invoices/api.ts`, `types.ts` | Map `payment_count`, `last_payment_date`, `last_payment_mode` |

**Employee**

| File | Change |
|---|---|
| `src/features/attendance/attendance-page.tsx` | Delete action, its confirm modal and their imports commented out |
| `src/features/pwa/attendance/mobile-employees.tsx` | Same on the mobile employee sheet |

**Work Order**

| File | Change |
|---|---|
| `src/routes/work-orders.tsx` | Progress and Delivery Date column definitions swapped |

## Why

Most of these came from day-to-day use and review comments: a booking product could be saved without a rate; the New Booking page had the wrong button label; staff could not add a missing bank account while approving a party payment; two employees could share one mobile number and a deleted employee left a gap in the codes (EMP-004 and later EMP-20261113-004 in Pravin Glasses). Purchase and invoice payments needed to be easier to follow from the lists, with proof attached, and the Payments tabs needed to match the naming used elsewhere.

The screens were the easy part. The real work was making the new figures match existing ones exactly: the Stock tab's Purchase Total / Paid had to equal the Purchase tab, the payment counts had to equal the rows in the view modals, and Remaining Balance had to add up once Advance was shown.

## How

**Party bank account quick add**: the Party master is the single place for party bank details, so Sender Account opens the existing Edit Party modal for the invoice's party (the party code is now part of the payment options). Before the dialog opens, the hook records the current sender account ids; after the party is saved and the options reload, the account that was not there before is selected.

**Deactivate instead of delete**: the existing delete endpoint is kept, so no client breaks; it now deactivates the employee and the linked login.

**Payment proof**: stored on the payment-transaction row as a data URI, the same way the supplier bill is stored. The history only carries whether a proof exists; the file is read on demand through the existing `GET /inventory-purchases/:id/payments/:paymentId/slip`, which now also looks in the transaction table. A correction does not touch the proof columns.

**Payments list columns**: counted from the same entries as the payment history (one-shot payment, else approval payments, plus recorded payments), so No. of Payments always equals the rows in the view modal. Invoice payments come from `invoice.approval_payment_detail`; the latest one's date is its transaction date, else the day it was recorded.

**Kept the same on purpose**: the server still sends invoice payments newest first, because the invoice details view and the payment panel show them that way; only the Payments modal sorts them oldest first.

## Decisions made

**Stock tab shows full purchase figures, not a share.** A share per product (split by line value) was built first; it was replaced at the author's request so the Stock tab shows the same figures as the Purchase tab. Products from the same purchase therefore show the same numbers, and these columns should not be totalled.

**Sender quick add opens Edit Party, not a separate bank form.** A standalone "Add Party Bank Account" form was built first; it was changed so party bank accounts are only ever added in the Party master.

**Mobile uniqueness is enforced in the app, not by a database index.** The data already has duplicates (see Bugs), so a unique index cannot be added yet. A database rule was judged not necessary for now because the Employee module is the only code that writes employees.

**Older payment records are left as they are.** The problems found in older records (listed under Bugs) cannot happen for new records, so they were not fixed.

**Proof upload added to the purchase Payment dialog.** The Proof column needed somewhere to store a file; the other options were a header "View Bill" link only, or a column filled only for older slips.

**Remove means comment out.** The employee Delete action and the Stock tab's Purchase Rate / Cost Unit columns are commented out, not deleted.

## Bugs found while testing (already there before, not fixed here)

**1. 19 final invoices have a Collected amount but no payment rows.** Older invoices saved an "amount collected" at creation without a payment row, so their Payments modal says "No payment has been recorded yet" and the new columns show ", ". One more invoice (FI-20261010-109) has a payment row with no amount. New invoices are not affected: creation now sends 0 and every payment amount must be above 0.

**2. Two older confirmed invoices are counted as paid but are missing from Invoice Payments.** They are from before collected amounts were tracked (`collected_amount` is empty).

**3. Two purchases with payments are hidden from Purchase Payments:** PUR-20261023-105 (₹43) and PUR-20261024-109 (₹5,000). Their Purchase Total is ₹0, so Paid is capped to ₹0. New purchases cannot reach this state: a payment above the balance is refused.

**4. Employees already sharing a mobile number:** `0000000000` (placeholder from the app-user backfill) on 9 employees across two companies, `1234567890` (EMP-001 Kamal nath, EMP-002 Suresh) and `9988776655` (Kumar A, Johnson A). They can still be edited; a unique database index needs these fixed first.

**5. Deleted employees cannot be recovered.** Old EMP-004 and Lohit (EMP-20261113-004) in Pravin Glasses were hard-deleted before this change, so those codes stay missing.

**6. New PI "Cannot work out the GST" error.** Items ITM-TUFF-GLS and ITM-ANNE-GLS have no Tax Profile, so the PI cannot work out GST. This is master data: pick a Tax Profile for those items in Items.

**7. Existing lint errors** in `payments-shared.tsx` (non-component exports, react-refresh rule) and the reset effect in `purchase-payment-dialog.tsx` (set-state-in-effect rule). Not from this change.

## Notes

- `POST /master/bank-accounts/party/:partyCode` and the `party` owner kind in `QuickAddBankAccountDialog` are no longer used by any screen (the Sender quick add now opens Edit Party). They were left in place, not removed.
- The Stock tab figures were checked against the database: products from purchase PUR-20261113-121 show ₹22,066 / ₹20,000 / ₹2,066, matching the Purchase tab. Payment counts and last dates matched the payment history for all 11 paid purchases.
- The Payments tabs build on the Payments feature added earlier today in commits `eba9cf9` (frontend) and `aef2dbe` (backend).
- The purchase Payment dialog's own width was not changed (an increase was tried and reverted).

## Next steps

- Decide whether to backfill payment rows for the 19 older invoices (bug 1).
- Fix the duplicate employee mobile numbers, then consider a unique database index.
- Pick a Tax Profile for items ITM-TUFF-GLS and ITM-ANNE-GLS.
- Remove the unused party bank-account endpoint and dialog kind, or keep them for a future screen.

## Test plan

**Automated tests:**
- [x] Frontend type check (`tsc --noEmit -p tsconfig.app.json`): no errors.
- [x] ESLint on the new and changed Payments files: no new errors (the existing ones are listed under Bugs).

**Manual checks (local):**
- [x] Add Product: Rate shows an asterisk and Save is blocked with "Rate is required" when empty; a Without Material product still saves.
- [x] New Booking page shows "Save"; editing a saved booking shows "Update".
- [x] Products table shows Chargeable SQM and Material Charges, in the new column order, with Qty and Sizes before Photos.
- [x] Party Approval & Advance: Sender "+ Add Bank Account" opens Edit Party, the new account is selected after saving; Receiver "+ Add Bank Account" adds a company account and selects it.
- [x] Purchase Payment dialog: Sender defaults to the primary company account; Payment Proof uploads and saves; Add Payment row order is Mode, Amount, Date, Proof; summary fields reach the right edge.
- [x] Purchase tab shows Paid; Stock tab shows Purchase Total, Paid, Balance Due matching the Purchase tab, with Purchase Rate and Cost Unit hidden.
- [x] Purchase Payments and Invoice Payments lists show the new columns; Invoice Payments rows add up with Advance shown; column names match the Final Invoice table.
- [x] View modals show the new header fields and the Proof View button, do not scroll sideways, and list invoice payments oldest first; Payment Mode sits on the Recorded On / Recorded By line.
- [x] Employee: a duplicate mobile is rejected on add and edit; Delete is gone and a deactivated employee moves to Inactive with its code.
- [x] Work Order table shows Progress before Products and Delivery Date after Plan.
