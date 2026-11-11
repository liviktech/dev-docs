# 2026-11-06 - credit-limit-tracker-supplier (backend)

## Summary

Every customer had to pay at checkout, there was no way to let a trusted customer buy now and pay later, and stock could only be changed by the admin by hand. This PR adds three things to the backend: a credit limit per customer (with a "Pay on credit" payment method and a way to pay the balance online), a separate supplier login, and supply requests that the admin approves to add stock. The delivery tracker needs no backend code, it only reads the fulfillment times Medusa already saves.

Customer login and supplier login are kept apart. The same email can sign up once as a customer and once as a supplier, each with its own password, and a supplier token cannot open customer routes (and the other way round).

Not run for this document: the type check, lint and any test run. Nothing here was re-checked while writing it, see "Test plan" below.

**Suppliers adding products to the catalog is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `medusa-config.ts` | Registers the `credit` and `supplier` modules and the `credit` payment provider. Replaces the default auth module entry with one that has two logins: the normal `emailpass` and a new `supplier-emailpass`. Adds `authMethodsPerActor` (user and customer use `emailpass`, supplier uses `supplier-emailpass`) and `requireActorAssociation` for the supplier login. |
| `src/api/middlewares.ts` | Adds `authenticate` and body validation for the new routes. Supplier routes accept only a supplier token, credit routes accept only a customer token. The credit webhook keeps the raw body so the Razorpay signature can be checked. |
| `src/modules/credit/` (new) | The credit module. Models `customer_credit` (limit and used per customer) and `credit_entry` (history: purchase, repayment, release, adjustments). The service has `getSummary`, `setLimit`, `charge`, `release`, `repay` (admin took money) and `repayOnline`. Includes the migration and snapshot. |
| `src/modules/credit-payment/service.ts` (new) | The "Pay on credit" payment provider (id `credit`, so Medusa calls it `pp_credit_credit`). Authorizing the payment is where the limit is enforced, under a lock per customer. Cancel and refund give the credit back. |
| `src/modules/credit-payment/repayment.ts` (new) | "Pay what I owe". Creates a Razorpay order for the amount owed, and settles it once Razorpay says it was paid. Works in mock mode too. |
| `src/modules/credit-payment/index.ts` (new) | Registers the provider with Medusa. |
| `src/modules/supplier/` (new) | The supplier module. Models `supplier`, `supply_request` and `supply_request_item`, a service, and migrations. `Migration20261106120000` makes `supply_request_item.variant_id` nullable so a request can name a product that is not in the catalog yet. |
| `src/modules/supplier-auth/index.ts` (new) | An auth provider with the same email and password logic as `emailpass`, under its own id `supplier-emailpass`, so supplier logins are a separate list from customer logins. |
| `src/workflows/create-supplier.ts` (new) | Creates the supplier profile and ties it to the login that was just registered, so the token becomes a `supplier` token. Undoes the profile if linking fails. |
| `src/workflows/approve-supply-request.ts` (new) | Takes a lock, checks the request is still pending, adds each quantity to inventory, then marks it approved. If a step fails the earlier steps are undone. |
| `src/workflows/reject-supply-request.ts` (new) | Marks a pending request rejected with a note. Stock is not touched. |
| `src/api/store/credit/route.ts` (new) | `GET /store/credit`, the logged in customer's limit, used and available. |
| `src/api/store/credit/pay/route.ts` (new) | `POST /store/credit/pay`, starts "Pay what I owe". The amount comes from the database, not from the browser. |
| `src/api/store/credit/pay/confirm/route.ts` (new) | `POST /store/credit/pay/confirm`, asks Razorpay what happened to the order and lowers the balance if it was paid. A customer can only confirm their own order. |
| `src/api/webhooks/credit-repayment/route.ts` (new) | Backup for the confirm route. Razorpay posts here if the customer closes the tab right after paying. Checks the signature, then settles the order. |
| `src/api/store/suppliers/route.ts` (new) | `POST /store/suppliers`, second step of supplier sign up (register the login first, then create the profile). Refuses if the login already has a profile. |
| `src/api/store/suppliers/me/route.ts` (new) | `GET /store/suppliers/me`, the logged in supplier's profile. |
| `src/api/store/suppliers/me/requests/route.ts` (new) | `GET` lists the supplier's own requests, `POST` creates one. An item is either a catalog variant or a new product name. |
| `src/api/admin/supply-requests/route.ts` (new) | `GET /admin/supply-requests` with an optional status filter. |
| `src/api/admin/supply-requests/[id]/approve/route.ts` and `reject/route.ts` (new) | Run the approve and reject workflows. |
| `src/api/admin/customers/[id]/credit/route.ts` (new) | `GET` limit, used and history, `POST` sets the limit. |
| `src/api/admin/customers/[id]/credit/repay/route.ts` (new) | `POST` records money the customer paid outside the app. |
| `src/admin/widgets/customer-credit.tsx` (new) | A box on the customer page in Admin: set the limit, see used and available, record a payment, see the history. |
| `src/admin/routes/supply-requests/page.tsx` (new) | A "Supply requests" page in Admin with a status filter and Approve and Reject buttons. |
| `src/scripts/enable-credit-payment.ts` (new) | Adds `pp_credit_credit` to the India region. Safe to run twice. |

## Why

The plan is in `plan.md` in this folder. The shop wanted to let chosen customers buy now and pay later, up to a set amount, without trusting the browser. It also wanted suppliers to tell the shop what stock they can bring, with the admin deciding what actually goes into inventory. The easy part was the data models. The real work was keeping the credit rule on the server (so it cannot be skipped from the checkout page), making repeated calls safe, and keeping the two kinds of logins apart.

