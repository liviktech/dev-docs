# 2026-11-03 - bank-accounts-payment-details-modals (backend)

## Summary

Five pieces of work, all in the same branch pair.

1. **Work Centers** master lost its downtime information, and the Edit Work Center modal's button now says **Save**.
2. **Add to Stock** (Inventory > Purchase) got UPI and Cash payment fields, its Payment Mode dropdown now comes from the **Payment Modes master**, and the modal was restyled (General Details card, breakdown table laid out like the Purchase View with a per-product discount and an HSN column, Payment Details card, tighter spacing, taller body).
3. **Multiple bank accounts** for **Party**, **Supplier** and the **Company Profile**, all in one shared table, `master.bank_account`. The old single supplier bank columns were copied into it.
4. **Payment details in both Party Approval modals** (PI Status and Final Invoice): a Payment Mode dropdown with the same fields and rules as Add to Stock, saved with the approval and shown read-only in the PI Status and Final Invoice view modals. Both modals now have a two-column layout.
5. The "**UPI**" label now shows in capitals in the Payment Modes master screen.

Both projects compile with no type errors and the frontend build passes. The database rules were checked with throwaway scripts that ran inside a transaction and rolled back (see "Test plan"). The screens were not opened by me, so the manual list below is left for the author to tick.

**Real payments and bank transfers are not included on purpose** - see "Decisions made" below. Add to Stock and the two approvals only record the payment details; nothing moves money. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/115_stock_posting_upi_cash_details.sql` (new) | Six optional columns on the Add to Stock record: sender / receiver UPI id, transaction id, paid by, received by, transaction date |
| `src/database/migrations/116_master_bank_account.sql` (new) | Shared `master.bank_account` table, constraints and indexes; copies each supplier's old bank details as its default account |
| `src/database/migrations/117_approval_payment_detail.sql` (new) | `invoice.approval_payment_detail`: how the party paid at a PI or Final Invoice approval, one row per document |
| `src/database/migrations/118_company_bank_account.sql` (new) | Company-owned accounts in the same table; extra account types; copies the company's old bank details as its primary account |
| `src/database/migrations/119_bank_account_type_length.sql` (new) | `account_type` widened to 20 characters (`CASH_CREDIT` did not fit in 10) |
| `src/modules/master/bank-account/` (new: `schema`, `types`, `repository`, `routes`) | Shared account rules: sync with a parent save, set default, soft deactivate, reveal; `GET /master/bank-accounts`, `GET /:id/reveal`, `PATCH /:id/default`, `PATCH /:id/deactivate` |
| `src/modules/master/master.routes.ts` | Mounts `/bank-accounts` |
| `src/modules/master/party/party.schema.ts`, `party.types.ts`, `party.repository.ts` | `bankAccounts` on create and update, synced in the same transaction; returned on the party read |
| `src/modules/master/stock-option/stock-option.config.ts`, `stock-option.schema.ts`, `stock-option.types.ts`, `stock-option.repository.ts`, `stock-option.routes.ts`, `stock-option.service.ts` | Suppliers take `bankAccounts` (saved with the supplier in one transaction); the four old bank fields are no longer read or written |
| `src/modules/master/company/company.schema.ts`, `company.types.ts`, `company.repository.ts`, `company.service.ts`, `company.controller.ts`, `company.routes.ts` | Profile read returns `bankAccounts` and fills `bankName`, `bankBranch`, `bankAccountNo`, `bankIfscCode` from the primary account; profile save no longer writes bank columns; five endpoints under `/master/companies/my-company/bank-accounts` |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts`, `types.ts`, `repository.ts`, `service.ts` | UPI and Cash fields with required rules; payment modes from the master and checked on save; supplier and company accounts read from `master.bank_account`; the receiver reference column now stores the supplier code; breakdown lines carry `hsnCode` |
| `src/modules/payment-detail/` (new: `schema`, `service`, `controller`) | Shared payment rules for approvals: master modes, the document's party accounts and the company accounts, validation, save once per document, read back |
| `src/modules/proforma-invoice/proforma-invoice.schema.ts`, `service.ts`, `routes.ts` | Optional `paymentDetail` saved in the party-approval transaction; `GET /:id/payment-options` and `GET /:id/approval-payment` |
| `src/modules/final-invoice/final-invoice.schema.ts`, `service.ts`, `routes.ts` | The same for Final Invoices |

## Why

- **Work Centers:** downtime is not wanted in the master data screen. It was hidden, not deleted, following the project's rule for UI removals.
- **Add to Stock fields and modes:** UPI and Cash payments had no fields, and the dropdown listed a fixed set of modes. Using the Payment Modes master means the options follow what the business configures. The restyle makes the modal read like the Purchase View and the other modals.
- **Multiple bank accounts:** parties had no bank details, suppliers and the company could hold only one account each. One shared table avoids three copies of the same rules.
- **Party Approval payment details:** the two approval modals collected an advance or a balance but kept no record of how the party paid. This reuses the Add to Stock rules so the three screens behave the same.
- **The real work** was the account rules (one default, soft deactivate, no duplicate numbers, company scoping), the transactions around them, and keeping every old reader of the company and supplier bank fields working.

