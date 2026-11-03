# 2026-11-03 - razorpay-upi-card-payments (frontend)

## Summary

Checkout only offered Cash on Delivery, and the Stripe code from the starter was still in the project. This PR adds UPI and Card as two separate choices next to Cash on Delivery. Both use the Razorpay popup, which opens on the method the shopper picked. After a successful payment the order is placed. The Stripe code and its leftovers are removed. All new text is in English, Hindi and Tamil.

The popup is styled for the store: VegIn green, the store logo, the name and a short order description. Closing it asks for confirmation first.

Checked live: the user paid with a Razorpay test account and a test card and the order went through. The popup styling (green, logo, description) was typechecked but not looked at in the browser, see "Test plan" below.

This covers only the two payment commits on the `feat/upi-payment` branch (`f55c6dd` and `013f9c7`). The home page and address work that also shows in the branch history was already merged through other PRs and is not described here.

**Netbanking and wallets are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/checkout/components/payment-button/razorpay-button.tsx` (new) | The Pay now button. Loads Razorpay's checkout script, opens the popup with the session data, calls `placeOrder()` after payment, and shows our own error for a failed, cancelled or blocked popup. In mock mode it shows "Simulate success" and "Simulate failure" instead. |
| `src/modules/checkout/components/payment-button/index.tsx` | Uses `RazorpayButton` for Razorpay sessions. The whole Stripe button (about 120 lines) is removed. Cash on Delivery path is unchanged. |
| `src/modules/checkout/components/payment/index.tsx` | Shows Razorpay as two choices, UPI and Card, saves the pick in `localStorage`, and creates the payment session with the real provider id. The summary line now says "You will pay securely when you place the order". |
| `src/modules/checkout/components/payment-container/index.tsx` | The Stripe card form is removed. The container is now just the radio choice with icon and description. |
| `src/lib/constants.tsx` | Stripe, PayPal, iDeal and Bancontact entries removed. Adds the two virtual choice ids `pp_razorpay_razorpay:upi` and `pp_razorpay_razorpay:card`, `isRazorpay`, `baseProviderId`, `razorpayChoice` and `savedRazorpayChoice`. |
| `src/lib/data/payment.ts` | Adds `simulateMockPayment`, a dev only call to the backend mock route. |
| `src/lib/i18n/dictionaries/en.ts`, `hi.ts`, `ta.ts` | Removed the Stripe, PayPal, iDeal and Bancontact keys. Added the UPI, Card, Pay now, cancelled, failed, script error, mock mode and popup description texts. Changed `checkout.anotherStep`. |
| `src/lib/i18n/errors.ts` | One more pattern so Razorpay and payment failure errors show the translated "Payment failed" text. |
| `src/app/[countryCode]/(checkout)/checkout/page.tsx` | Removed the `PaymentWrapper` around the checkout form. |
| `src/modules/order/components/payment-details/index.tsx` | Removed the Stripe card last four digits branch. |
| `src/modules/checkout/components/payment-wrapper/index.tsx`, `payment-wrapper/stripe-wrapper.tsx` (removed) | Stripe Elements wrapper, no longer needed. |
| `src/app/api/payment-return/route.ts` (removed) | The Stripe redirect return route. The Razorpay popup does not redirect. |
| `src/modules/common/icons/bancontact.tsx`, `ideal.tsx`, `paypal.tsx` (removed) | Icons for payment methods we do not offer. |
| `package.json`, `pnpm-lock.yaml` | Removed `@stripe/react-stripe-js` and `@stripe/stripe-js`. The lockfile was updated by pnpm, not edited by hand. |

## Why

Shoppers in India expect UPI and cards, and the starter's Stripe setup could never be used here. The backend part of this work is in `PR-backend.md`. The popup itself is Razorpay's, so the real work was around it: two clear choices, opening on the picked method only, and making sure the order is placed from a server checked result.

## How

**Two choices, one provider**: Razorpay is one provider in Medusa, but the shopper sees UPI and Card. The id suffix (`:upi` or `:card`) only exists in the browser. `baseProviderId` strips it before the payment session is created, and the pick is saved in `localStorage` under `vegin:razorpay-choice`. If storage is blocked, the popup falls back to UPI.

**Popup**: `razorpay-button.tsx` loads `checkout.razorpay.com/v1/checkout.js` when the button is pressed, then opens it with the key, order id, amount and currency the backend stored in the session. Name, email and phone are prefilled from the billing address. The `method` option and `config.display` show only the picked method and hide Razorpay's default blocks.

**Look of the popup**: `theme.color` is `#15803d` (brand-700 in `tailwind.config.js`, dark enough for white button text), `image` is `/logo.svg` from the site, `description` is translated, and `modal.confirm_close` asks before the popup is closed.

