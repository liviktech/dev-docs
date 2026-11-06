# 2026-11-06 - credit-limit-tracker-supplier (frontend)

## Summary

The storefront had no way to pay on credit, no delivery stages on an order, no supplier area, and a plain account section. This PR adds "Pay on credit" at checkout and a "Pay what I owe" button (UPI or card), a four step delivery tracker on the order page, a supplier area at `/supplier` with its own sign in and its own header and footer, a page loader, and a redesign of the account pages. Strings are added in English, Tamil and Hindi.

Customer pages and supplier pages are kept apart. A signed in supplier is sent back to `/supplier` from any other page, and customer sign in goes to the store home page, not the account page.

Not run for this document: type check, lint, build or any test. Checked in the browser earlier in the work, when asked: customer login going to the home page, and a supplier being kept out of customer pages. The two small CSS fixes at the end of this work (summary row spacing and autofill label) were not looked at in a browser, see "Test plan" below.

**Suppliers adding products straight to the catalog is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/lib/data/credit.ts` (new) | Server actions for credit: `retrieveCredit` (limit, used, available), `startRepayment` and `confirmRepayment` for "Pay what I owe". The amount is never sent from the browser. |
| `src/lib/data/supplier.ts` (new) | Supplier sign up, login, logout, profile, product list for the picker, and create and list requests. Uses the supplier login `/auth/supplier/supplier-emailpass`. |
| `src/lib/data/cookies.ts` | Adds `_medusa_supplier_jwt` with get, set and remove helpers, so a supplier token never mixes with the customer token. |
| `src/lib/data/customer.ts` | After a successful customer login, redirects to the store home page (using the country code from the form) instead of the account page. |
| `src/lib/data/orders.ts` | Asks for `*fulfillments` on the order and stops caching the order fetch, so the tracker always shows the current stage. |
| `src/lib/constants.tsx` | Adds `pp_credit_credit` to the payment info map and an `isCredit` helper. |
| `src/middleware.ts` | A signed in supplier (valid, unexpired token cookie) is redirected to `/<country>/supplier` from every other page. The token is only read for its expiry, the backend still checks it on each call. The rest is a formatting change in the error log. |
| `src/app/[countryCode]/(main)/supplier/page.tsx` (new) | The supplier page. Shows the sign in and sign up form when logged out, and the dashboard (profile, new request form, my requests) when logged in. |
| `src/modules/supplier/components/auth-form.tsx` (new) | Supplier sign in and sign up form. |
| `src/modules/supplier/components/dashboard.tsx` (new) | The supplier dashboard layout and request list with status labels. |
| `src/modules/supplier/components/request-form.tsx` (new) | The new stock request form with several rows of product and quantity. |
| `src/modules/supplier/components/product-picker.tsx` (new) | Search box dropdown that shows product names only. When nothing matches it offers to ask for a new product by name. |
| `src/modules/layout/components/supplier-menu/` (new) | Header pieces for the supplier area: the account menu (`index.tsx`, `dropdown.tsx`), `brand-position.tsx` which centres the logo and name on the supplier page, and `on-supplier-page.tsx` which shows or hides children by path. |
| `src/modules/layout/templates/nav/index.tsx` | Uses the supplier menu and brand position on the supplier page, and keeps the normal header elsewhere. |
| `src/modules/layout/templates/footer/index.tsx` | Adds a "Supplier" link to the store footer. |
| `src/modules/layout/templates/footer/supplier-footer.tsx` (new) | A footer for the supplier area: brand, a short line about what suppliers do, and contact details. No links into the customer store. |
| `src/app/[countryCode]/(main)/layout.tsx` | Shows the store footer everywhere except the supplier page, and the supplier footer only there. |
| `src/modules/account/components/pay-owed/index.tsx` (new) | The "Pay what I owe" button. Opens a dialog to pick UPI or card, opens the Razorpay popup (or the mock popup), then asks the backend to confirm. |
| `src/modules/checkout/components/payment/index.tsx` | Shows "Pay on credit" for customers with a limit, with the available amount, and disables it when the cart total is above the available credit. |
| `src/modules/checkout/components/payment-button/index.tsx` | Credit uses the same place order button as cash on delivery. |
| `src/modules/checkout/templates/checkout-form/index.tsx` | Fetches the customer's credit and passes it to the payment step. |
| `src/modules/order/components/delivery-tracker/index.tsx` (new) | The tracker: order placed, ready, out for delivery, delivered. Times come from the fulfillment. Shows a single canceled state for canceled orders. Shows the chosen delivery date and slot above it. |
| `src/modules/order/templates/order-details-template.tsx` | Puts the tracker at the top of the order page. The delivery status is first. |
| `src/modules/order/components/status-badge/index.tsx` (new) | A small badge with the order stage, used on order cards and the overview. |
| `src/modules/order/components/payment-details/index.tsx`, `shipping-details/index.tsx`, `order-summary/index.tsx` | Restyled to match the account redesign. "Paid on credit" is shown as the payment method. |
| `src/modules/account/components/overview/index.tsx` | Redesigned overview: navigation cards, one summary card (profile, addresses, credit with the pay button), recent orders. The summary groups are spread across the card with `justify-between`. |
| `src/modules/account/components/account-nav/index.tsx`, `order-card/index.tsx`, `order-overview/index.tsx` | Redesigned nav and order cards. The nav is much smaller than before. |
| `src/modules/account/templates/account-layout.tsx`, `login-template.tsx` | Layout changes for the redesign. Login and sign up forms are centred with more space above and below. |
| `src/modules/account/components/login/index.tsx`, `register/index.tsx` | Pass the country code to the login action, and add a "Are you a supplier?" link. |
| `src/modules/account/components/address-book/index.tsx`, `address-card/add-address.tsx`, `address-card/edit-address-modal.tsx` | Small style changes for the redesign. |
| `src/app/[countryCode]/(main)/account/@dashboard/addresses/page.tsx`, `orders/page.tsx`, `profile/page.tsx` | Page headings and text moved to the new design and translated strings. |
| `src/modules/common/components/page-loader/index.tsx` (new) | A full page loader. |
| `src/app/[countryCode]/loading.tsx`, `(main)/loading.tsx`, `(checkout)/loading.tsx` (new), `account/loading.tsx`, `account/@dashboard/loading.tsx`, `cart/loading.tsx`, `order/[id]/confirmed/loading.tsx` | Use the page loader while a page loads. |
| `src/lib/i18n/dictionaries/en.ts`, `ta.ts`, `hi.ts` | New strings for credit, tracker, supplier area, account redesign and the loader, in all three languages. |
| `src/styles/globals.css` | The floating label also moves up for a browser autofilled input (`input:-webkit-autofill ~ label`), so the label no longer sits on top of the filled in value. |

## Why

The plan is in `plan.md` in this folder. The shop wanted credit for chosen customers, a tracker so customers can see where an order is, and a place for suppliers to offer stock. The account redesign, loader and the layout fixes came from the follow up requests during the work. The easy parts were the tracker and the credit option, since the backend does the rules. The real work was keeping suppliers and customers in separate areas (cookie, redirects, header and footer) and the pay-the-balance flow.

## How

**Credit at checkout**: the payment step asks the backend for the credit. If the customer has no limit, the option is hidden. If the cart total is above the available credit, the option shows but is disabled with a short message. The backend still checks the limit when the order is placed, so this is only a hint.

**Pay what I owe**: the button asks the backend to create a Razorpay order for exactly what is owed, opens the popup, and then asks the backend to confirm. The backend also has a webhook as a backup, so closing the tab right after paying is fine. After a paid confirm the page refreshes and the balance drops.

**Separate supplier area**: the supplier token lives in its own cookie and is sent only by the supplier actions. The middleware sends a signed in supplier back to `/supplier` from other pages, and the layout swaps the header menu and footer by path. Customer sign in goes to the store home page. A customer token and a supplier token never touch each other's routes, because the backend only accepts the right actor type on each route.

**Tracker**: the order is fetched with its fulfillments and without cache. The tracker shows the furthest stage that has a time. For a canceled order it shows one canceled state.

**Labels on autofilled inputs**: Chrome fills a field without making the placeholder disappear, so the label stayed down and overlapped the value. The new selector moves the label up for autofill as well. This is in the shared input styles, so it applies to every form that uses the shared input.

## Decisions made

**Suppliers can ask for a new product by name, not create one.** The picker offers "add as a new product request" when there is no match. The admin decides and creates the product. Direct catalog access for suppliers was left out.

**Supplier page is one page.** Sign in, sign up, new request and my requests are all on `/supplier`, with a footer link, instead of several routes.

**The supplier cookie is separate.** A shared cookie would have been simpler, but then a login on one side could be read as the other.

**No cache on the order page.** The order fetch is `no-store` so the tracker is current. The cost is one uncached request each time the order page is opened.

## Bugs found while testing (already there before, not fixed here)

**1. Deleting one old test customer in Admin failed with "Auth identity not found".** This is in the backend, see `PR-backend.md`. Mentioned here only because it came up while testing the account pages.

## Notes

- Hard refresh the browser after pulling this, the autofill label fix is a global CSS change.
- The credit option and "Pay what I owe" need the backend migrations and the `enable-credit-payment.ts` script to have been run, see `PR-backend.md`.
- Part of the store footer and the account pages changed only in style. Files listed as "small style changes" have no logic change.

## Next steps

- Look at the account overview row and the autofilled sign in form in a browser.
- Run the type check, lint and a production build.
- Add tests for the checkout credit option and the supplier redirect in the middleware.

## Test plan

**Automated tests (none added, none run):**
- [ ] Tests for the checkout credit option (hidden without a limit, disabled over the limit): not written.
- [ ] Tests for the supplier redirect in `src/middleware.ts`: not written.
- [ ] `npm run lint` and `npm run build`: not run when this document was written, so no result is claimed.

**Manual checks (local):**
- [x] Customer login goes to the store home page, not the account page. Checked in the browser when asked.
- [x] A signed in supplier is kept in the supplier area, and customer and supplier pages do not mix. Checked in the browser when asked.
- [ ] Account overview summary row spacing (`justify-between`): changed, not looked at in a browser.
- [ ] Autofilled sign in label (supplier sign in and other forms): changed, not looked at in a browser.
- [ ] Delivery tracker moving through the four stages and the canceled state: not re-run for this document.
- [ ] "Pay on credit" at checkout and "Pay what I owe" with UPI and card: not re-run for this document.
- [ ] Tamil and Hindi text on the new screens: not re-checked for this document.