## How

**Saving with the parent**: a Party or Supplier save sends the whole list. Accounts with an id are updated, accounts without one are added, and accounts left out are untouched. An account is removed from use by sending `isActive: false`, never by leaving it out. The owner's rows are locked first so two saves cannot both change the default, and an owner with active accounts always ends with exactly one default.

**Account numbers**: lists and reads send them masked (`********4567`). The full number comes only from the reveal endpoint, which needs the Master Data edit right. A saved account can be saved without its number to keep the stored one.

**Company Profile**: each action (add, edit, set primary, deactivate) saves on its own, not with Save Changes. The first account becomes primary. Ticking "Set as Primary" moves the primary. The primary account cannot be deactivated until another is primary. The profile read still returns the single-account bank fields, now taken from the primary account, so invoices, PDFs and the PI sheet are unchanged.

**Add to Stock**: the sender is the company's own account and the receiver is one of the purchase's supplier's active accounts (default first and preselected). Only the fields of the chosen mode are kept and sent. The stock lots, movements, status change and payment record are still saved in one transaction.

**Party Approval payment details**: the same modes and rules as Add to Stock. Bank Transfer needs both accounts, Cheque may carry them, UPI needs both UPI ids, the transaction id and the date, Cash needs paid by, received by and the date. The sender is the document's party's account and the receiver the company's. Everything is re-checked on the server, and a retried approval keeps the first record.

## Decisions made

**No real payment.** Add to Stock and the approvals only record details. No bank or payment integration was added.

**Migrated accounts use placeholders where nothing was recorded.** The holder is the supplier's or company's name, and the type starts as Savings (suppliers) or Current (company).

**Direction of accounts in an approval.** In Add to Stock the company pays a supplier. In an approval the party pays the company, so Sender is the party's account and Receiver the company's.

**Master payment modes that Add to Stock cannot record are left out.** Only Bank (shown as Bank Transfer), UPI, Cash and Cheque map to the stored modes. A new mode added in the master would need a migration and its own fields.

**Company accounts add with a POST.** Under the Master Data rules that needs the Add right as well as Edit.

## Bugs found while testing (already there before, not fixed here)

**1. `npm test` finds no tests.** The backend script looks for `src/**/*.test.ts`, but the existing tests live in `tests/`. Anyone running `npm test` sees 0 tests and assumes all is well.

**2. Two migrations share the same number.** `112` and `113` each exist twice (for example `112_supplier_bank_details.sql` and `112_without_material_customer_lot.sql`). They apply in filename order, so nothing breaks, but the numbering is misleading.

## Notes

- Five migrations must run in order: `115`, `116`, `117`, `118`, `119` (`npm run migrate`). `118` and `119` must both be applied before a company account with type Cash Credit can be saved.
- The supplier and company bank data were copied by the migrations. Check the migrated accounts before anyone drops the old columns.
- Old Add to Stock records, and PIs and Final Invoices approved earlier, have no payment detail. Their views show a short "not recorded" message.
- Posting stock stores a copy of the chosen accounts, so earlier postings do not change when an account is edited or deactivated.
- Lint reports errors in some touched files. They follow patterns the project already has (setState in effects, `any`), and none were introduced as new kinds.
- Nothing was committed or pushed; this file is written to disk for the PR description.

## Next steps

- Check the migrated supplier and company accounts, then add a migration that drops the old bank columns.
- Point `npm test` at the `tests/` folder, or move the tests under `src/`.
- Add a Payment Modes master option for modes beyond Bank, UPI, Cash and Cheque if the business needs them.

## Test plan

**Automated tests:**
- [x] Throwaway script, rolled back, 12 cases on the shared account rules: two accounts saved with the first as default; moving the default; number kept when omitted; deactivating the default reassigns it; a duplicate active number is refused; another owner's account id is refused; both owners at once is refused by the CHECK; two defaults are refused by the index; set default and deactivate; company isolation.
- [x] Throwaway script, rolled back, 9 cases on company accounts: first account is primary, a second is secondary, Set as Primary moves it, an edit changes only that account, the primary cannot be deactivated, deactivate is soft with one primary, a duplicate number is refused, company isolation, and the profile still returns the old single-account fields from the primary.
- [x] The four migrated supplier accounts and the two migrated company accounts were read back from the database after `116` and `118`.

**Manual checks (local):**
- [ ] Payment Modes master shows "UPI" in capitals.
- [ ] Add to Stock with a supplier that has several accounts: default preselected, another can be chosen; the saved record keeps the chosen account.
- [ ] Company Profile: View shows every active account with Primary / Secondary and masked numbers; Edit mode adds, edits, sets primary and deactivates each on its own without Save Changes; the other sections and Save Changes still work.
- [ ] PI Status view: the saved payment mode and only that mode's details show, read-only.
- [ ] Final Invoice Party Approval: invoice total, advance adjusted, amount to collect, locked amount collected; proof attach, Preview, Remove; Bank Transfer, UPI and Cash; the invoice approves as before.
- [ ] Final Invoice view: the saved payment details show, read-only.
- [ ] Invoice and PDF generation still print the company's bank details (from the primary account).
- [ ] Older records without payment details open without errors.