**Placing the order**: after the popup says paid, `placeOrder()` is called. The backend asks Razorpay what happened to the order, so completing the cart is safe whatever the popup said. A failed payment, a closed popup or a script that did not load shows a translated message that also points to Cash on Delivery.

**Razorpay alerts swallowed**: the Razorpay script shows a browser alert and its own error page if the popup cannot start. The button replaces `window.alert` while the popup is open, shows our message instead, and puts the original back afterwards.

**Mock mode**: if the session data has `mock: true`, the button shows the two simulate buttons and a notice that no real money is charged. This only happens when the backend runs with `PAYMENT_MODE=mock`.

**Kept the same on purpose**: Cash on Delivery behaves as before. The manual provider id `pp_system_default` and its button are untouched.

## Decisions made

**Two radio choices instead of one "Pay online" entry.** An earlier version had one "Pay online (UPI / Card)" entry. UPI and Card next to Cash on Delivery is clearer for shoppers. The old `pp_razorpay_razorpay` entry and the `payment.online` texts are still in `constants.tsx` and the dictionaries and can be removed if nobody needs them.

**Popup limited to the picked method.** Netbanking, wallets, EMI and Pay Later are turned off in the popup, so the shopper only sees what they chose. They can be switched on later in `razorpay-button.tsx`.

**Brand colour kept in the file.** `BRAND_COLOR` is a constant in `razorpay-button.tsx` because the popup runs outside our CSS and cannot read Tailwind tokens. If the brand colour changes, change it here too.

**Stripe removed completely, not just hidden.** The wrapper, the redirect route, the icons and the packages are gone so there is no dead payment code left.

## Notes

- `public/logo.svg` is still the placeholder logo. Replacing that file updates the popup. If the popup does not show the logo, Razorpay may not accept SVG for the header image and a PNG would fix it.
- The logo is passed as a full URL built from `window.location.origin`. Razorpay has to be able to fetch it, so it may not show on a local address. This was not checked.
- The dashboard branding (Settings in the Razorpay dashboard) is a second place where the store name and logo can be set.
- The Hindi and Tamil texts for the new keys are a first draft and have not been reviewed by a native speaker.
- After switching the backend from mock to real mode, choose the payment method again so the cart gets a new payment session.

## Next steps

- Look at the popup in the browser on both UPI and Card and adjust the look if needed.
- Replace the placeholder logo with the real one.
- Native speaker review of the new Hindi and Tamil payment texts.
- Remove the unused `payment.online` texts and the generic `pp_razorpay_razorpay` entry if they are not needed.
- Show a refund status on the order page once refunds have been tested on the backend.

## Test plan

**Automated tests (type check passing, no test suite run):**
- [x] `npx tsc --noEmit`: no TypeScript errors after the popup branding change (`razorpay-button.tsx` and the three dictionaries).
- [ ] Lint and a production build: not run.
- [ ] Unit or end to end tests: none added for the Razorpay button or the choice logic.

**Manual checks (local):**
- [x] Mock mode: the checkout showed "Simulate success" and "Simulate failure" and the summary text was reworded after the user asked why an extra step appeared.
- [x] UPI and Card showed as two choices next to Cash on Delivery after the region was linked to the provider.
- [x] Real Razorpay test account with a test card: the popup opened, the payment went through and the order was placed (reported by the user).
- [ ] Popup colour, logo, description and single method view - not looked at in the browser.
- [ ] UPI test IDs through the popup - not checked.
- [ ] Closing the popup and the confirm question - not checked.
- [ ] A failed card payment - not checked.
- [ ] Hindi and Tamil screens at checkout - not looked at.
