# 2026-11-10 - routing-masters-signature-removal-payment-quick-add (frontend)

## Summary

Eight pieces of work, all in the same frontend / backend branch pair.

1. **Washing stage modal** (Order Booking > Add Product > Work-Centre Routing). Before, the Washing stage looked different from the other stages and nothing happened when it was clicked. It now opens a Washing Detail modal with "Process Required" (Yes / No) and a Remark, saved with the product like the other stage details.
2. **Cutting Type, Edge Type and Grinding Type** in the Cutting and Grinding stage modals are now searchable dropdowns from three new masters, each with a Quick Add. Each master is seeded with four options.
3. **Quick Add for Wave Limits, Bend Limits and Load Direction** in the Tempering Detail modal.
4. **"None" removed** from those routing-stage dropdowns. An empty dropdown already shows "Select".
5. **Digital signatures removed completely.** They no longer appear in any preview or PDF (Proforma Invoice, Final Invoice, E-Invoice, Purchase Order, Gate Pass, emailed PI / Final Invoice PDF) or on the Users page. The backend no longer sends or accepts them, and signatures saved in the browser are cleared.
6. **Add Rate / Add Charge** in the Add / Edit Product modal now use the shared `AddButton` component.
7. **Bank account Quick Add in the Purchase Payment modal.**
   - **Sender Account:** "+ Add Bank Account" opens a small Add Company Bank Account form.
   - **Receiver Account:** "+ Add Bank Account" opens the existing Edit Supplier modal for the purchase's supplier.
   - **After saving:** the new account is selected.
8. **Any master payment mode in the Purchase Payment modal, plus a Quick Add for Payment Mode.**
   - **The four built-in modes** (Bank Transfer, UPI, Cash, Cheque) keep their own fields.
   - **Any other mode** added in the master (Card, NEFT ...) shows the Cash fields: Paid By, Received By and Transaction Date.

Both projects typecheck. All changes were tested in the browser.

Existing records keep working:
- **Old free-text values.** Cutting / Edge / Grinding values typed before this change (e.g. "cnc", "flat") still show in the dropdown as the selected value and are not lost.
- **Old purchase payments.** Payments recorded before item 8 still show their mode correctly.

**The PI and Final Invoice payment screens are not included on purpose** in item 8 - see "Decisions made" below. They still offer only the four built-in modes. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/work-center-detail-modal.tsx` | New `WashingDetailModal`; new `MasterOptionField` (searchable master dropdown with Quick Add, keeps an unmatched old value as an extra option, Retry on load error) used for Cutting Type, Edge Type, Grinding Type, Load Direction, Wave Limits, Bend Limits; `QuickAddMasterDialog`; the old `TemperingOptionField` and the "None" option commented out |
| `src/features/order-booking/process-details.ts` | `WASH` added as a routing process code with its own detail state |
| `src/features/order-booking/schema.ts` | `washingDetailSchema` (process YES / NO, remark) |
| `src/features/order-booking/api.ts` | Washing detail read from the item bundle and sent on save |
| `src/features/order-booking/add-product-modal.tsx` | Loads the saved washing detail into the form and saves it with the product |
| `src/features/stock-masters/config.ts` | `CUTTING_TYPE_DEF`, `EDGE_TYPE_DEF`, `GRINDING_TYPE_DEF` |
| `src/features/stock-masters/queries.ts` | Hooks for the three new masters and their lookup datasets |
| `src/lib/lookup.ts` | `cuttingTypes`, `edgeTypes`, `grindingTypes` in the central lookup and its refresh list |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx` | Signature images commented out; blank signature lines kept |
| `src/lib/pdf/proforma-invoice-pdf.ts` | Signature images no longer drawn on the PDF |
| `src/features/e-invoice/components/einvoice-tax-invoice-sheet.tsx` | Signature image and the "Signature Valid / Digitally Signed by" lines commented out |
| `src/features/inventory/components/purchase-sheet.tsx` | Signature images and the signature lookups commented out |
| `src/features/dispatch/components/gate-pass-sheet.tsx` | Signature images commented out |
| `src/routes/admin.users.tsx` | Signature column and the "Digital Signature" row of the View popup commented out |
| `src/features/users/api.ts` | Clears signatures saved in the browser; no longer saves any; users never carry a signature |
| `src/features/products/product-dialog.tsx` | Add Rate and Add Charge use the shared `AddButton` |
| `src/features/bank-accounts/quick-add-bank-account-dialog.tsx` (new) | Quick Add form for one bank account, built on the shared bank-account fields and checks |
| `src/features/inventory/components/purchase-payment-dialog.tsx` | Quick Add on Sender Account (company form), Receiver Account (Edit Supplier modal) and Payment Mode (Payment Modes master form); new account / mode selected after save; fields chosen by the mode's field set, so a custom mode shows the Cash fields; the modal is hidden, not closed, while a Quick Add is open, so typed values are kept |
| `src/features/payment-terms/payment-term-dialog.tsx` | Optional `onSaved` callback with the saved code and name |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Shows the mode by its saved name; bank cards shown by the mode's field set |
| `src/features/inventory/purchase-types.ts` | Payment mode is any string; options carry `fields`; posting carries `paymentModeName` and `paymentFields` |

