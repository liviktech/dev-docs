# 2026-11-03 - razorpay-upi-card-payments (backend)

## Summary

Only Cash on Delivery worked before, so shoppers could not pay with UPI or cards. This PR adds a Razorpay payment provider to Medusa. It creates a Razorpay order for each payment session, checks the result with Razorpay itself before the order is placed, captures the money, refunds it when an order is cancelled, and listens to Razorpay webhooks. Cash on Delivery keeps working as the built in manual provider next to it.

For developing without Razorpay keys there is also a mock provider and a small dev only route. Set `PAYMENT_MODE=mock` to use them. The server refuses to start with mock mode in production.

Checked live: the user paid with a Razorpay test account and a test card through the storefront popup and the order went through. Webhooks, UPI test IDs, refunds and a real failed payment were not checked, see "Test plan" below.

**Netbanking, wallets, EMI and Pay Later are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/razorpay/service.ts` (new) | The payment provider (`AbstractPaymentProvider`, id `razorpay`, so Medusa calls it `pp_razorpay_razorpay`). Creates the order, authorizes, captures, refunds, cancels, reads the status and handles webhooks. |
| `src/modules/razorpay/client.ts` (new) | A small wrapper over the Razorpay REST API (create order, list payments for an order, capture, refund, webhook signature check) plus rupee to paise helpers. |
| `src/modules/razorpay/index.ts` (new) | Registers the provider with Medusa. |
| `src/modules/razorpay-mock/service.ts` (new) | A fake provider with the same id and the same shape of data, used when `PAYMENT_MODE=mock`. Its sessions carry `mock: true`. |
| `src/modules/razorpay-mock/state.ts` (new) | Remembers, in memory, whether the tester chose success or failure for each fake order. |
| `src/modules/razorpay-mock/index.ts` (new) | Registers the mock provider. |
| `src/api/store/mock-payment/route.ts` (new) | Dev only `POST /store/mock-payment`. Lets the storefront say a fake payment succeeded or failed. Returns not found unless mock mode is on and `NODE_ENV` is not production. |
| `src/api/middlewares.ts` | Adds body validation for the mock route. No other change. |
| `medusa-config.ts` | Registers the payment module with the Razorpay provider (or the mock one), reading the keys from env. The translation module entry is kept as it was, only moved into the same `modules` list. Throws at startup if mock mode is used in production. |
| `.env.template` | Documents `PAYMENT_MODE`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` and `RAZORPAY_WEBHOOK_SECRET`. |
| `src/scripts/enable-online-payments.ts` (new) | Adds `pp_razorpay_razorpay` to the India region's payment providers. Safe to run twice. |

## Why

Most Indian shoppers expect UPI and cards, and the store only offered cash on delivery. The plan for this work is in `tasks/upi/plan.md`. Registering the provider was the easy part. The real work was making sure an order is only placed when Razorpay says the money arrived, and keeping the amount in paise and the session id tied to the Razorpay order so webhooks can find the right cart.

## How

**Order per session**: `initiatePayment` creates a Razorpay order for the cart total (rupees are turned into paise) and stores `razorpay_order_id`, the public `key_id`, the amount and the currency in the session data. The storefront popup needs exactly these. The Medusa session id goes into the Razorpay order notes so a webhook can be matched back to the session.

**Changed cart total**: a Razorpay order cannot be edited. If the amount in `updatePayment` is the same, nothing happens. If it changed, a new order is created.

**Result is checked on the server**: `authorizePayment` asks Razorpay for the payments on that order and does not trust anything the browser sends. A captured payment means captured, an authorized one means authorized, all failed means error, anything else means still pending.

**Capture**: if the Razorpay account does not auto capture, a payment stays only authorized and Razorpay refunds it after a few days. We sell on payment, so `authorizePayment` captures it straight away. `capturePayment` does the same if it is called later and the payment is still only authorized.

**Refund and cancel**: `refundPayment` refunds the given amount against the stored payment id. `cancelPayment` refunds in full if the payment was captured, and does nothing if nothing was paid. Unpaid Razorpay orders expire on their own, so `deletePayment` does nothing.

**Webhooks**: Razorpay posts to `/hooks/payment/razorpay_razorpay`. The signature is checked with `RAZORPAY_WEBHOOK_SECRET`. Valid `payment.authorized`, `payment.captured` and `payment.failed` events become the matching Medusa actions. Anything else is ignored as not supported.

**Mock mode**: the mock provider uses the same id, so the region setup, the storefront and the stored data look the same. Instead of calling Razorpay it makes fake `order_mock_...` orders and takes its result from the in memory map filled by the mock route. That map is lost on restart, which is fine for a mock.

**Setup**: the provider id has to be added to the region once, with `npx medusa exec ./src/scripts/enable-online-payments.ts`. Without this only Cash on Delivery shows at checkout. In the Razorpay dashboard the webhook should point to `/hooks/payment/razorpay_razorpay` and use the same secret as `RAZORPAY_WEBHOOK_SECRET`.

## Decisions made

**Real provider is the default, mock is opt in.** `medusa-config.ts` uses mock only when `PAYMENT_MODE=mock`. Anything else, including an empty value, means Razorpay. With real mode the keys are required, so a missing key fails at startup instead of at the first payment.

**Mock mode is blocked in production in two places.** The config throws on startup, and the mock route returns not found. One guard would have been enough, but a fake payment path is the kind of thing worth locking twice.

**Always capture.** We did not keep authorize only payments for a manual capture later. The shop sells on payment, and uncaptured payments are refunded by Razorpay on their own after some days, so waiting only risks lost orders.

**Only UPI and card at the storefront.** Netbanking, wallets, EMI and Pay Later are not offered. The provider itself does not block them, the storefront popup is what limits the methods, so they can be turned on later without backend changes.

## Notes

- The keys live in `.env` and were never printed or copied. `.env.template` has empty values only.
- After switching `PAYMENT_MODE` the cart's old payment session still has the old data (for example `mock: true`). Choosing the payment method again at checkout creates a fresh session.
- The India region must have the provider added, either with the script or in Medusa Admin under the region settings.
- The test account and live account use different keys and different webhook secrets. Moving to live means changing the three values in the environment, nothing else.

## Next steps

- Create the webhook in the Razorpay dashboard and test it end to end (needs a public URL, for example a tunnel in development).
- Test a refund from Medusa Admin on a captured test payment.
- Test a failed payment and a cancelled popup with the Razorpay test failure cards and UPI IDs.
- Decide whether netbanking and wallets should be added later.

## Test plan

**Automated tests (type check only, no test suite added):**
- [ ] Unit or integration tests: none added for the provider, the client or the mock route.
- [ ] Type check and lint on the backend: not re-run when this document was written, so no result is claimed.

**Manual checks (local):**
- [x] Mock mode: the storefront showed the "Simulate success" and "Simulate failure" buttons from the mock provider.
- [x] Real mode with a Razorpay test account and a test card: the popup opened, the payment went through and the order was placed (reported by the user after trying it).
- [x] Region setup: after running `enable-online-payments.ts` the UPI and Card choices showed next to Cash on Delivery.
- [ ] UPI test IDs through the popup - not checked.
- [ ] Webhook delivery and signature check - not checked, needs a public URL.
- [ ] Refund and cancel of a captured payment - not checked.
- [ ] A failed card payment and a closed popup with the real provider - not checked.
- [ ] Startup refusal with `PAYMENT_MODE=mock` and `NODE_ENV=production` - not run.
