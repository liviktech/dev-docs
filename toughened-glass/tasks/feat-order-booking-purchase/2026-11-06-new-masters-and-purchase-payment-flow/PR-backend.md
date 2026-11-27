# 2026-11-06 - new-masters-and-purchase-payment-flow (backend)

## Summary

Two main pieces of work, plus a set of smaller fixes, all in the same branch pair.

1. **Four new Master Data screens: Layer, Port, Destination and Logo.** The Add / Edit Product form in Order Booking had Layer, Port, Destination and Logo as a free number box, two free text boxes and a fixed list of four logos. They are now searchable dropdowns fed by these masters, with a **+ Add** (Quick Add) at the end of each. The new masters start empty.
2. **Add to Stock and Payment are now two separate purchase actions.** Before, Add to Stock asked for the payment mode and bank accounts and saved them together with the stock transfer. Now **Add to Stock** shows only General Details and Product Details, and a new **Payment** action in the kebab menu records the bank and payment details. Either can be done first, and Payment can be done days after the stock transfer.
3. **Smaller fixes** across Purchase, Order Booking, Work Order, Production Plan, Proforma / Final Invoice, FIR and Master Data screens (listed in the tables below).

The author checked all of these changes in the browser. Both projects compile with no type errors. No automated tests were added (see "Test plan").

**Seed data for the four new masters is not included on purpose** - see "Decisions made" below. The records are added by hand later. This was a choice, not a miss.

## What changed

**New masters (Layer, Port, Destination, Logo)**

| File | Change |
|---|---|
| `src/database/migrations/125_layer_port_destination_logo_masters.sql` (new) | Four tables in `master` (`layer`, `port`, `destination`, `logo`): company, code, name, is_active, timestamps, and a case-insensitive unique name per company. No rows inserted |
| `src/modules/master/stock-option/stock-option.config.ts` | Four new list configs; Layer is flagged as whole-number-name |
| `src/modules/master/stock-option/stock-option.schema.ts`, `stock-option.routes.ts` | Layer name must be a whole number from 1 to 999; four new routers built by the same factory (list, get, create, update, deactivate) |
| `src/modules/master/master.routes.ts` | Mounted at `/master/layers`, `/ports`, `/destinations`, `/logos` |
| `src/modules/master/lookup/lookup.repository.ts`, `lookup.datasets.ts` | The four lists added to the central `/lookup` and to its cache refresh, instead of separate lookup endpoints |

**New purchase flow (Add to Stock and Payment)**

| File | Change |
|---|---|
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | `addPurchaseToStock` takes no payment input and records no payment. New `recordPurchasePayment` and `getPaymentOptions`. Edit and delete refused once a payment is recorded |
| `src/modules/inventory-purchase/inventory-purchase.routes.ts`, `controller.ts` | `POST /:id/add-to-stock` has no body; `GET /:id/stock-posting-options` replaced by `GET /:id/payment-options`; new `POST /:id/payment` |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts`, `types.ts` | The payment schema is now `recordPurchasePaymentSchema`; summary gets `paymentRecorded` |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | The list query returns `payment_recorded`; the purchase search is limited to the visible columns |

**Smaller fixes**

| File | Change |
|---|---|
| `src/modules/work-order/work-order.repository.ts`, `work-order.types.ts`, `work-order.service.ts` | Lot numbers (issued and selected) added to each work order item for the View Work Order modal |

## Why

- **New masters:** Layer, Port, Destination and Logo were typed in by hand or came from a fixed list, so spellings differed and the logo list needed a code change to grow. Masters give one clean list, and Quick Add means a user is not sent away from a half-filled product. The shared master list code already existed, so the real work was wiring them into the product form and keeping the stored values unchanged.
- **Purchase flow:** payment was tied to the stock transfer, so a purchase could not be received first and paid later, which is how it works in practice. The real work was moving the payment record out of the stock transaction and making the two actions independent.
- **Smaller fixes:** each was a user-reported problem found while using the screens.

## How

**New masters**: they reuse the shared list code (one repository, service and router builder), so they behave exactly like Glass Colors and Glass Finishes, including duplicate-name errors and soft deactivation. The product form reads them through the central `/lookup`, so only the lookup and master write paths needed the new names.

**Payment availability**: Payment works for a Pending Stock purchase and for one already added to stock. It is disabled only once the payment is saved, and the server enforces this with a 409.

**Edit and Delete after payment**: once a payment is recorded, Edit and Delete are disabled in the menu and refused by the server, because the saved amounts would no longer match what was paid. Add to Stock is still allowed.

## Decisions made

**Payment History became Supplier Bill in the Purchase View.** The row it listed is the invoice record saved with the purchase (its amount and bill file), not a payment. It was making every new purchase look fully paid. A real payment now exists only after the Payment action is saved.

## Bugs found while testing (already there before, not fixed here)

**1. `paidAmount` still counts the invoice record.** The backend adds up the invoice record as paid money, so `paidAmount` and `remainingAmount` in the API are not true paid figures. This is why the old view showed a new purchase as paid. The view no longer shows them, and no list column uses them (those columns are already commented out), but the API fields are unchanged. It affects any future screen that reads them.

## Notes

- The four tables were created by running the migration on the local database.
- Nothing is seeded, so a fresh database has empty Layer, Port, Destination and Logo lists.
- The old document `docs/PR-party-gstin-supplier-bank-add-to-stock.md` still describes the earlier Add to Stock that took payment details. It was left as history.

## Next steps

- Add the Layer, Port, Destination and Logo records in each environment after deploy.
- Decide whether to fix the `paidAmount` / `remainingAmount` meaning in the API (bug 1) before any screen uses it.
- Clean up the lint errors listed in bug 2 when those files are next touched.
- Run `npm run migrate` in the backend on every other environment (staging, production). It applies only `125_layer_port_destination_logo_masters.sql` and is safe to re-run.

## Test plan

**Automated tests:**
- [x] Backend `tsc --noEmit`: no errors
- [ ] ESLint on the touched files: the errors listed in bug 2 remain, and no new kind of error was added

**Manual checks (local):**
- [x] A purchase with a recorded payment cannot be edited or deleted, and Add to Stock still works for it
- [x] The smaller fixes listed above
