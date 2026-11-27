# 2026-11-03 - bank-accounts-payment-details-modals (frontend)

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
| `src/routes/admin.work-centers.tsx` | Downtime column, the downtime line in the View Details dialog, the downtime query and its imports commented out |
| `src/features/work-centers/work-center-dialog.tsx` | Submit button label set to "Save" |
| `src/lib/pascal-text.ts` | New always-capitals list (`upi`) so "UPI" is no longer shown as "Upi" |
| `src/features/inventory/components/add-to-stock-dialog.tsx` | UPI and Cash fields with their own validation; payment modes from the master; sender and receiver details cards; new General Details card; breakdown table laid out like the Purchase View (bordered, centred cells, HSN, per-product Discount, Additional Charges / Tax and Final Amount in the footer, no own scrollbars, corner borders fixed); Payment Details in a card; tighter spacing and a taller body; typography aligned with the other modals |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Shows the saved UPI or Cash details and hides the bank cards for those modes; the breakdown table now has the HSN column |
| `src/features/inventory/purchase-types.ts` | UPI, Cash and date fields; payment modes with labels; breakdown lines carry `hsnCode`; bank accounts carry `isDefault` and `accountType` |
| `src/features/bank-accounts/types.ts` (new) | Account draft and payload types, validation, default-account helpers |
| `src/features/bank-accounts/api.ts` (new) | Call that reveals a full account number |
| `src/features/bank-accounts/bank-accounts-section.tsx` (new) | `BankAccountFields` (the six fields, masked number with Show and Change) and `BankAccountsSection` (the card used in Party and Supplier) |
| `src/features/parties/party-dialog.tsx`, `types.ts`, `api.ts`, `party-view-modal.tsx`, `src/routes/party.tsx` | Bank Details section in the Add / Edit Party modal, bank accounts carried through the types and the save payload, accounts listed in the view modal; GSTIN lookup code untouched |
| `src/features/stock-masters/stock-master-dialog.tsx`, `types.ts`, `stock-master-page.tsx` | The four single bank inputs replaced by the shared Bank Details section; accounts listed in the view panel |
| `src/features/company/company-bank-accounts.tsx` (new) | Company Profile "Bank Account Details" card (Primary / Secondary, Edit, Set as Primary, Deactivate) and the Add / Edit Bank Account dialog |
| `src/features/company/company-page.tsx`, `types.ts`, `schema.ts`, `api.ts`, `queries.ts` | Bank fields removed from the profile form; new card; calls and hooks for add, edit, set primary and deactivate; the form is no longer reset while editing when a bank account changes |
| `src/features/payment-details/types.ts`, `queries.ts`, `payment-details-fields.tsx` (new) | Shared payment form for approvals: options hook, validation, payload, the fields (stacked or side-by-side layout) and the read-only view |
| `src/features/pi-status/components/approve-party-modal.tsx` | Two-column Party Approval & Advance modal: Advance Terms & Summary on the left, Payment Recording on the right, PI and order numbers in the header, equal-height panels |
| `src/features/final-invoices/components/approve-final-invoice-modal.tsx` | Two-column Party Approval modal: Invoice & Collection Summary (invoice total, advance adjusted, amount to collect, locked amount collected, proof with Preview and Remove) and Payment Recording |
| `src/features/pi-status/api.ts`, `queries.ts`, `pi-status-page.tsx`, `components/pi-status-drawer.tsx` | Payment detail sent with the party approval; read-only "Payment Details" block in the PI Status view |
| `src/features/final-invoices/api.ts`, `queries.ts`, `final-invoices-page.tsx`, `components/final-invoice-detail-modal.tsx` | Same for Final Invoices |

## Why

- **Work Centers:** downtime is not wanted in the master data screen. It was hidden, not deleted, following the project's rule for UI removals.
- **Add to Stock fields and modes:** UPI and Cash payments had no fields, and the dropdown listed a fixed set of modes. Using the Payment Modes master means the options follow what the business configures. The restyle makes the modal read like the Purchase View and the other modals.
- **Multiple bank accounts:** parties had no bank details, suppliers and the company could hold only one account each. One shared table avoids three copies of the same rules.
- **Party Approval payment details:** the two approval modals collected an advance or a balance but kept no record of how the party paid. This reuses the Add to Stock rules so the three screens behave the same.
- **The real work** was the account rules (one default, soft deactivate, no duplicate numbers, company scoping), the transactions around them, and keeping every old reader of the company and supplier bank fields working.

## How

**Bank accounts, one table**: `master.bank_account` has a column per owner kind (party, supplier, or the company flag). A CHECK makes exactly one of them set. Partial unique indexes allow one default per owner and one active account number per owner. A default must be active.

**Saving with the parent**: a Party or Supplier save sends the whole list. Accounts with an id are updated, accounts without one are added, and accounts left out are untouched. An account is removed from use by sending `isActive: false`, never by leaving it out. The owner's rows are locked first so two saves cannot both change the default, and an owner with active accounts always ends with exactly one default.

**Company Profile**: each action (add, edit, set primary, deactivate) saves on its own, not with Save Changes. The first account becomes primary. Ticking "Set as Primary" moves the primary. The primary account cannot be deactivated until another is primary. The profile read still returns the single-account bank fields, now taken from the primary account, so invoices, PDFs and the PI sheet are unchanged.

**Approval modals**: both use one two-column shell: summary on the left, Payment Recording on the right, equal-height panels, and the sender and receiver cards kept the same height.

## Decisions made

**No real payment.** Add to Stock and the approvals only record details. No bank or payment integration was added.

