# 2026-11-06 - update-labels-and-bank-upi (frontend)

## Summary

Edit forms said "Save" (or "Edit") on their submit button, which read oddly when you were changing an existing record. They now say "Update" when editing and still say "Save" when adding. The Company Profile page also had hand-styled buttons that did not match each other (the top and bottom Cancel and Update looked different), so they now use shared buttons. Bank accounts got a UPI checkbox, and the UPI payment mode in Party Approval and Add to Stock now also records the sender and receiver bank account, with their bank detail cards.

The "Update" wording was checked by type check and a production build only, not by clicking through every modal. The UPI account dropdowns were not clicked through in the browser either.

**Non-edit "Save" buttons (attendance, downtime, "Save draft", the New PI flow, booking and invoice save flows) are not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/components/ui/form-dialog.tsx` | In edit mode the submit label shows "Update" when the label is "Save", "Edit" or not set. Custom labels like "Request Edit" are left alone. This one change covers most master data edit dialogs. |
| `src/components/ui/submit-button.tsx` (new) | Shared primary action button (Save, Update, Create). Pairs with the existing `CancelButton`. |
| `src/components/ui/secondary-button.tsx` (new) | Shared outlined button for secondary actions. Used for Retry on the company page. |
| `src/components/ui/index.ts` | Exports the two new buttons. |
| `src/features/company/company-page.tsx` | Edit Profile, Cancel and Update (top and bottom) now use `SubmitButton` and `CancelButton`, so both rows look the same. "Save Changes" became "Update" (and "Updating…" while saving). Retry uses `SecondaryButton`. |
| `src/features/company/company-bank-accounts.tsx` | Add Bank Account uses `SubmitButton` and is always visible. Each account card shows a "⋮" menu (Edit, Set as Primary, Deactivate) using the same `RowActionsMenu` the tables use, only after Edit Profile is clicked. UPI shows as "Enabled" on the card. The add and edit dialog has a UPI checkbox next to Set as Primary. |
| `src/features/bank-accounts/types.ts` | Bank account draft, payload and saved types carry an optional `upiEnabled` flag. |
| `src/features/bank-accounts/bank-accounts-section.tsx` | The shared bank account form has a UPI checkbox at the end (used by the Party and Supplier modals). A `showUpi` option lets the Company dialog place it elsewhere. |
| `src/features/payment-details/payment-details-fields.tsx` | Party Approval: when the mode is UPI, Sender Account and Receiver Account dropdowns with bank cards show below Transaction ID and Date. Required only if that side has accounts. The read-only recorded view shows the account cards for UPI too. |
| `src/features/inventory/components/add-to-stock-dialog.tsx` | Same idea for Add to Stock: UPI shows company sender and supplier receiver account dropdowns and cards below the transaction fields. |
| `src/features/inventory/components/purchase-view-dialog.tsx` | The read-only stock posting shows the account cards for UPI payments too. |
| `src/features/inventory/components/purchase-dialog.tsx` | Submit says "Update" when editing a purchase, "Save Purchase" otherwise. |
| `src/features/inventory/components/add-customer-material-dialog.tsx` | "Update" when editing, "Save" when adding. |
| `src/features/inventory/components/lot-group-edit-dialog.tsx` | Admin button says "Update" (non-admin still "Request Edit"). |
| `src/features/order-booking/booking-header-form.tsx` | "Update" for an existing booking, "Save" for a new one (also in the saving state). |
| `src/features/order-booking/add-product-modal.tsx` | "Update" when editing a product line. |
| `src/features/order-booking/work-center-detail-modal.tsx` | The five work center detail dialogs (Cutting, Grinding, Fabrication, Store, Tempering) say "Update". |
| `src/features/order-booking/work-order-edit-dialog.tsx` | "Update" in place of "Save". |
| `src/features/invoices/create-proforma-invoice.tsx` | "Update" when editing an existing PI (has a PI code), "Save" for a new one. |
| `src/features/products/product-dialog.tsx` | "Update" when a product is being edited, "Save" otherwise. |
| `src/features/enquiries/enquiry-form-modal.tsx` | "Update" when editing, "Save" when adding. |
| `src/features/parties/party-dialog.tsx` | "Update Party Account" when editing. It now checks the party id, because the `party` prop is empty when editing from the party page. |
| `src/features/notifications/event-edit-dialog.tsx` | "Update" (edit only dialog). |
| `src/features/pwa/enquiry/mobile-enquiry-form.tsx` | "Save changes" became "Update". |
| `src/features/pwa/inventory/mobile-lot-form.tsx` | "Update" when editing a lot, "Save" when adding. |
| `src/features/pwa/work-orders/mobile-work-order-detail.tsx` | "Update" in place of "Save". |
| `src/features/pwa/inventory/locations-tab.tsx` | "Update" in place of "Save" when editing. |

## Why

Users asked for edit modals to say "Update", and for the Company Profile buttons to be the shared ones so the page is consistent. The UPI work came from a request to mark which bank accounts accept UPI and to show the bank details when UPI is picked as the payment mode. Most of the label work was easy because `FormDialog` already knows when it is editing. The real work was the Company Profile button cleanup and the UPI account sections.

## How

**Update label**: `FormDialog` decides the text. If it is in edit mode and the label is "Save", "Edit" or empty, it shows "Update". Dialogs that build their own button got the same rule by hand.

**Shared buttons**: `SubmitButton` and `CancelButton` carry the same height, padding and font, so the header and the footer on the Company Profile page match. Bank account actions use the same `EditActionButton`/`RowActionsMenu` family the tables use.

**UPI checkbox**: it is only a yes or no flag on a bank account. No UPI ID or name is stored on the account.

**UPI payment accounts**: in Party Approval and Add to Stock, the new dropdowns sit under the existing UPI fields, which are unchanged. The chosen accounts are sent as the same sender and receiver account ids a bank transfer uses.

## Decisions made

**UPI accounts list all accounts, not only UPI ticked ones.** An earlier version filtered the payment dropdowns to accounts with the UPI box ticked, but it was reverted. The current dropdowns list every account, so they do not depend on the checkbox. The checkbox is saved and shown but does not filter anything yet.

**Account dropdowns are required only when accounts exist.** If a party, supplier or the company has no bank account, UPI approval is not blocked.

**Only edit modals were changed.** Attendance, downtime, "Save draft", "Save Purchase" for a new purchase, the New PI flow and the booking and invoice save flows keep "Save".

**Work center detail dialogs are treated as edit only.** They only change an existing product line, so they always say "Update".

## Bugs found while testing (already there before, not fixed here)

**1. Lint errors in the files touched.** `eslint` reports about 31 errors in the touched folders, for example the React rule against calling setState inside an effect in `payment-details-fields.tsx`. That effect pattern was there before this change. It affects code quality checks only, not behaviour.

## Notes

- Type check (`tsc --noEmit`) and `vite build` pass.
- This repo had other uncommitted changes from other work (dispatch, offcut, work order stock section, inventory route). They are not part of this PR and are not listed above.
- An edit I made while moving the bank card actions collapsed some lines and broke `vite build`. It was fixed and the build passes now.

## Next steps

- Decide whether the payment account dropdowns should only offer UPI ticked accounts now that the checkbox exists.
- Clean up the existing lint errors in the payment details form.

## Test plan

**Automated tests (no test files added):**
- [x] `tsc --noEmit` in the frontend: passes.
- [x] `vite build` in the frontend: builds.
- [ ] `eslint` on the touched folders: not clean (about 31 existing errors, see above).

**Manual checks (local):**
- [ ] Open an edit modal in each area (master data, purchase, order booking, PI) and check the button says "Update" - not done by hand. Only the type check and build cover this.
- [ ] Company Profile: click Edit Profile, check top and bottom Cancel and Update match, and the "⋮" menu on a bank account shows Edit, Set as Primary and Deactivate - not done by hand.
- [ ] Add a bank account with UPI ticked and see "UPI: Enabled" on the card - not done by hand.
- [ ] Party Approval and Add to Stock with UPI: pick sender and receiver accounts and submit - not done by hand.