## Why

These were requested changes for the Order Booking routing modals, the document previews and the Purchase Payment modal.

- **Washing and the type masters.** The Washing stage had no modal at all. Cutting / Edge / Grinding types were free text, so the same value was typed several ways.
- **Signatures.** They had been removed from the app, but some previews and the emailed PDF still showed them, because the backend kept sending them and the browser kept a saved copy.
- **Purchase Payment.** Users had to leave the modal to add a missing bank account. Payment modes added in the master never appeared in the dropdown.

The real work in item 8 was on the backend and the database. The purchase payment record only allowed the four built-in modes, so a migration and new validation rules were needed before a Quick Add on Payment Mode could be of any use.

## How

**Master value stored by name**: Cutting / Edge / Grinding choices are saved as the option's name, the same way Tempering options already were. Old free-text values therefore keep loading, and are shown as an extra option when they match nothing in the master.

**Quick Add keeps the parent form**: The parent modal is hidden, not closed, while a Quick Add dialog is open. Its state and typed values survive, and the new record is selected when the Quick Add saves.

**Field set per payment mode**: The payment options now return every active master mode with a `fields` value (`BANK_TRANSFER`, `UPI`, `CASH` or `CHEQUE`). The master's code `BANK` still maps to Bank Transfer as before. Any other code gets `CASH`. The modal and the server both pick the fields and rules from that value.

**Mode name saved with the payment**: The master's name at the time of payment is stored in `payment_mode_name`, so renaming a mode later does not change old records. Older payments have no name and fall back to the built-in label.

**Signatures**: Removed by not selecting or serving them, and by commenting out the code that renders them. The signature boxes and lines stay, blank, so documents can still be signed by hand.

## Decisions made

**Removed code is commented out, not deleted.** This follows the project's rule for removals ("None" option, old Tempering field, signature rendering, signature endpoints).

**The `signature_url` column and the stored signature files were kept.** Dropping them would be hard to undo, and nothing reads them now.

**The Receiver Account Quick Add opens the existing Edit Supplier modal.** It does not open a separate bank-account form, so supplier bank details are managed in one place. It needs the supplier edit right.

**Custom payment modes are only enabled for purchase payments.** PI and Final Invoice approvals use a separate payment table, which still allows only the four modes. Widening that table was not asked for.

**Add Rate / Add Charge reuse the existing `AddButton`.** That component already serves every other "+ Add" button, so I did not create a new one. As a result, both buttons changed from light cyan to the standard primary style.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- **The company bank account Quick Add can fail for some users.** The company bank account endpoint already accepted only these roles: Super Admin, Admin, Manager, Customer Care and Sales Manager. Users outside those roles who have the Master Data add right see the Sender Account Quick Add, but get an error when they save.
- **Two pieces of code are now unused.** They are the `POST /master/bank-accounts/supplier/:supplierCode` endpoint and the supplier mode of `QuickAddBankAccountDialog`. Both were built before the Receiver Account was switched to the Edit Supplier modal. They are kept and could be removed in a later clean-up.
- **Leftover lint errors.** They are in lines that existed before this change (`any` types, `set-state-in-effect`, `only-export-components`).

## Next steps

- Decide whether PI and Final Invoice approvals should also accept custom payment modes.
- Optional cleanup:
  - remove the unused supplier bank-account endpoint;
  - clear or drop the `signature_url` column;
  - delete the old files under `signatures/` in storage.

## Test plan

**Automated tests:**
- [x] Frontend type check (`npx tsc -p tsconfig.app.json --noEmit`): passes
- [x] ESLint on the changed files: no new errors; the remaining ones were already there

**Manual checks (local):**
- [x] Order Booking > Add Product > Work-Centre Routing: Washing opens its modal; Process Required and Remark save and load back
- [x] Cutting Type, Edge Type, Grinding Type dropdowns list the seeded options, search, Quick Add a new option and select it; an old free-text value still shows
- [x] Tempering Detail: Load Direction, Wave Limits and Bend Limits Quick Add works
- [x] Routing dropdowns no longer show "None"; empty shows "Select"
- [x] No signature in the Proforma Invoice, Final Invoice, E-Invoice, Purchase Order and Gate Pass previews, the PI PDF, or the Users page
- [x] Add / Edit Product: Add Rate and Add Charge buttons work and match the other "+ Add" buttons
- [x] Purchase > Payment: Sender Account Quick Add adds a company account and selects it; Receiver Account opens Edit Supplier and the new account is selected after save
- [x] Purchase > Payment: Payment Mode Quick Add adds a mode (e.g. Card) and selects it; it shows the Cash fields and saves; the View shows the mode by name; the four built-in modes behave as before
