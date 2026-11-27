# 2026-11-11 - purchase-invoice-payments-cost-and-edit-approval (backend)

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

**Duplicate invoice, dates, item matching**

| File | Change |
|---|---|
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | `assertInvoiceNoFree`, `checkSupplierInvoice`, `duplicateInvoiceError`; received date must not be before the invoice date (400 `INVENTORY_PURCHASE_RECEIVED_BEFORE_INVOICE`) on create and update; `resolveItems` uses the new matcher and refuses ambiguous matches; update accepts a replacement bill |
| `src/modules/inventory-purchase/purchase-item-match.ts` (new) | Exact, most-specific, ambiguous matching |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | `normalizeInvoiceNo`, `findPurchaseByInvoiceNo`, `setInvoiceSlip`, `hasPayments` (amount above 0 only) |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts`, `.controller.ts`, `.routes.ts`, `.types.ts` | Check-invoice query and route (declared before `/:id`), update schema takes the bill, "Goods received date" messages, controller passes the user to update |
| `src/database/migrations/130_inventory_purchase_unique_supplier_invoice.sql` (new) | Unique index on company + supplier + normalized invoice number. **Not applied** - see "Notes" |

**LOT numbers and stock cost**

| File | Change |
|---|---|
| `src/modules/inventory/inventory.repository.ts` | LOT counter taken per company; Stock query joins each lot to its purchase line for cost |
| `src/modules/inventory/inventory.service.ts` | `toLotTableRow` works out cost state, purchase rate, cost unit, stock value and lots without cost |

**Payments**

| File | Change |
|---|---|
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | Totals query returns approval-paid, transaction-paid and paid amount; `payment_recorded`; transaction read, exists and insert functions |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | Paid rule `stockPosting ? total : MIN(approval_paid + txn_paid, total)`; `paymentStatusOf`, `toPaymentHistory`; rewritten `recordPurchasePayment` (row lock, retry-safe request id, overpay refused); the edit / delete guard also counts the new payments |
| `src/database/migrations/131_inventory_purchase_payment_transaction.sql` (new) | New `inventory_purchase_payment_txn` table. Applied |
| `src/database/migrations/132_inventory_purchase_invoice_record_zero_amount.sql` (new) | The invoice record may hold 0 so a bill can be stored on a purchase saved without rates. Applied |

**Edit request with admin approval (partially completed, not tested)**

| File | Change |
|---|---|
| `src/database/migrations/133_inventory_purchase_edit_request.sql` (new) | New `inventory_purchase_edit_request` table (one pending request per purchase and per payment), and edit audit columns on the payment table. Applied |
| `src/modules/inventory-purchase/purchase-edit-request.repository.ts` (new) | Request reads and writes, locked reads, and the fixed list of fields an approval may write |
| `src/modules/inventory-purchase/purchase-edit-request.service.ts` (new) | Raise, approve, reject and cancel; checks again at approval; applies in one transaction |
| `src/modules/inventory-purchase/purchase-edit-request.schema.ts`, `.controller.ts` (new) | Input checks and handlers |
| `src/modules/inventory-purchase/inventory-purchase.routes.ts` | List, approve, reject, cancel, raise-purchase-request and raise-payment-request routes; approve and reject need ADMIN or SUPER_ADMIN |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | `resolvePaymentDetails` moved out of `recordPurchasePayment` so recording and correcting a payment use the same checks; a few helpers exported |

## Why

These came from a series of specs on the Purchase module: duplicate invoices, per-company LOT numbers, the wrong paid state, partial payments, date rules, lot cost, wrong item selection, several modal fixes, and finally a correction workflow for purchases that are already in stock or paid. The easy parts were labels and the modal layout. The real work was the payment rules (so every screen shows the same figures), concurrency (two payments at once) and the approval flow.

## How

**Duplicate invoice**: the invoice number is compared trimmed, with inner spaces collapsed and in lower case, per company and supplier. The form checks it while typing, the API checks it again, and the database index (migration 130) is the last guard.

**Payment rules**: the server decides the paid amount and the status. The purchase row is locked while a payment is recorded, so two payments at once cannot overpay. A payment form carries a request id, so a double click records it once. The older one-shot payment (stock posting) still counts as fully paid and is shown as one payment.

**Item matching**: an item must match glass type, colour and thickness exactly. Among matches, the one with the fewest colours wins. If two are equally specific the save is refused and both item codes are named.

**Lot cost**: each lot is joined to its purchase line. The cost columns are worked out on the server from that line, so no cost is stored twice.

**Edit request**: a correction is stored as "before" and "proposed" values with a reason. The real records are not touched. On approval, in one transaction, the purchase row is locked, the request is checked to be still pending, the current values are compared with the "before" values (if they differ the approval is refused as out of date), the same validations run again, and then the purchase, lines, invoice record, received-stock rate or the payment are written. If anything fails, nothing is written and the request stays pending. The requester cannot approve their own request. Totals, paid amount, balance and status are worked out from the saved rows, so they follow the correction.

## Decisions made

**What a purchase correction can change.** Invoice number, invoice date, additional charges, remarks, and per product the rate, rate unit, discount, HSN, supplier item code and description. Not supplier, product, glass, colour, size, quantity, location or Goods Received Date, because those make up the stock lots (the received date is copied onto them). Easy to widen later.

**Which payments can be corrected.** Only those recorded through the Payment option. Older one-shot payments show "Older payment" and cannot be corrected.

**Products that cannot be re-priced.** A product received in parts at different rates, or on a purchase paid the old one-shot way, is refused with a message.

**Admins also need a second admin.** The spec says nobody approves their own request, so an admin's own request also goes to another admin. A company with one admin cannot approve its own requests.

**Edit Payment needs the Add right.** It reuses the payment-options call, which needs the inventory Add right. A user with Edit but not Add cannot use it.

**Migration 130 is not applied.** Three existing groups of duplicate supplier invoices would make it fail. They have to be fixed first.

**Stock value uses gross or net discount?** Left open for the team to decide.

## Bugs found while testing (already there before, not fixed here)

**1. Bill dropped silently on a zero-total purchase.** A purchase saved without rates (total 0) had no invoice row to hold a bill, so an uploaded bill was lost (seen on PUR-20261029-114). The old rule needed every payment row to be above 0. This was fixed here with migration 132, but that purchase must have its bill uploaded again.

## Notes

- Migrations 131, 132 and 133 were applied one by one to the configured database. 130 was not.
- The backend must be restarted to pick up the new routes.
- Purchase PUR-20261029-114 needs its bill uploaded again.
- The approval flow was exercised only by a backend script that ran inside a transaction and was rolled back, then confirmed that no request rows and no changed amounts were left behind. It was not used from the screens.

## Next steps

- Fix the three duplicate supplier-invoice groups, then apply migration 130.
- Decide on gross or net discount in stock value.
- Add automated tests for the payment rules and the approval flow.

## Test plan

**Automated tests:**
- [ ] No test files were added or changed. The checks below are not automated.

**Manual checks (local):**
- [x] Backend scripts on real data (rolled back or test rows deleted): partial, retried and over-limit payments; two payments at once (one succeeded, history intact); LOT counters; lot cost against an independent query for 22 lots; 16 item-matching cases; the date rejection cases.
- [x] Edit request flow, backend only, in a rolled-back transaction on a purchase that was in stock with a payment: a rate change raised the total from 1000 to 1100 and the invoice record and received-stock rate followed; a payment correction moved the paid amount from 400 to 399; a duplicate pending request, self-approval, a non-admin approval, a second approval, an out-of-date approval, a total below the paid amount, a future payment date and an amount above the total were all refused. Nothing was left behind.
- [ ] Bank transfer and cheque payments end to end - the test supplier had no bank accounts, so only the checks were covered.
- [ ] Migration 130 on the real database - not applied because of existing duplicates.
