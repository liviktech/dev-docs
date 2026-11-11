# Plan: UPI and card payments with Razorpay

## What we are building

Right now the shop only takes Cash on Delivery (COD). We want customers to also pay online with UPI (GPay, PhonePe, Paytm and so on) and with credit or debit cards. COD stays as an option.

We will use Razorpay for this. One Razorpay checkout popup gives the customer UPI, cards, netbanking and wallets, so we only write one integration. The money goes to our Razorpay account and then to our bank account.

Nothing is built yet. This file is the plan.

## How a payment will work

1. The customer reaches the payment step and picks "Pay online (UPI / Card)".
2. The backend asks Razorpay to create an order for the cart total (in paise) and sends the order id to the storefront.
3. The storefront opens the Razorpay popup. The customer pays with UPI or a card.
4. Razorpay tells us the result in two ways: the popup returns the result to the browser, and Razorpay also calls a webhook on our backend.
5. The backend checks the signature to make sure the result is real, marks the payment as paid, and the order is placed.
6. If the customer closes the tab after paying, the webhook still finishes the order, so the money is never taken without an order.

The webhook is the source of truth. The browser result is only for a quick response to the customer.

## What we have today

Found by reading the code, nothing changed:

- Backend: `medusa-config.ts` has no payment provider entry. Medusa's built in manual provider (`pp_system_default`) is what works as COD.
- Frontend: the checkout still has the Stripe starter code. The files are `payment-wrapper/index.tsx`, `payment-wrapper/stripe-wrapper.tsx`, `payment-button/index.tsx`, `payment-container/index.tsx`, `payment/index.tsx`, `order/components/payment-details/index.tsx`, `lib/constants.tsx`, `lib/data/payment.ts` and `app/api/payment-return/route.ts`.
- Payment method names (COD text and so on) are already in the English, Tamil and Hindi dictionaries.

## Backend changes (ecommerce-vegetables-backend)

| Where | Change |
|---|---|
| `src/modules/razorpay/` (new) | A custom payment provider that extends Medusa's `AbstractPaymentProvider`. It creates Razorpay orders, checks payment signatures, captures payments, refunds and cancels, and reads payment status. |
| `medusa-config.ts` | Register the new provider in the Payment Module, next to the existing manual provider. |
| Webhook handling | The provider's webhook method verifies the Razorpay signature and handles `payment.authorized`, `payment.captured`, `payment.failed` and `refund.processed`. It must be safe to receive the same event twice. |
| `.env.template` | Add placeholders for `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` and `RAZORPAY_WEBHOOK_SECRET`. Real values go only in `.env`, never in the repo. |
| Medusa Admin (settings, not code) | Add the provider to the India region (INR). Keep COD enabled in the same region. |
| Mock mode | A `PAYMENT_MODE=mock` setting that swaps in a fake provider for development without Razorpay keys. The backend must refuse to start in mock mode when running in production. |

Refunds: the Admin refund button calls the provider, which calls Razorpay. Cancelling an already paid order should trigger a refund. Stale payment sessions (customer left without paying) should expire.

## Frontend changes (ecommerce-vegetables-frontend)

| Where | Change |
|---|---|
| Stripe files (listed above) | Remove the Stripe code and packages (`@stripe/stripe-js`, `@stripe/react-stripe-js`) and the Stripe env vars. |
| `lib/constants.tsx` | Replace the Stripe, PayPal and Medusa Payments entries with a Razorpay entry (title, description, icon). Add an `isRazorpay` helper. Keep the COD entry. |
| `payment-button/index.tsx` | Add a Razorpay button. It loads the Razorpay script, opens the popup with the order id, and on success completes the payment and places the order. On failure or close, it shows a message and lets the customer retry or switch to COD. |
| `payment-wrapper/index.tsx` | Remove the Stripe wrapper. Razorpay needs no wrapper. |
| `payment/index.tsx`, `payment-container/index.tsx` | Show the "Pay online (UPI / Card)" choice next to COD. |
| `app/api/payment-return/route.ts` | Remove the Stripe return handler, or replace it with a Razorpay return handler if we use redirect mode for some UPI apps. |
| `lib/i18n/dictionaries/en.ts`, `ta.ts`, `hi.ts` | New text for the online payment option, "Payment failed, try again", "Payment pending" and similar. |
| `lib/i18n/errors.ts` | Map the Razorpay error messages to translated text. |
| Order pages | Show the payment method and status (paid online or COD) on the order confirmation and order details pages. |

The brand name stays "VegIn" and is never translated.

## Failure cases we must handle

- Customer closes the popup: show a retry message, cart stays intact.
- Payment fails (wrong OTP, bank declined, UPI timeout): show a clear translated message and allow retry or COD.
- Customer paid but the browser died: the webhook completes the order.
- The same webhook arrives twice: ignore the repeat.
- Wrong or missing signature: reject the request.
- Refund requested: Razorpay refunds to the original source, which can take several days for cards.

## Order of work

1. Backend provider with mock mode, so the checkout can be built without keys.
2. Frontend payment step with the mock provider, plus Stripe removal.
3. Sign up for Razorpay, get test keys, swap the mock for the real provider.
4. Test success, failure, webhook and refund in test mode.
5. Add translations (English, Tamil, Hindi) and error mapping.
6. Complete KYC, switch to live keys on staging, and do a real Rs 1 payment from a phone.
7. Write the PR documents for both repos in the dev-docs template.

## What we need from outside the code

- A Razorpay account. Test keys need only an email and phone. Live keys need KYC approval (PAN, bank account, business proof, GSTIN if we have one).
- A public HTTPS URL for the webhook. In development use a tunnel such as ngrok. In production use the real backend domain.
- A production domain with SSL, and the terms, privacy, refund, shipping and contact pages that Razorpay checks (these pages already exist).
- A webhook secret set in the Razorpay dashboard.
- A decision on the refund policy and who checks Razorpay against our orders.

## How to test

- Without a Razorpay account: use mock mode with "Succeed" and "Fail" buttons, and send a hand made webhook with curl.
- With test keys: use Razorpay's test UPI IDs and test cards (check their "Test Payment Details" page, the exact values are not confirmed here). GPay is not involved in test mode.
- With live keys: pay Rs 1 from a real phone with GPay, then refund it and check the refund in the dashboard.

## Decisions to confirm

- Razorpay is the recommended provider. PhonePe is the alternative.
- Keep COD alongside online payment (recommended).
- Place the order only after payment succeeds (recommended).
- Saved cards are left out of the first version because of the extra card-storage rules. **This is on purpose, not a miss.**

## Not in this plan

- Searching or catalog changes, and anything unrelated to payment.
- Subscriptions and EMI.
- Saved cards (see above).
