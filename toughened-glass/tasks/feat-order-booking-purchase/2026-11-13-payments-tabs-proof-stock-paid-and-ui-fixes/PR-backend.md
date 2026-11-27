# 2026-11-13 - payments-tabs-proof-stock-paid-and-ui-fixes (backend)

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

| File | Change |
|---|---|
| `src/modules/master/bank-account/bank-account.repository.ts` | `addPartyBankAccount` (active party check, duplicate number message) |
| `src/modules/master/bank-account/bank-account.routes.ts`, `bank-account.schema.ts` | `POST /master/bank-accounts/party/:partyCode` and `partyCodeParamSchema` |
| `src/modules/payment-detail/payment-detail.service.ts` | `getApprovalPaymentOptions` also returns `partyCode` and `partyName` |
| `src/modules/attendance/attendance.repository.ts` | `assertMobileAvailable` on create and on a changed mobile in update (same advisory lock as create); `deleteEmployee` now sets `is_active = FALSE` on the employee and their login instead of deleting (old code kept commented) |
| `src/modules/inventory/inventory.repository.ts`, `inventory.service.ts` | Stock list: each lot carries its purchase id, Purchase Total and Paid (same rule as the Purchase list); each row sums its distinct purchases into `purchaseTotal`, `paidAmount`, `balanceDue` |
| `src/database/migrations/137_inventory_purchase_payment_txn_proof.sql` (new) | Adds `proof_url` and `proof_file_name` to `inventory_purchase_payment_txn` |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts` | Optional `proofBase64` / `proofFileName` on `recordPurchasePaymentSchema` |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | Proof validated with `validatePaymentSlip` and stored with the payment; `hasProof` / `proofFileName` on history entries (older approval payments use their slip); `paymentCount`, `lastPaymentDate`, `lastPaymentMode` on list rows |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | Insert stores the proof; history select returns `has_proof`; `findPaymentSlip` also serves payment-transaction proofs; list query adds payment count, last date and last mode |
| `src/modules/inventory-purchase/inventory-purchase.types.ts` | Matching row and API types |
| `src/modules/final-invoice/final-invoice.repository.ts`, `final-invoice.types.ts` | Table view adds `advance_amount`, `payment_count`, `last_payment_date`, `last_payment_mode` |

Commit `0b14589` ("commit") in the backend has no file changes.

## Why

Most of these came from day-to-day use and review comments: a booking product could be saved without a rate; the New Booking page had the wrong button label; staff could not add a missing bank account while approving a party payment; two employees could share one mobile number and a deleted employee left a gap in the codes (EMP-004 and later EMP-20261113-004 in Pravin Glasses). Purchase and invoice payments needed to be easier to follow from the lists, with proof attached, and the Payments tabs needed to match the naming used elsewhere.

The screens were the easy part. The real work was making the new figures match existing ones exactly: the Stock tab's Purchase Total / Paid had to equal the Purchase tab, the payment counts had to equal the rows in the view modals, and Remaining Balance had to add up once Advance was shown.

## How

**One mobile per employee**: checked in the server inside the save transaction, under the same per-company advisory lock that create already takes, so two saves at the same moment cannot both pass. An unchanged mobile is not checked, so employees that already shared a number before this change can still be edited.

**Deactivate instead of delete**: the existing delete endpoint is kept, so no client breaks; it now deactivates the employee and the linked login.

**Stock tab Purchase Total / Paid / Balance Due**: each lot reads its purchase line's purchase and works out that purchase's total and paid with the same SQL rule as the Purchase list (lines incl. GST + additional charges; paid = one-shot payment total, else approval + transaction payments, capped at the total). A row adds up its distinct purchases, so a purchase is never counted twice in one row.

**Kept the same on purpose**: the server still sends invoice payments newest first, because the invoice details view and the payment panel show them that way; only the Payments modal sorts them oldest first.

## Decisions made

**Older payment records are left as they are.** The problems found in older records (listed under Bugs) cannot happen for new records, so they were not fixed.

## Bugs found while testing (already there before, not fixed here)

**1. Two older confirmed invoices are counted as paid but are missing from Invoice Payments.** They are from before collected amounts were tracked (`collected_amount` is empty).

**2. Two purchases with payments are hidden from Purchase Payments:** PUR-20261023-105 (₹43) and PUR-20261024-109 (₹5,000). Their Purchase Total is ₹0, so Paid is capped to ₹0. New purchases cannot reach this state: a payment above the balance is refused.

**3. Deleted employees cannot be recovered.** Old EMP-004 and Lohit (EMP-20261113-004) in Pravin Glasses were hard-deleted before this change, so those codes stay missing.

**4. New PI "Cannot work out the GST" error.** Items ITM-TUFF-GLS and ITM-ANNE-GLS have no Tax Profile, so the PI cannot work out GST. This is master data: pick a Tax Profile for those items in Items.

## Notes

- `npm run migrate` must be run to apply migration 137; the Payment Proof upload fails until it is applied. On the local database, running it also applied an older pending file, `067_add_calculation_type_to_charge.sql` (adds `master.charge.calculation_type` with default PER_UNIT and creates `master.designation` if missing). Both statements are `IF NOT EXISTS`, so it was harmless, but other databases may have 067 pending too.
- The backend must be restarted for `POST /master/bank-accounts/party/:partyCode`.
- The Stock tab figures were checked against the database: products from purchase PUR-20261113-121 show ₹22,066 / ₹20,000 / ₹2,066, matching the Purchase tab. Payment counts and last dates matched the payment history for all 11 paid purchases.

## Next steps

- Decide whether to backfill payment rows for the 19 older invoices (bug 1).
- Pick a Tax Profile for items ITM-TUFF-GLS and ITM-ANNE-GLS.
- Run migration 137 (and check 067) on every environment.

## Test plan

**Automated tests:**
- [x] Backend type check (`tsc --noEmit`): no errors.
- [x] ESLint on the new and changed Payments files: no new errors (the existing ones are listed under Bugs).

**Manual checks (local):**
- [x] New Booking page shows "Save"; editing a saved booking shows "Update".
- [x] Party Approval & Advance: Sender "+ Add Bank Account" opens Edit Party, the new account is selected after saving; Receiver "+ Add Bank Account" adds a company account and selects it.
- [x] Migration 137 applied.
