# Plan: credit limit, delivery tracker and supplier login

## What we are building

Three features for the VegIn shop:

1. **Credit limit per customer.** Admin sets a limit for each customer. Until the customer reaches that limit they can buy without paying now. After that, they must pay (UPI or card) to buy.
2. **Delivery tracker.** Every order shows its stage (order placed, order ready, out for delivery, delivered) on the order details page.
3. **Supplier login.** A second type of storefront account for suppliers. A supplier asks to add stock (for example 20 kg potato). The admin checks and approves it. Once approved, the stock is added to inventory automatically and customers can buy it.

Nothing is built yet. This file is the plan only.

## Quick answer: do we need new tables?

| Feature | New tables? | Which |
|---|---|---|
| Credit limit | Yes, 2 | `customer_credit` (limit and used amount per customer), `credit_entry` (history: purchase, repayment, correction) |
| Delivery tracker | **No** | Medusa already stores packed, shipped and delivered times on the order's fulfillment. We only read them. |
| Supplier login | Yes, 3 | `supplier` (the supplier's profile), `supply_request` (one stock request), `supply_request_item` (each product and quantity inside a request) |

So 5 new tables in total, in 2 new Medusa modules (`credit` and `supplier`). We do not change any existing Medusa table. Medusa creates the tables for us with a migration (`medusa db:generate` and `medusa db:migrate`).

**All 5 tables live in their own database schema called `vegin`**, next to the Medusa `public` schema, in the same database. Medusa's tables stay in `public`, ours are only in `vegin`. See the next section.

## Separate schema for our tables

A schema is like a folder inside one Postgres database. We will make one called `vegin` and keep every custom table there.

| Schema | What is in it |
|---|---|
| `public` | All Medusa tables (`product`, `order`, `customer` and so on). Untouched. |
| `vegin` | `customer_credit`, `credit_entry`, `supplier`, `supply_request`, `supply_request_item` |

How:
1. Create the schema once: `CREATE SCHEMA IF NOT EXISTS vegin;`. We add this to a small setup script (`src/scripts/create-vegin-schema.ts`) and to the setup notes, so it also runs on the other computer before the migration.
2. In `medusa-config.ts`, each of our two modules gets `options: { database: { schema: 'vegin' } }`. I checked the installed Medusa 2.22 code and a module-level `database.schema` option is supported (it can also be set with a `DATABASE_SCHEMA` env var per module).
3. `medusa db:generate` and `medusa db:migrate` then create the tables inside `vegin`. In a DB tool you open `vegin` > Tables to see exactly what we created.

Good to know:
- The tables cannot have real database links (foreign keys) to Medusa tables in `public`. This is how Medusa modules work anyway. We store ids like `customer_id` and `variant_id` as plain text and look the records up through Medusa services. The plan already does this.
- The database user in `DATABASE_URL` must be allowed to create schemas. On a local Postgres it is, on a hosted one we must check.
- Backups: a normal full database dump includes both schemas. A dump that is limited to `public` would miss our tables, so use a full dump.
- **Not yet checked live:** that `db:migrate` really creates the tables inside `vegin` (and also writes its migration history there). The config option exists, but I have not run it. This is part of step 1 of the work: create one dummy model, migrate, and look.

## What we have today

Found by reading the code, nothing changed:

- Backend has no custom data models yet. Only the Razorpay payment modules, the delivery slot routes (`src/api/store/delivery-slots`, `src/api/store/carts/[id]/delivery-slot`), one admin widget (`src/admin/widgets/order-delivery-slot.tsx`) and a thumbnail sync subscriber.
- The delivery date and slot are saved in the cart metadata and end up in the order metadata.
- Payment today: COD (Medusa's built in manual provider), plus Razorpay UPI and card.
- Storefront has one login and register flow, and it always signs in as a `customer` (`lib/data/customer.ts`).
- Order details page (`modules/order/templates/order-details-template.tsx`) shows the order details, items, shipping, summary and help. There is no tracker. Order status text exists in `order-details/index.tsx` but only as plain text.
- The seed scripts do not set `manage_inventory` on variants. We must check this first (see Risks).

## 1. Credit limit

### How it works for the customer

1. At checkout, the payment step shows a new option: **Pay on credit**, next to COD, UPI and Card. It shows "Available credit: Rs X".
2. If the order total is within the available credit, the option is enabled. The order is placed and the amount is added to what the customer owes.
3. If the order total is more than the available credit, the option is disabled with a short message ("Not enough credit, please pay now"). The customer pays with UPI or card.
4. The account page gets a small box: credit limit, used, available.

The whole order is either on credit or paid. Part credit and part pay is **left out on purpose** to keep this simple.

### How it works for the admin

- A box on the customer details page in Admin to set the credit limit and see used and available amounts (an admin widget, like the delivery slot one).
- A "Record payment" button in the same box. When the customer pays their dues in cash or bank transfer, the admin enters the amount and the used amount goes down.
- A list of the history (`credit_entry`) so the admin can see where the numbers came from.
- Default limit for a new customer is 0, so nobody gets credit until the admin sets it.

### Backend changes

| Where | Change |
|---|---|
| `src/modules/credit/` (new) | Data models `customer_credit` and `credit_entry`, a service with `getAvailable`, `charge`, `repay` and `release`. |
| `src/modules/credit-payment/` (new) | A small payment provider ("credit"), same style as the Razorpay module. Its authorize step checks the available credit and refuses if the order is too big. This is the real guard, so the rule cannot be skipped from the browser. |
| `medusa-config.ts` | Register the two modules and the credit payment provider. |
| `src/api/admin/customers/[id]/credit/route.ts` (new) | GET (limit, used, history), POST (set limit), and `.../repay` (record payment). |
| `src/api/store/credit/route.ts` (new) | GET the logged in customer's limit, used and available. |
| `src/api/middlewares.ts` | Body validation for the new routes. |
| `src/subscribers/` (new) | On order placed with the credit method, add a charge entry. On order canceled, release it. Must be safe if the same event arrives twice. |
| `src/admin/widgets/customer-credit.tsx` (new) | The admin box described above. |
| `src/scripts/enable-credit-payment.ts` (new) | Adds the credit provider to the India region, like `enable-online-payments.ts`. |

### Frontend changes

| Where | Change |
|---|---|
| `lib/constants.tsx`, `modules/checkout/components/payment*` | Add the "Pay on credit" option and its button. Disable it when the total is above the available credit. |
| `lib/data/credit.ts` (new) | Fetch available credit from `/store/credit`. |
| `modules/account/components/overview` | Show the credit box. |
| `lib/i18n/dictionaries/en.ts`, `ta.ts`, `hi.ts` | New text for all of the above. |
| Order pages | Show "Paid on credit" as the payment method. |

### Things to decide while building

- Two checkouts at the same time could both pass the check. The charge step must lock the customer's credit row (or use a single update that fails if over the limit).
- Order canceled or refunded gives the credit back. Admin edits to an order total are **not handled in the first version**.

## 2. Delivery tracker

### Stages (made for a vegetable delivery shop)

| # | Stage | Customer sees | Comes from |
|---|---|---|---|
| 1 | Order placed | "We got your order" | Order created time |
| 2 | Order ready | "Your vegetables are packed" | Fulfillment `packed_at` |
| 3 | Out for delivery | "On the way to you" | Fulfillment `shipped_at` |
| 4 | Delivered | "Delivered, enjoy" | Fulfillment `delivered_at` |

If the order is canceled, the tracker shows a single "Canceled" state instead.

The chosen delivery date and slot (already in the order metadata) are shown above the tracker, for example "Tomorrow, 7 to 10 AM", so the customer knows when to expect it.

### How the admin moves an order forward

No new admin screen. Admin already has the buttons on the order page: create fulfillment (order ready), mark as shipped (out for delivery), mark as delivered. The tracker just reads what these buttons save. This means **no new table and no backend code** for the tracker itself.

### Frontend changes

| Where | Change |
|---|---|
| `lib/data/orders.ts` | Add `*fulfillments` to the fields asked from `/store/orders/:id`. |
| `modules/order/components/delivery-tracker/` (new) | The tracker: four steps in a row (a column on small phones), done steps filled, current step highlighted, time shown under each done step. |
| `modules/order/templates/order-details-template.tsx` | Put the tracker at the top, so it shows when the customer clicks "See details" on an order. |
| `modules/account/components/order-card` | Optionally show the current stage as a small label on the card. |
| `lib/i18n/dictionaries/en.ts`, `ta.ts`, `hi.ts` | Stage names and messages. |

### Things to check first

- That Medusa 2.22 really fills `packed_at`, `shipped_at` and `delivered_at` on the buttons above, and that the store API returns them. I read the field names from memory, so this must be checked live in the first step.
- A paid order with COD has a payment that is not captured yet. That does not matter for the tracker, it only looks at fulfillments.

### Not in this version

Push, SMS or email messages when a stage changes. Live map tracking. Partial deliveries (one order, many shipments) show the furthest stage reached.

## 3. Supplier login and stock approval

### The flow

1. Supplier opens `/supplier`, registers or logs in. This is separate from the customer login.
2. Supplier clicks "New stock request", picks products (for example Potato) and enters the quantity (20), and sends it. The request is saved as **Pending**.
3. Admin opens "Supply requests" in Admin, sees the pending request, and clicks **Approve** or **Reject** (with a note).
4. On Approve, the backend adds the quantity to the product's inventory in one step. The product can be bought by customers straight away.
5. Supplier sees the request as Approved or Rejected in "My requests".

The product catalog stays owned by the admin. The supplier picks from existing products, they do not create new products. Suppliers proposing new products is **left out on purpose** for now (see open questions).

### Two kinds of accounts

Medusa already supports different account types through "actor types" in its auth module. Customers use `customer` today. We add a second one called `supplier`. The login page is the same email and password method, but the token is for a supplier, so a supplier token cannot open customer pages and a customer token cannot open supplier pages.

### Backend changes

| Where | Change |
|---|---|
| `src/modules/supplier/` (new) | Data models `supplier`, `supply_request`, `supply_request_item` and a service. |
| `medusa-config.ts` | Register the supplier module. |
| `src/api/store/suppliers/route.ts` (new) | POST creates the supplier profile after register (same pattern as the customer: register, login, create profile). GET `/store/suppliers/me`. |
| `src/api/store/supplier/requests/` (new) | Supplier-only: POST create a request, GET list own requests, GET one. Each supplier can only see their own. |
| `src/api/admin/supply-requests/` (new) | Admin: list, get one, `approve`, `reject`. |
| `src/workflows/approve-supply-request.ts` (new) | A Medusa workflow with steps: check the request is still Pending, add the quantity to the inventory level for each item, mark Approved. If a step fails, earlier steps are undone, so stock and status never disagree. Approving twice must do nothing the second time. |
| `src/api/middlewares.ts` | `authenticate("supplier", ["bearer", "session"])` on the supplier routes, `authenticate("customer", ...)` stays on customer routes, plus body validation. |
| `src/admin/routes/supply-requests/page.tsx` (new) | The admin page: list with a status filter, detail view with Approve and Reject. |
| `src/scripts/` (new) | A script to turn on inventory tracking for the existing variants and create their inventory levels at the stock location, if they are missing. |

### Frontend changes

| Where | Change |
|---|---|
| `app/[countryCode]/(main)/supplier/` (new) | Pages: login/register, dashboard, new request, my requests. |
| `lib/data/supplier.ts` (new) | Supplier login, register, logout and request calls. It must keep the supplier token in its own cookie (for example `_medusa_supplier_jwt`) so it never mixes with the customer token. |
| `modules/supplier/` (new) | The UI components (forms, request list, status label). |
| Header / nav | A small "Supplier login" link. |
| `lib/i18n/dictionaries/en.ts`, `ta.ts`, `hi.ts` | New text for the supplier area. |

### Units

The catalog sells every product as a "Pack Size: 1 kg" variant, so 20 kg of potato is a quantity of 20 on that variant. The form will show "kg" next to the quantity and save the number of packs. If we add 5 kg or 10 kg packs later, the form already works because it is per variant.

## Order of work

1. **Check first (small):** inventory tracking on the variants, fulfillment time fields, how auth handles a second actor type, and that a test model really migrates into the `vegin` schema. Write down what is found.
2. **Delivery tracker.** Smallest, no new tables, shows value quickly.
3. **Credit limit.** Backend modules and migration, admin box, then checkout.
4. **Supplier login.** Backend module and workflow, admin page, then the supplier storefront pages.
5. Translations (English, Tamil, Hindi) added along with each feature, not at the end.

Each feature can be its own PR, but they are all planned on this branch (`feat/supplier-login`).

## Risks and unknowns

- **Inventory may not be tracked today.** If variants have `manage_inventory` off, approving a supply request would add stock to nothing and customers could always buy any amount. The script in section 3 fixes this, but turning it on changes shopping behaviour: items with 0 stock become unbuyable. Needs a go ahead before we run it on real data.
- **Credit and money.** We hold a balance owed by the customer, but we do not collect it online. Collection is done by the admin outside the app and entered by hand. This is fine for a small shop, it is not a billing system.
- **Supplier sign up is open.** Anyone could register as a supplier. This is safe for stock because every request needs admin approval, but spam requests are possible. See open questions.
- **Second actor type.** I expect Medusa 2.22 to allow it with a config line and a middleware. If the admin or store routes reject unknown actor types, we need a workaround. This is checked in step 1.
- **Cache.** The storefront caches order data (`force-cache` with tags). The tracker must refresh when the stage changes, so we need to check the cache tag is cleared or use a shorter cache for the order details page.

## Open questions for you

1. Should the admin also approve **supplier accounts** (not only stock requests)? The plan assumes no. Anyone can register, and only approved requests change stock.
2. Should suppliers be allowed to propose **new products** that the admin turns into catalog items? The plan assumes no.
3. Should the credit limit be one number in rupees per customer (as planned), or also a due date after which the customer is blocked until they repay?
4. Should the supplier see **prices** or get paid through the app? The plan assumes no, stock only.

## Out of scope on purpose

- Part credit and part online payment in one order.
- Interest, late fees, due dates and invoices for credit.
- Supplier payments, supplier prices and purchase invoices.
- SMS, push or email on stage changes.
- Live location of the delivery person.
- Suppliers creating or editing products.

## How we will test

- **Tracker:** place an order, then in Admin create the fulfillment, mark shipped, mark delivered. After each click, open "See details" and check the tracker moves. Also check a canceled order, and the three languages.
- **Credit:** set a limit of Rs 500 for a test customer. Order Rs 300 on credit (works, available drops to Rs 200). Order Rs 300 again (credit is disabled, UPI works). Record a payment of Rs 300 in Admin, available goes up. Cancel a credit order, available goes up. Try two checkouts at the same time to check the lock. Try calling the order complete route directly with credit while over the limit, it must be refused.
- **Supplier:** register a supplier, request 20 kg potato, check it is Pending and stock has not changed. Approve it in Admin, check stock goes up by 20 and the product can be bought. Approve again, nothing changes. Reject another request, stock does not change. Check a supplier token cannot open customer account pages and the other way round. Check supplier A cannot read supplier B's requests.
- Run the existing backend integration tests (`integration-tests/`) and add new ones for the credit check and the approve workflow.

## What changed while building

- **No separate `vegin` schema.** I tried it. The migration puts the tables in `vegin` but creates their indexes without the schema name, so `db:migrate` fails. All 5 tables are in the normal `public` schema instead. The schema script was removed.
- **Inventory tracking is already on.** All 100 variants have it and a stock level, so the enable-tracking script was not needed.
- **Credit is enforced in the credit payment provider** (when the payment is authorized). Charge and release happen there too, so the order subscribers in the plan were not needed. Cancelling an order gives the credit back.
- **Supplier storefront is one page** at `/supplier` (login, register, new request, my requests), with a footer link.
- Tracker: the order details fetch is no longer cached, so the stage is always current.