## How

**Credit is a payment method**: "Pay on credit" is a small payment provider. When Medusa authorizes the payment at the end of cart completion, the provider calls `charge`, which refuses if the order is over the available credit. Medusa cancels the payment if a later step fails, and cancel or refund calls `release`, so the credit comes back.

**Safe to repeat**: `charge`, `release` and `repayOnline` use a reference (the payment session or the Razorpay order id) and a unique index on `reference` plus `type`. A second call with the same reference changes nothing. Each call also runs under a lock per customer, so two checkouts at the same time cannot both pass the limit.

**Paying the balance online**: `createRepaymentOrder` makes a Razorpay order for exactly what the customer owes. `settleRepayment` asks Razorpay for the order and its payments, captures an authorized payment if the account does not auto capture, and then calls `repayOnline`. The browser confirm route and the webhook both call the same function, so it is counted once whichever arrives first. If the customer paid more than they owed (for example they also paid an order in the meantime), the extra part is refunded.

**Two kinds of logins**: Medusa lets each actor type have its own allowed login methods. Customers keep `emailpass`. Suppliers use `supplier-emailpass`, which is the same code under a different name, so its logins are stored as a separate list. That is why the same email can exist as a customer and as a supplier. Sign up is two steps, like the customer flow: register the login, then `POST /store/suppliers` creates the profile and links it.

**Approving stock**: the approve workflow finds the stock location that already holds each variant, adds the quantity, and only then marks the request approved. The lock key is the request id, so a double click cannot add the stock twice. A second approve on the same request is refused because it is no longer pending. Items with no `variant_id` (new product requests) are skipped, there is no stock to add for them.

**Delivery tracker**: nothing in this repo. The storefront reads `packed_at`, `shipped_at` and `delivered_at` from the order's fulfillments, which the existing admin buttons already fill.

**Setup**: run the migrations (`npx medusa db:migrate`), then `npx medusa exec ./src/scripts/enable-credit-payment.ts` once per environment. Without the script "Pay on credit" does not show at checkout. For the webhook, add `https://<backend>/webhooks/credit-repayment` in the Razorpay dashboard with the events `payment.captured` and `payment.authorized`, and the same secret as `RAZORPAY_WEBHOOK_SECRET`.

## Decisions made

**No separate `vegin` schema.** The plan was to keep the 5 new tables in their own schema. I tried it, but the migration puts the tables in `vegin` and creates their indexes without the schema name, so `db:migrate` fails. All 5 tables are in the normal `public` schema.

**Credit is enforced in the payment provider, not in order subscribers.** The plan used subscribers on order placed and order canceled. Doing it in `authorizePayment` is stricter (the order cannot be placed over the limit) and Medusa already undoes the payment when a later step fails, so the subscribers were not needed.

**Whole order on credit or whole order paid.** Part credit and part online payment is not supported.

**Suppliers can ask for a new product by name, but the backend does not create it.** Such an item is saved with an empty `variant_id`. Approving the request skips it, and the admin creates the product separately. Letting suppliers write to the catalog directly was left out so the catalog stays owned by the admin.

**Supplier sign up is open.** Anyone can register as a supplier. This is safe for stock because every request needs admin approval, but spam requests are possible.

**Orders edited by the admin are not re-checked.** If an admin changes an order total after it was placed on credit, the amount owed does not change.

## Bugs found while testing (already there before, not fixed here)

**1. Deleting a customer in Admin can fail with "Auth identity not found".** Medusa's delete route removes the customer and then looks for the login linked to it. A customer marked as having an account but with no linked login (an old test customer made by a script) fails there and nothing is deleted. It affected one account (`cust-test-1793950548020@example.test`), which I deleted with a one off script that is not kept in the repo. Other leftover test customers like it will fail the same way. Normal sign ups are not affected.

## Notes

- There are 5 new tables: `customer_credit`, `credit_entry`, `supplier`, `supply_request`, `supply_request_item`. No existing Medusa table was changed.
- Inventory tracking was already on for all 100 variants, so no script to turn it on was needed.
- The customer credit limit starts at 0, so nobody gets credit until the admin sets it.
- The repayment mock mode follows `PAYMENT_MODE=mock` like the checkout payment. Its fake orders live in memory and are lost on restart.
- The old test accounts `redesign.test@example.test` and `supplier.test@example.test` and some test requests are still in the local database.

## Next steps

- Decide whether the admin delete route should get a fallback for customers without a login record.
- Create the credit repayment webhook in the Razorpay dashboard and test it with a public URL.
- Decide whether supplier accounts should need admin approval before they can send requests.
- Add integration tests for the credit check and the approve workflow.

## Test plan

**Automated tests (none added, none run):**
- [ ] Integration tests for the credit limit check (`charge` over the limit, repeated `charge`, two at the same time): not written.
- [ ] Integration tests for `approve-supply-request` (approve twice, approve with an untracked variant): not written.
- [ ] Type check (`medusa build`) and lint (`medusa lint`): not run when this document was written, so no result is claimed.

**Manual checks (local):**
- [x] Deleted the orphan test customer with a one off script after confirming it had no linked login. The script printed that the customer was deleted.
- [ ] Credit end to end (set a limit, order on credit, order over the limit, repay in Admin, cancel an order): not re-run for this document.
- [ ] Supplier flow end to end (register, send a request, approve in Admin, stock goes up): not re-run for this document.
- [ ] Razorpay "Pay what I owe" with real test keys and the webhook: not checked for this document.