**Old bank columns kept.** `master.supplier` (four columns) and `core.company` (four columns) still hold their values. The application no longer reads or writes them. Dropping them would destroy data, so that is a separate, later step.

**Migrated accounts use placeholders where nothing was recorded.** The holder is the supplier's or company's name, and the type starts as Savings (suppliers) or Current (company).

**Direction of accounts in an approval.** In Add to Stock the company pays a supplier. In an approval the party pays the company, so Sender is the party's account and Receiver the company's.

**Payment details are optional on the API, required in the modals.** Older callers (the mobile approval sheets) keep working and record nothing.

**Company accounts add with a POST.** Under the Master Data rules that needs the Add right as well as Edit.

**Add to Stock modal styling.** The Purchase View layout was followed for the table, and the per-product discount shown like New Purchase. Two bigger restyles of this modal (a two-panel layout, and the Create Final Invoice structure) were tried and rolled back by the author, so they are not part of this PR.

**Same breakdown component.** `BreakdownTable` is shared with the Purchase View. The new table layout is opt-in for Add to Stock, so the Purchase View keeps its side summary card. Both now show the HSN column.

**The approval modals are wider than the wireframes** (`sm:max-w-6xl`), at the author's request.

## Bugs found while testing (already there before, not fixed here)

**1. The payment slip hint promises PDFs.** The PI Party Approval modal says "PNG, JPG, WEBP, or PDF (up to 10MB)" but the file picker and check only accept JPEG, PNG and WebP images. It affects anyone trying to attach a PDF.

## Notes

- Old Add to Stock records, and PIs and Final Invoices approved earlier, have no payment detail. Their views show a short "not recorded" message.
- The mobile (PWA) Party Approval sheets were not changed, so they record no payment details.
- Posting stock stores a copy of the chosen accounts, so earlier postings do not change when an account is edited or deactivated.
- Lint reports errors in some touched files. They follow patterns the project already has (setState in effects, `any`), and none were introduced as new kinds.
- Nothing was committed or pushed; this file is written to disk for the PR description.

## Next steps

- Check the migrated supplier and company accounts, then add a migration that drops the old bank columns.
- Decide whether the mobile Party Approval sheets should also record payment details.
- Fix the PDF hint (or accept PDFs) on the PI Party Approval modal.
- Point `npm test` at the `tests/` folder, or move the tests under `src/`.
- Add a Payment Modes master option for modes beyond Bank, UPI, Cash and Cheque if the business needs them.

## Test plan

**Automated tests:**
- [x] Backend `tsc --noEmit` and frontend `tsc --noEmit -p tsconfig.app.json`: no errors.
- [x] Frontend `npm run build`: passes.
- [x] Throwaway script, rolled back, 12 cases on the shared account rules: two accounts saved with the first as default; moving the default; number kept when omitted; deactivating the default reassigns it; a duplicate active number is refused; another owner's account id is refused; both owners at once is refused by the CHECK; two defaults are refused by the index; set default and deactivate; company isolation.
- [x] Throwaway script, rolled back, 11 cases on approval payment details: UPI and Cash required fields, Bank Transfer needs the sender, a foreign sender account is refused, a retry adds no second row and keeps the first, other modes' fields are not stored, only UPI details read back, a missing record reads as none, and a Final Invoice Cash record reads back.
- [x] Throwaway script, rolled back, 9 cases on company accounts: first account is primary, a second is secondary, Set as Primary moves it, an edit changes only that account, the primary cannot be deactivated, deactivate is soft with one primary, a duplicate number is refused, company isolation, and the profile still returns the old single-account fields from the primary.
- [x] The four migrated supplier accounts and the two migrated company accounts were read back from the database after `116` and `118`.

**Manual checks (local):**
- [ ] Work Centers: no Downtime column or Downtime row in View Details; Edit Work Center button says "Save".
- [ ] Add to Stock: Payment Mode dropdown follows the master; Bank Transfer / Cheque show the accounts; UPI shows its four fields and Cash its three; missing fields are refused; saved details appear in the Purchase View.
- [ ] Add to Stock layout: General Details card; breakdown table with HSN, per-product Discount, Additional Charges / Tax and Final Amount rows; no scrollbars inside the table and clean corner borders; Payment Details card; spacing and height.
- [ ] Party modal: add, edit, set default, deactivate and reactivate bank accounts; numbers shown masked, Show and Change work; accounts survive a refresh; the GSTIN Fetch Details still fills the form.
- [ ] Supplier modal: the same, and the four migrated suppliers show their old bank details as the default account.
- [ ] Add to Stock with a supplier that has several accounts: default preselected, another can be chosen; the saved record keeps the chosen account.
- [ ] Company Profile: View shows every active account with Primary / Secondary and masked numbers; Edit mode adds, edits, sets primary and deactivates each on its own without Save Changes; the other sections and Save Changes still work.
- [ ] PI Status Party Approval & Advance: Bank Transfer, UPI and Cash fields and validation; advance percentage and amount calculation; slip upload, preview and remove; panels and account cards equal height; the PI approves as before.
- [ ] PI Status view: the saved payment mode and only that mode's details show, read-only.
- [ ] Final Invoice Party Approval: invoice total, advance adjusted, amount to collect, locked amount collected; proof attach, Preview, Remove; Bank Transfer, UPI and Cash; the invoice approves as before.
- [ ] Final Invoice view: the saved payment details show, read-only.
- [ ] Invoice and PDF generation still print the company's bank details (from the primary account).
- [ ] Older records without payment details open without errors.
- [ ] Not exercised: Add to Stock for a supplier with no bank accounts, and a Final Invoice or PI approved by the mobile sheet.
