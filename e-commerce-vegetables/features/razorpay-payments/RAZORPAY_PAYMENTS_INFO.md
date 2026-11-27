# Razorpay Payments (UPI and Card): Complete Process & Field Guide

> **Target Audience:** Store owners, operations staff, support team, and anyone who wants to understand how online payments with UPI and cards work in the VegIn store without technical jargon.

---

## 1. What is Razorpay Payments?

Think of Razorpay as the card machine at a shop counter: the shopper pays on the machine, the machine tells the shop if the money arrived, and only then does the shop hand over the bill.

**Razorpay Payments** lets a shopper pay for an order online with UPI or a debit or credit card at checkout. The shopper picks UPI or Card next to Cash on Delivery, pays in a Razorpay popup, and the order is placed only after Razorpay confirms the money arrived.

### Key Rules:
* 💳 **Methods offered:** Cash on Delivery, UPI and Card. Netbanking, wallets, EMI and Pay Later are not offered on purpose.
* 🛡️ **No order without money:** the store asks Razorpay directly whether the payment succeeded. It does not trust what the shopper's browser says.
* 💰 **Money is taken straight away:** a successful payment is captured immediately, so Razorpay does not refund it on its own after a few days.
* 🔁 **Refunds:** cancelling a paid order refunds the money to the shopper. This is built in but has not been tested yet.
* 🧪 **Test first:** with a Razorpay test account no real money moves. Going live only needs new keys.

---

## 2. Key Components

The shopper sees a normal checkout page. Behind it, three things work together.

```text
┌─────────────────────────────────────────────┐
│               Checkout page                 │
├─────────────────────────────────────────────┤
│  Payment choices                            │
│  • Cash on Delivery                         │
│  • UPI                                      │
│  • Card                                     │
│                                             │
│  Razorpay popup (VegIn green and name)      │
│  • Shows only the method the shopper chose  │
│  • Asks before closing                      │
└─────────────────────────────────────────────┘
```

1. **The popup:** Razorpay's own secure window where the shopper enters the UPI ID or card details. VegIn never sees or stores card numbers.
2. **The Razorpay order:** a record Razorpay creates for each checkout with the exact amount. The payment is tied to it.
3. **The webhook:** a message Razorpay sends to the store when a payment succeeds or fails. It is a backup in case the shopper closes the page right after paying.

---

## 3. Step-by-Step Process

```text
  [1. Pick UPI or Card]  ──►  [2. Press Pay now]  ──►  [3. Pay in popup]
                                                              │
                                                              ▼
  [6. Refund if cancelled]  ◄──  [5. Order placed]  ◄──  [4. Store checks with Razorpay]
```

### Step 1: Pick UPI or Card
At checkout the shopper chooses UPI or Card next to Cash on Delivery. The store creates a Razorpay order for the cart total at this point.

### Step 2: Press Pay now
The Razorpay popup opens. It already shows the chosen method, the store name and a short description. Name, email and phone are filled in from the address.

### Step 3: Pay in the popup
The shopper pays with their UPI app or card. If they try to close the popup, it asks them to confirm first.

### Step 4: The store checks with Razorpay
The store asks Razorpay what happened to this order. Only a successful payment moves on. A failed or cancelled payment shows a clear message that also points to Cash on Delivery.

### Step 5: Order placed
The money is captured and the order is placed.

### Step 6: Refund if the order is cancelled
If a paid order is cancelled, the store asks Razorpay to refund the full amount.

---

## 4. Field-by-Field Explanation

---

### 4.1 Settings the store owner needs
* **Razorpay Key ID:** the public key from the Razorpay dashboard. Test keys start with `rzp_test_`.
* **Razorpay Key Secret:** the private key from the dashboard. It is kept in the server settings and is never shown to shoppers.
* **Webhook Secret:** a password you choose when you create the webhook in the dashboard. Razorpay and the store must use the same one.

---

### 4.2 What the store remembers for each payment
* **Razorpay order ID:** links the checkout to the Razorpay order.
* **Amount and currency:** the cart total. Razorpay works in paise, so Rs 10 is 1000 paise. The store converts this automatically.
* **Payment ID:** saved once the shopper pays. It is used for refunds.

---

### 4.3 What the shopper types
* **UPI ID or card details:** typed into Razorpay's popup only. They are never sent to or saved by VegIn.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Can I try it without a Razorpay account?
Yes. The developer can switch the store to mock mode, which shows "Simulate success" and "Simulate failure" buttons instead of the popup. No money and no account is needed. This mode refuses to run on a live server.

### Q2: How do I test with real Razorpay screens but no real money?
Create a free Razorpay account and use the test keys. Test cards and test UPI IDs from the Razorpay docs work in the popup. A test card payment has been tried and the order went through. UPI test IDs, failed payments and refunds have not been tested yet.

### Q3: What happens if the shopper pays and then closes the page?
The store also listens to Razorpay's webhook, so the payment can still be found. This is built but the webhook has not been tested end to end yet.

### Q4: Can shoppers pay with netbanking or wallets?
Not today. This was left out on purpose to keep checkout simple. It can be added later in the storefront popup settings without backend changes.

### Q5: How do we go live?
Switch to the live Razorpay keys and live webhook secret in the server settings, then add the live webhook in the Razorpay dashboard. Nothing else changes. The developer guide has the exact steps.
