# Razorpay Payments Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.
> **Purpose:** A concise reference detailing required settings, the data stored for each payment, validation rules, and the exact steps to set up, test or take Razorpay payments (UPI and Card) live.

---

## 1. Required Fields & Originating Modules

To take an online payment, the system uses data from three places: the **backend environment**, the **cart and billing address**, and the **payment session** that Medusa stores.

### 1.1 Backend Environment
*Set in `.env` in the backend repo. `.env.template` documents the keys. Never print or copy the real values.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **`PAYMENT_MODE`** | Optional | `.env` | Not stored | `mock` uses the fake provider. Anything else, including empty, uses Razorpay. Refused when `NODE_ENV=production`. |
| **`RAZORPAY_KEY_ID`** | Yes (real mode) | `.env` | Not stored | Server fails at startup if missing in real mode. `rzp_test_` for test, `rzp_live_` for live. |
| **`RAZORPAY_KEY_SECRET`** | Yes (real mode) | `.env` | Not stored | Used for the Razorpay API and for checking payment signatures. |
| **`RAZORPAY_WEBHOOK_SECRET`** | Yes for webhooks | `.env` | Not stored | Must match the secret set on the webhook in the Razorpay dashboard. |

---

### 1.2 Cart and Billing Address
*Captured at checkout and used to prefill the popup.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Cart total** | Yes | Cart | Medusa cart totals | Rupees. `src/modules/razorpay/client.ts` converts to paise for Razorpay. |
| **Currency** | Yes | Region | Medusa region currency | Sent to Razorpay as the order currency. |
| **Name, email, phone** | Optional | Checkout address step | Medusa cart billing address | Prefilled in the popup only. |

---

### 1.3 Payment Session Data
*Written by `initiatePayment` in `src/modules/razorpay/service.ts` and stored in the Medusa payment session `data`.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **`razorpay_order_id`** | Yes | Auto-created | Payment session `data` | The popup needs it. Webhooks use it to find the session. |
| **`key_id`** | Yes | Auto-derived | Payment session `data` | The public key, safe for the browser. |
| **`amount`, `currency`** | Yes | Auto-derived | Payment session `data` | Amount is in paise. |
| **Medusa session id** | Yes | Auto-derived | Razorpay order `notes` | Lets a webhook be matched back to the session. |
| **`mock`** | Mock mode only | Auto-derived | Payment session `data` | `true` makes the storefront show the simulate buttons. |

---

## 2. Eligibility & Validation Rules Checklist

Before an order is placed with Razorpay, the system verifies:
1. **Provider on the region:** `pp_razorpay_razorpay` must be added to the India region, with `npx medusa exec ./src/scripts/enable-online-payments.ts` or in Medusa Admin. Otherwise only Cash on Delivery shows.
2. **Keys present:** in real mode all Razorpay keys must be set or the server will not start.
3. **Result checked on the server:** `authorizePayment` asks Razorpay for the payments on the order. Captured is captured, authorized is authorized, all failed is an error, anything else stays pending. Browser data is never trusted.
4. **Amount unchanged:** `updatePayment` keeps the Razorpay order if the amount is the same. If the amount changed it creates a new order, because Razorpay orders cannot be edited.
5. **Webhook signature:** `/hooks/payment/razorpay_razorpay` checks the signature with `RAZORPAY_WEBHOOK_SECRET`. Only `payment.authorized`, `payment.captured` and `payment.failed` are handled. Other events are ignored as not supported.
6. **Mock is blocked in production:** `medusa-config.ts` throws at startup, and `POST /store/mock-payment` returns 404, when mock mode is on with `NODE_ENV=production`.

---

## 3. Implementation Guide

The backend is a custom Medusa v2 payment provider (`AbstractPaymentProvider`, id `razorpay`, so Medusa calls it `pp_razorpay_razorpay`). Cash on Delivery stays on the built in manual provider `pp_system_default`. The storefront shows Razorpay as two choices, UPI and Card, using virtual ids `pp_razorpay_razorpay:upi` and `pp_razorpay_razorpay:card` that exist only in the browser (saved in `localStorage` key `vegin:razorpay-choice`). Everything is already built. These steps are for setting up an environment or going live.

```text
┌─────────────────────────────────────────────────────┐
│        Storefront checkout and Razorpay popup       │
│          (NO CHANGES NEEDED - COMPLETE)             │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│     Backend provider src/modules/razorpay           │
│          (NO CHANGES NEEDED - COMPLETE)             │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│     1. .env (EDIT: keys, PAYMENT_MODE)              │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│     2. Region (RUN: enable-online-payments script)  │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│     3. Razorpay dashboard (CREATE: webhook)         │
└─────────────────────────────────────────────────────┘
```

---

### Step 1: Configure Environment Variables
Edit `.env` in the backend repo (see `.env.template`):

```bash
# Leave empty for real Razorpay. Use "mock" only in development.
PAYMENT_MODE=
# Test keys for development, live keys for production
RAZORPAY_KEY_ID=
RAZORPAY_KEY_SECRET=
RAZORPAY_WEBHOOK_SECRET=
```

Restart the backend after changing these. The test account and the live account have different keys and different webhook secrets.

---

### Step 2: Add the provider to the region
Run once per environment:

```bash
npx medusa exec ./src/scripts/enable-online-payments.ts
```

The script adds `pp_razorpay_razorpay` to the India region and is safe to run twice. It can also be done in Medusa Admin under the region settings.

---

### Step 3: Create the webhook in the Razorpay dashboard
Point it to `https://<backend-host>/hooks/payment/razorpay_razorpay`, use the same secret as `RAZORPAY_WEBHOOK_SECRET`, and tick `payment.authorized`, `payment.captured` and `payment.failed`. A public URL is needed, so in development use a tunnel.

---

### Step 4: Going live
Replace the three env values with the live ones, create the live webhook, restart the backend. No code change. A cart that already has a payment session from before the switch should choose the payment method again so a new session is created.

---

## 4. What Already Works (Zero Changes Needed)

When you make the above changes, the following work automatically without modification:

1. **Order creation:** a Razorpay order is created per payment session with the amount in paise.
2. **Server side result check and capture:** `authorizePayment` verifies the result and captures straight away, so Razorpay does not auto refund an uncaptured payment.
3. **Refund and cancel:** `refundPayment` refunds an amount against the stored payment id. `cancelPayment` refunds in full if the payment was captured. Neither has been tested live yet.
4. **Popup behaviour:** `src/modules/checkout/components/payment-button/razorpay-button.tsx` in the frontend opens the popup on the chosen method only, applies the VegIn green (`#15803d`), name, logo and description, and asks before closing. It replaces Razorpay's browser alert with the store's own translated message.
5. **Translations:** all payment texts exist in English, Hindi and Tamil. The Hindi and Tamil text has not been reviewed by a native speaker.
6. **Mock mode:** `PAYMENT_MODE=mock` swaps in `src/modules/razorpay-mock` and the dev only `POST /store/mock-payment` route, so checkout can be tested with no account.

**Not yet checked:** UPI test IDs, webhook delivery, refunds, failed payments, the popup look in the browser, and the startup refusal in production. A real test card payment was checked and the order was placed. See `tasks/feat-upi-payment/2026-11-03-razorpay-upi-card-payments/` for the full test plan.
