# 2026-11-02 - party-gstin-supplier-bank-add-to-stock (frontend)

## Summary

Five pieces of work, all in the same branch pair.

1. Every table now opens with **20 rows per page** instead of 5 or 10.
2. The **Add / Edit Party** modal lost its three tabs and became one two-column view (Party Information on the left, Registered Addresses and Architectural Projects on the right). It also got a **GSTIN "Fetch Details"** lookup that fills the form from the GST data provider (gstinapi.in), and a new optional **PAN** field.
3. The **Add / Edit Supplier** modal has a new **Bank Details** section (bank name, branch, IFSC, account number).
4. **Add to Stock** on a purchase now opens a modal that shows the amount breakdown and asks for the payment mode and the company and supplier bank accounts, then adds the purchase to stock. What was chosen is saved and shown in the Purchase View.
5. `.env.example` now explains the new `GSTINAPI_KEY` setting.

The author checked the changes in the browser. Both projects compile with no type errors, and the new GSTIN lookup has automated tests with a mocked provider. The live provider's error cases were not exercised (see "Test plan").

**Real bank transfers are not included on purpose** - see "Decisions made" below. Add to Stock only records the payment details; nothing moves money. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/components/ui/data-table.tsx` | Default page size 10 to 20. The 5 / 10 / 15 / 20 / 25 dropdown was already there and is unchanged |
| `src/lib/list-url-state.ts` | `DEFAULT_PAGE_SIZE` 10 to 20 (lists that keep the size in the web address) |
| `src/routes/admin.charges.tsx`, `admin.invoice-terms.tsx`, `admin.notifications.tsx`, `admin.order-charges.tsx`, `admin.payment-terms.tsx`, `admin.sales-reps.tsx`, `admin.sequences.tsx`, `admin.shifts.tsx`, `admin.tax-profiles.tsx`, `admin.users.tsx`, `admin.work-centers.tsx`, `party.tsx` | Initial page size set to 20 |
| `src/routes/work-orders.tsx`, `src/features/order-booking/header-detail-tab.tsx` | Default 20; the "hide the size from the address when it is the default" check now compares with 20 |
| `src/routes/invoices.tsx`, `src/features/production-plan/production-plan-panel.tsx`, `src/features/dispatch-plan/dispatch-plan-panel.tsx` | Initial page size set to 20 (server-side paging, request code unchanged) |
| `src/features/attendance/attendance-page.tsx`, `attendance/users-tab.tsx`, `glass-thickness/glass-thickness-page.tsx`, `inventory/locations-tab.tsx`, `inventory/movements-tab.tsx`, `notifications/messages-tab.tsx`, `products/product-details-tab.tsx`, `roles/roles-access-page.tsx`, `stock-masters/stock-master-page.tsx` | Initial page size set to 20 |
| `src/features/reports/` (`fir`, `inventory/batch-consumption`, `inventory/offcut-register`, `performance/sales-rep-achievement`, `performance/sales-rep-summary-reports`, `production/department-report`, `production/production-detail-report`, `sales-income/use-report-page`, `visit-register`) | Initial page size set to 20 |
| `src/features/parties/party-dialog.tsx` | Tabs removed; two-column layout; Party Information card with GSTIN box and Fetch Details, Party, Sales Executive, PAN, Mobile / WhatsApp, Email; address cards; projects table; GSTIN autofill logic; PAN field and its check; the per-address GSTIN input commented out (the top GSTIN box is the single GSTIN field, saved on the default address); shared "Add" button |
| `src/features/parties/gstin-lookup-bar.tsx` (new) | GSTIN box and Fetch Details button; checks format and check character locally; calls the server only on click or Enter; ignores double clicks. The fetched-details panel is commented out |
| `src/features/parties/gstin-lookup.ts` (new) | Result type and the call to `GET /gstin/lookup/:gstin` |
| `src/features/parties/types.ts`, `api.ts`, `party-view-modal.tsx` | Party `pan` carried through the types, the save payload and the view modal |
| `src/lib/gstin.ts` (new) | GSTIN format and check-character test, and GST state code to state name |
| `src/features/stock-masters/stock-master-dialog.tsx`, `types.ts`, `stock-master-page.tsx` | Supplier Bank Details section (IFSC upper-cased and checked, account number digits only), loaded on edit, shown in the view panel |
| `src/features/inventory/components/add-to-stock-dialog.tsx` (new) | The Add to Stock modal: amount breakdown table and Payment Summary card, Payment Mode, Sender and Receiver account fields and cards. Also exports the breakdown table and account card reused by the Purchase View |
| `src/features/inventory/purchase-tab.tsx` | Add to Stock opens the new modal; the old confirmation is commented out; the modal stays open if saving fails |
| `src/features/inventory/purchase-api.ts`, `purchase-queries.ts`, `purchase-types.ts` | Add to Stock now sends the payment details; new call and hook for the modal's options; new types |
| `src/features/inventory/components/purchase-view-dialog.tsx` | New read-only section "Stock Transfer & Payment Details" |

## Why

- **Pagination:** a bigger default means fewer clicks on long lists. This was a small change because the shared table and the URL-state helper hold most of the setting.
- **Party modal and GSTIN lookup:** the three tabs hid information, and typing a party's name, address and PAN by hand is slow and error-prone. The wireframe asked for one view.
- **Supplier bank details and Add to Stock:** Add to Stock used to move stock with a plain confirmation and keep no record of how or to whom the supplier was paid. This records that, using the bank details of the company and the supplier. The real work was the transaction and the server checks, not the screen.
- **The GSTIN lookup guide** (`docs/GSTIN_LOOKUP_FEATURE_GUIDE.md`) drove the lookup design. It was adapted, not copied: the app has no state dropdown, and GSTIN lives on the address, not the party.

## How

**GSTIN lookup**: the key stays on the server. The lookup runs only when the user clicks Fetch Details, never while typing. The server re-checks the GSTIN, then calls the provider. A provider 401, 402 or 403 is returned to the browser as 502, so an invalid key cannot log the user out.

**Autofill rules**: Fetch Details fills Party name (trade name, else legal name), PAN, Address Line 1, City, State and Pincode on the default address. It fills only blank fields, or fields that an earlier fetch filled and the user has not edited since. Fetching a different GSTIN therefore replaces the old values, but never anything the user typed. A kept value is named in the message. A registration that is not Active shows a warning.

**GSTIN field**: the top box is now the only GSTIN field. It opens with the saved default address's GSTIN, is checked on Save, and is written onto the default address. The data model was not changed: GSTIN is still stored on the address.

**Modal layout**: left card (40%) and right column (60%) on large screens, stacked on small ones. When the modal opens, the left card takes the height of the right column and keeps it as rows are added.

**Supplier bank details**: suppliers run through one shared code path, so the four fields were added to a field list, not to separate code for each layer.

**Add to Stock**:

- The amounts come from the purchase's stored values: each line's total, the discount worked out as gross minus net, the invoice charges and the invoice total. Nothing is recalculated differently.

- Payment Mode is required. Bank Transfer needs both accounts; Cheque shows them but does not require them; Cash and UPI hide them, and the server ignores any account sent with those modes.

- Only the masked account number is sent to the browser (for example `********0188`).

## Decisions made

**No real bank transfer.** Add to Stock records the payment details only. No bank or payment integration was added, as required.

**One bank account per company and per supplier.** The company profile and the supplier master each hold one set of bank details, and I did not create a separate bank-account master. So each dropdown lists one account today. The dropdowns and checks accept several, so a future bank-account master would not need a rewrite of the modal.

**Bank accounts required for Bank Transfer only.** Cheque shows the fields but does not require them. Cash and UPI hide them.

**PAN needs its own column.** The Party record had nowhere to keep PAN, so a small optional column was added (migration 114). Other values the lookup returns (registration status, taxpayer type, registration date, constitution) have no Party field and were not stored, to avoid more database changes.

**Mobile keeps a fixed +91.** The wireframe showed a country-code selector, but saving builds `91` plus a 10-digit Indian number and the server validates that exactly. A selector would change business logic.

**No Active Status column on addresses and projects.** Neither has an active/inactive field in the data model. The wireframe's "[x]" was treated as the delete action.

**Address Line 2, State and Pincode kept.** A one-line table row cannot hold every address field, so addresses became cards, with all fields labelled above their inputs.

**Server cache for lookups.** Results are kept in server memory for 3 days to save provider credits. It resets on restart. A shared cache was not built.

**Removed items were commented out, not deleted.** This follows the project's rule for UI removals (the old confirmation modal, the fetched-details panel, the per-address GSTIN input, the "Party: name" badge).

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- The provider's response field names come from the guide. The registration date and constitution names were guessed; if they differ, those two values stay blank. They are not stored, so nothing breaks.
- A GSTIN that passes the format and check-character test is only well formed. The screen never says it is verified.
- An address that has no GSTIN saved still shows as blank; other addresses of a party keep their old stored GSTIN, which can no longer be edited from the modal.
- The page-size change touches many files but only the starting value.

## Next steps

- Decide whether a bank-account master (several accounts for the company and each supplier) is wanted. That is what would make the Sender and Receiver dropdowns offer a real choice.
- Run one lookup with the provider's sandbox GSTIN and check the real response, then decide whether status, taxpayer type or registration date deserve their own Party fields.
- Consider a per-user limit on GSTIN lookups. Only the app's global rate limit applies today.
- Rename the "Registered Addresses & GSTIN" wording if it still appears anywhere; the address-level GSTIN field is gone.

## Test plan

**Automated tests:**
- [x] `tests/gstin-lookup.test.ts`: 9 cases pass - the reference GSTINs and bad check character or format; reply mapping with the missing-field fallbacks; a bad GSTIN never reaches the provider; missing key gives 503; 404, 429, 401/402/403 (as 502) and 500 mapping; an unexpected 200 body; timeout gives 504; simultaneous requests share one call; failures are not cached and entries expire.
- [x] `tests/purchase-cost.test.ts`, `tests/inventory-purchase-create.test.ts`: 21 cases pass (existing purchase tests, run after the Add to Stock change).
- [x] Backend `tsc --noEmit` and frontend `tsc -b --noEmit`: no errors.
- [ ] Authentication on `GET /gstin/lookup/:gstin` - uses the existing sign-in check; no separate test was written.

**Manual checks (local):**
- [x] Tables open with 20 rows and the size can be switched.
- [x] Add / Edit Party modal: the two-column layout, the address cards, projects, and saving a party.
- [x] Fetch Details with a valid GSTIN fills Party name, PAN, address, city, state and pincode into the right fields; the GSTIN box saves and reloads on edit.
- [x] Supplier Bank Details: saving, editing and the view panel.
- [x] Add to Stock: the modal opens instead of a confirmation, the breakdown is shown, the accounts show for Bank Transfer and hide for Cash and UPI, the Purchase View shows the saved details.
