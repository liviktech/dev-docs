# 2026-11-02 - party-gstin-supplier-bank-add-to-stock (backend)

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
| `src/database/migrations/112_supplier_bank_details.sql` (new) | Four optional columns on `master.supplier`: bank name, branch name, IFSC code, account number |
| `src/database/migrations/113_inventory_purchase_stock_posting.sql` (new) | New table with one row per purchase: payment mode, sender company and receiver supplier references, a copy of both account details, and the amount breakdown |
| `src/database/migrations/114_party_pan.sql` (new) | Optional `pan` column on `master.party` |
| `src/modules/master/stock-option/stock-option.config.ts`, `stock-option.schema.ts` | The four bank fields added to the supplier fields; field definitions can now say "upper-case" and "must match this pattern" (used for IFSC and account number) |
| `src/modules/master/party/party.schema.ts`, `party.types.ts`, `party.repository.ts` | PAN saved and returned; blank means none; upper-cased and checked as 5 letters, 4 digits, 1 letter |
| `src/modules/gstin-lookup/gstin-lookup.client.ts` (new) | Calls gstinapi.in, maps the reply, keeps a 3-day in-memory cache, shares one call between simultaneous requests, 10 second timeout, maps provider errors to our own codes |
| `src/modules/gstin-lookup/gstin-lookup.service.ts`, `gstin-lookup.controller.ts`, `gstin-lookup.routes.ts` (new) | Wires the key from the environment; signed-in users only; `GET /gstin/lookup/:gstin` |
| `src/shared/utils/gstin.util.ts` (new) | GSTIN format and check-character test |
| `src/config/env.ts`, `src/api/route-mapper.ts` | `GSTINAPI_KEY` (optional); the new route is mounted at `/gstin` |
| `.env.example` | `GSTINAPI_KEY` with a full description of the feature |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts`, `types.ts`, `repository.ts`, `service.ts`, `controller.ts`, `routes.ts` | New `GET /:id/stock-posting-options`; `POST /:id/add-to-stock` now takes the payment mode and accounts, checks them on the server, and saves everything in the same transaction as the stock; the purchase read returns the saved details |
| `tests/gstin-lookup.test.ts` (new) | 9 tests for the GSTIN check and the lookup client |

## Why

- **Pagination:** a bigger default means fewer clicks on long lists. This was a small change because the shared table and the URL-state helper hold most of the setting.
- **Party modal and GSTIN lookup:** the three tabs hid information, and typing a party's name, address and PAN by hand is slow and error-prone. The wireframe asked for one view.
- **Supplier bank details and Add to Stock:** Add to Stock used to move stock with a plain confirmation and keep no record of how or to whom the supplier was paid. This records that, using the bank details of the company and the supplier. The real work was the transaction and the server checks, not the screen.
- **The GSTIN lookup guide** (`docs/GSTIN_LOOKUP_FEATURE_GUIDE.md`) drove the lookup design. It was adapted, not copied: the app has no state dropdown, and GSTIN lives on the address, not the party.

## How

**Page size**: only the starting value changed. Server-side lists still send the chosen size in the existing `limit` / `pageSize` request fields, and filters, sort and search code were not touched.

**GSTIN lookup**: the key stays on the server. The lookup runs only when the user clicks Fetch Details, never while typing. The server re-checks the GSTIN, then calls the provider. A provider 401, 402 or 403 is returned to the browser as 502, so an invalid key cannot log the user out.

**Autofill rules**: Fetch Details fills Party name (trade name, else legal name), PAN, Address Line 1, City, State and Pincode on the default address. It fills only blank fields, or fields that an earlier fetch filled and the user has not edited since. Fetching a different GSTIN therefore replaces the old values, but never anything the user typed. A kept value is named in the message. A registration that is not Active shows a warning.

**GSTIN field**: the top box is now the only GSTIN field. It opens with the saved default address's GSTIN, is checked on Save, and is written onto the default address. The data model was not changed: GSTIN is still stored on the address.

**Supplier bank details**: suppliers run through one shared code path, so the four fields were added to a field list, not to separate code for each layer.

**Add to Stock**:

- The amounts come from the purchase's stored values: each line's total, the discount worked out as gross minus net, the invoice charges and the invoice total. Nothing is recalculated differently.

- The sender must be the company's own account and the receiver must belong to the purchase's supplier. This is checked on the server.

- Payment Mode is required. Bank Transfer needs both accounts; Cheque shows them but does not require them; Cash and UPI hide them, and the server ignores any account sent with those modes.

- The stock lots, the receipt movements, the status change and the new record are saved in one transaction. The purchase row is locked first, so a second click or retry is refused. The saved record keeps a copy of the account details, so it stays correct if the master data changes later.

## Decisions made

**PAN needs its own column.** The Party record had nowhere to keep PAN, so a small optional column was added (migration 114). Other values the lookup returns (registration status, taxpayer type, registration date, constitution) have no Party field and were not stored, to avoid more database changes.

**Server cache for lookups.** Results are kept in server memory for 3 days to save provider credits. It resets on restart. A shared cache was not built.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- Three migrations must run in this order: `112`, `113`, `114` (`npm run migrate`). Saving a party fails until `114` is applied.
- `GSTINAPI_KEY` is set in the backend `.env`; the same variable must be set on any deployed environment, and the API restarted. If it is empty, Fetch Details answers "not configured" and the rest of the modal works.
- The page-size change touches many files but only the starting value.

## Next steps

- Run one lookup with the provider's sandbox GSTIN and check the real response, then decide whether status, taxpayer type or registration date deserve their own Party fields.
- Consider a per-user limit on GSTIN lookups. Only the app's global rate limit applies today.
- Rename the "Registered Addresses & GSTIN" wording if it still appears anywhere; the address-level GSTIN field is gone.

## Test plan

**Automated tests:**
- [x] `tests/purchase-cost.test.ts`, `tests/inventory-purchase-create.test.ts`: 21 cases pass (existing purchase tests, run after the Add to Stock change).
- [x] A throwaway script showed the party schema upper-cases `aaacc1206d`, turns a blank into none, rejects `BAD`, and accepts an update with only a PAN.
- [ ] Add to Stock for a supplier with no bank details, and a purchase that fails halfway - not confirmed in the browser; the server checks and the single transaction cover them.
- [ ] Authentication on `GET /gstin/lookup/:gstin` - uses the existing sign-in check; no separate test was written.

**Manual checks (local):**
- [x] Fetch Details with a valid GSTIN fills Party name, PAN, address, city, state and pincode into the right fields; the GSTIN box saves and reloads on edit.
- [x] Supplier Bank Details: saving, editing and the view panel.
