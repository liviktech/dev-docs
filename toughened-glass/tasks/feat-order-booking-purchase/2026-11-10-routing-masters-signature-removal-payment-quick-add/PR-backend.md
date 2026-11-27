# 2026-11-10 - routing-masters-signature-removal-payment-quick-add (backend)

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
| `src/database/migrations/127_order_booking_item_washing_detail.sql` (new) | `sales.order_booking_item_washing_detail` (process, remark), one per item |
| `src/database/seeds/036_washing_detail_rbac.sql` (new) | `washing_detail` tab and its VIEW / ADD / EDIT / DELETE rights for Order Booking |
| `src/database/migrations/128_cutting_edge_grinding_type_masters.sql` (new) | `master.cutting_type`, `master.edge_type`, `master.grinding_type` with a case-insensitive unique name per company; seeds the four options per company; old free-text values left untouched |
| `src/database/migrations/129_purchase_payment_any_master_mode.sql` (new) | Drops the four-mode check on the purchase payment record, widens `payment_mode` to 50 characters, adds `payment_mode_name` |
| `src/modules/order-booking/order-booking.schema.ts`, `types.ts`, `repository.ts`, `service.ts`, `controller.ts` | Washing detail read, saved with the item bundle, deleted and copied with the item, and gated by the `washing_detail` tab rights |
| `src/modules/master/stock-option/stock-option.config.ts`, `stock-option.routes.ts`, `src/modules/master/master.routes.ts` | List / create / update / deactivate APIs at `/master/cutting-types`, `/master/edge-types`, `/master/grinding-types` |
| `src/modules/master/lookup/lookup.repository.ts`, `lookup.datasets.ts` | The three lists in `GET /lookup`, refreshed on writes |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts`, `src/modules/final-invoice/final-invoice.repository.ts`, `src/modules/dispatch/dispatch.repository.ts` | Signature URLs no longer selected |
| `src/modules/notifications/attachments/pi-sheet/pi-sheet.loader.ts` | Emailed PI / Final Invoice PDF no longer loads signature images |
| `src/modules/master/user/user.routes.ts`, `controller.ts`, `service.ts`, `repository.ts`, `schema.ts`, `types.ts` | Signature upload / download endpoints and the signature field on users commented out; create / update no longer write `signature_url` |
| `src/modules/master/bank-account/bank-account.routes.ts`, `repository.ts`, `schema.ts` | `POST /master/bank-accounts/supplier/:supplierCode` (add one account to an active supplier) - not used by the screen any more, see "Notes" |
| `src/modules/payment-detail/payment-detail.service.ts` | `MASTER_CODE_TO_MODE` exported; no behaviour change |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts`, `types.ts`, `repository.ts`, `service.ts` | Payment mode accepts any active master mode; a non-built-in mode must fill the Cash fields; the master name is saved with the payment |

## Why

These were requested changes for the Order Booking routing modals, the document previews and the Purchase Payment modal.

- **Washing and the type masters.** The Washing stage had no modal at all. Cutting / Edge / Grinding types were free text, so the same value was typed several ways.
- **Signatures.** They had been removed from the app, but some previews and the emailed PDF still showed them, because the backend kept sending them and the browser kept a saved copy.
- **Purchase Payment.** Users had to leave the modal to add a missing bank account. Payment modes added in the master never appeared in the dropdown.

The real work in item 8 was on the backend and the database. The purchase payment record only allowed the four built-in modes, so a migration and new validation rules were needed before a Quick Add on Payment Mode could be of any use.

## How

**Master value stored by name**: Cutting / Edge / Grinding choices are saved as the option's name, the same way Tempering options already were. Old free-text values therefore keep loading, and are shown as an extra option when they match nothing in the master.

**Field set per payment mode**: The payment options now return every active master mode with a `fields` value (`BANK_TRANSFER`, `UPI`, `CASH` or `CHEQUE`). The master's code `BANK` still maps to Bank Transfer as before. Any other code gets `CASH`. The modal and the server both pick the fields and rules from that value.

## Decisions made

**Removed code is commented out, not deleted.** This follows the project's rule for removals ("None" option, old Tempering field, signature rendering, signature endpoints).

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- **The company bank account Quick Add can fail for some users.** The company bank account endpoint already accepted only these roles: Super Admin, Admin, Manager, Customer Care and Sales Manager. Users outside those roles who have the Master Data add right see the Sender Account Quick Add, but get an error when they save.
- **Migration 129 was dry-run before use.** It ran inside a transaction that was rolled back, with no errors and the table left unchanged.
- **Leftover lint errors.** They are in lines that existed before this change (`any` types, `set-state-in-effect`, `only-export-components`).

## Next steps

- Run `npm run migrate` and then `npm run seed` in `rubine-toughened-glass-backend` on every environment. They apply migrations 127, 128 and 129 and seed 036. Saving a product with a Washing detail, the type dropdowns and purchase payments all depend on them.
- Decide whether PI and Final Invoice approvals should also accept custom payment modes.
- Optional cleanup:
  - remove the unused supplier bank-account endpoint;
  - clear or drop the `signature_url` column;
  - delete the old files under `signatures/` in storage.

## Test plan

**Automated tests:**
- [x] Backend type check (`npx tsc --noEmit`): passes
- [x] ESLint on the changed files: no new errors; the remaining ones were already there
- [x] Migrations 127, 128 and 129 and seed 036: dry-run in a rolled-back transaction with no errors

**Manual checks (local):**
- [x] Backend API endpoints and database operations verified.
