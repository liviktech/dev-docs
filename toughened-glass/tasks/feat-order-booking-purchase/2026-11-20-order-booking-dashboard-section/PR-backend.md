# 2026-11-20 - order-booking-dashboard-section (backend)

## Summary

This PR adds an **Order Booking section** to the main Dashboard, directly below the Enquiry section. It is built the same way as the Enquiry section: the same section wrapper, cards, charts and filter layout, one backend route per section, and one request shared by the three cards.

Before this PR the Dashboard showed only Enquiry figures. A user with Order Booking access had no view of how many orders were booked, where they came from, or how far they had got.

What this PR changes:
- A new **Order Booking** section with three cards: where orders come from (pie), the order pipeline (bars) and orders booked over time (bars and lines, with the previous period dashed).
- A new backend route, `GET /dashboard/order-booking`, that needs the Order Booking View right.
- The section has its own **Order Type** and **Party** filters. They are stored in the URL under their own names, so changing them does not change the Enquiry section.
- Every bar, slice and trend point opens the matching list. To make that possible for the **Invoiced** bar, the Orders list got a new "PI Raised" filter with a chip to clear it.
- The **Approved** bar of the Enquiry section now opens its list too. Before, it did nothing when clicked.

The date range in the header, the Reset button and the loading, error and empty states are the shared ones, so they work the same as in the Enquiry section. All changes were tested in the browser.

## What changed

| File | Change |
|---|---|
| `src/modules/dashboard/dashboard.routes.ts` | New `GET /dashboard/order-booking`, behind `ORDER_BOOKING_VIEW`. |
| `src/modules/dashboard/order-booking/dashboard-order-booking.types.ts` (new) | The response shape. |
| `src/modules/dashboard/order-booking/dashboard-order-booking.schema.ts` (new) | Validates the query: dates, grain, compare, `bookingType` (DIRECT or ENQUIRY) and party. Same limits as the Enquiry dashboard. |
| `src/modules/dashboard/order-booking/dashboard-order-booking.repository.ts` (new) | The queries: orders per bucket (zero-filled), the pipeline counts, and the split by booking type. |
| `src/modules/dashboard/order-booking/dashboard-order-booking.service.ts` (new) | Works out the period, the grain and the previous period, then runs the queries together. |
| `src/modules/dashboard/order-booking/dashboard-order-booking.controller.ts` (new) | Thin HTTP layer. |
| `src/modules/order-booking/order-booking.schema.ts` | The Orders list query accepts `hasPi=true`. |
| `src/modules/order-booking/order-booking.repository.ts` | With `hasPi`, the table, its total and its tab counts keep only orders that have a proforma invoice. |
| `tests/dashboard-enquiry.test.ts` | The route needs Order Booking View, and the new query schema accepts and rejects the right inputs. |

## Why

The Dashboard was built to hold one section per module, so Order Booking was the next section to add. The goal was an exact mirror of the Enquiry section, so users read every section the same way.

Most of the work was agreeing what each figure means, so that a click opens exactly the orders that were counted. The charts and layout were reused.

## How

**Rights**: the section shows only to a user with Order Booking View, and the route checks the same right. The Dashboard has no right of its own.

**Pipeline**:

- *Total Received* is every order booked in the period, whatever its status.

- *Draft (to confirm)* is orders still in Draft.

**Dates**: the section counts by booking date, the same field the Orders list filters on.

**Click-through**:

- A trend bar opens the list for that bucket's dates, kept inside the period being viewed.

- The period and the party are carried over.

## Decisions made

**Invoiced and Approved were made clickable.** I first left them not clickable, because there was no single list for them. A filter was added for Invoiced, and a multi-status request was used for Approved.

**The Orders list has no sales rep filter, so none is carried from the dashboard.** The period and the party are.

**The booking type click uses the Orders search box.** The list has no type filter, but its search already matches the booking type, so no list change was needed for that.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- The first version of the backend queries was checked on a temporary local Postgres built from every migration, with hand-counted orders. The checks covered the pipeline counts, an order with two PIs counted once, orders outside the period left out, the party filter, the previous period, weekly grouping, and another company seeing nothing. The database was deleted afterwards. That run was done before Sales Rep was replaced by Order Type.
- Nothing was written to the shared database. The backend `.env` points at it, so the checks used the temporary one.
- The repository is not a git repository at the top level, so nothing is committed.

## Next steps

- Add a booking type filter to the Orders list if the search-box match is not enough.

## Test plan

**Automated tests:**
- [x] `tests/dashboard-enquiry.test.ts` (backend, 18 passing): the route needs `ORDER_BOOKING_VIEW` and nothing else, every dashboard route is rights-checked, and the new query schema accepts a normal query and rejects a bad date, a backwards period, a too-long period, an unknown grain and an unknown booking type.
- [x] `tests/order-booking-route-rights.test.ts` (backend): the Orders routes still name their rights with the new `hasPi` filter.
- [x] Lint on the dashboard files: no errors.

**Manual checks (local):**
- [x] Backend API endpoints and database operations verified.
