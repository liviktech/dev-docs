# Plan: Cash on Delivery and an Indian payment gateway

Roadmap: Phase 2, item 3. Priority P1 (listed P0 in the report matrix for payment options).

## What is wrong today

- The backend has only one payment provider in the database: `pp_system_default` (manual payment). There is no payment module set up in `medusa-config.ts`, so Medusa uses its default.
- The storefront shows it as "Manual Payment" (`paymentInfoMap` in `src/lib/constants.tsx`). A customer sees a test looking label. Nothing tells them they pay in cash on delivery.
- There is no online payment. No UPI, card or netbanking.
- Stripe code exists in the starter (`isStripeLike`, `NEXT_PUBLIC_STRIPE_KEY`), but there is no Stripe provider in the backend, and Stripe is not the best fit for UPI in India.
- Nothing stops cash on delivery for very large orders or limits it to places we deliver to.

## What we are changing

Step A, Cash on Delivery (do this first, no outside account needed):
1. Keep the built in `pp_system_default` provider, enable it for the India region, and use it as Cash on Delivery.
2. Frontend: in `paymentInfoMap` and the payment step, show it as "Cash on Delivery" with a short line such as "Pay when your vegetables arrive". Use a rupee or cash icon.
3. Order confirmation page: say "Payment: Cash on Delivery, please keep the amount ready".
4. Order is created with payment status "not paid". The admin marks it paid when cash is collected. Write this step down for the shop owner.

Step B, online payments with Razorpay:
5. Choose Razorpay (UPI, cards, netbanking, wallets). It has a test mode and is common in India. The final choice is the owner's. Cashfree or PhonePe would follow the same steps.
6. Backend: add a Razorpay payment provider module. Use an existing community plugin if it supports Medusa 2.22, otherwise write a small provider module in `src/modules/` that follows the Medusa payment provider interface (initiate, authorize, capture, refund, webhook).
7. Register it in `medusa-config.ts` using keys from `.env` (`RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`). Link it to the India region.
8. Add a webhook route so the order is marked paid even if the customer closes the browser.
9. Frontend: add a payment container for Razorpay that opens the Razorpay checkout window, and add `pp_razorpay_razorpay` to `paymentInfoMap`.
10. Add the Razorpay public key to `.env.local` as `NEXT_PUBLIC_RAZORPAY_KEY_ID`.

## Why we are changing it

Cash on delivery is the main way people pay for fresh vegetables in India, so it must be there and must be clear. Online payment cuts failed deliveries and lets the shop get money earlier.

## Files we expect to touch

Backend:
- `medusa-config.ts`
- New `src/modules/razorpay/` (or a plugin install), a webhook route in `src/api/hooks/`
- `.env` (keys, never committed)

Frontend:
- `src/lib/constants.tsx`
- `src/modules/checkout/components/payment/index.tsx`
- `src/modules/checkout/components/payment-container/`
- `src/modules/checkout/components/payment-button/`
- Order confirmed page text

## Risks and things to watch

- **Step B needs a Razorpay account and test keys from the owner. We cannot test it without them, so we build Step A first and ship it alone if the keys are late.**
- Never put the secret key in the frontend or in git.
- Webhook calls must be checked with the signature, or anyone could mark an order paid.
- Money amounts: Razorpay uses paise (minor units), Medusa 2 uses rupees. Convert in one place only.
- Refunds, partial capture and failed payment retries need their own tests.
- Do not enter any real card or UPI details while testing. Use Razorpay test mode details only.

## Out of scope

- GST invoices (Phase 3 area).
- Wallet or saved cards.
- Online payment for COD fee or COD limits. These can be added after the owner decides the rules.

## How to test

- Step A: checkout shows "Cash on Delivery", place order, the order shows as not paid, the confirmation message is clear.
- Step B (with test keys): pay by test UPI and test card, order becomes paid, webhook marks it paid when the tab is closed early, a failed payment lets the customer retry.
- Admin: capture and refund work.
