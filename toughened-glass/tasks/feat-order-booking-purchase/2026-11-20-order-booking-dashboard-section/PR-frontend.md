# 2026-11-20 - order-booking-dashboard-section (frontend)

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
| `src/features/dashboard/sections/order-booking-section.tsx` (new) | The section: three cards on the shared section and card components. Builds the pie, pipeline and trend data, and the click-through to the Orders list. |
| `src/features/dashboard/components/order-booking-filters.tsx` (new) | The Order Type and Party dropdowns of this section, laid out like the Enquiry filters. |
| `src/features/dashboard/registry.ts` | Registers the section under `ORDER_BOOKING_VIEW`, after Enquiries. |
| `src/features/dashboard/api.ts` | `OrderBookingDashboard` type and `getOrderBookingDashboard`. `DashboardFilters` gets an optional `bookingType`. |
| `src/features/dashboard/queries.ts` | `orderBookingDashboardQueryOptions`, cached like the Enquiry one. |
| `src/features/dashboard/use-dashboard-filters.ts` | Two new URL values, `obType` and `obParty`. Reset clears them. |
| `src/features/dashboard/sections/enquiry-section.tsx` | The Approved bar is clickable and opens Approved and Converted together. |
| `src/features/order-booking/api.ts` | The table request can send `hasPi`. |
| `src/features/order-booking/header-detail-tab.tsx` | The Orders list reads the PI filter from the URL, sends it, shows a "PI Raised ×" chip, and clears it with the chip or Reset. |
| `src/routes/orders.tsx` | The Orders route accepts `pi=1` in its URL. |
| `tests/dashboard-filters.test.ts` | Updated for the two new URL values. |

## Why

The Dashboard was built to hold one section per module, so Order Booking was the next section to add. The goal was an exact mirror of the Enquiry section, so users read every section the same way.

Most of the work was agreeing what each figure means, so that a click opens exactly the orders that were counted. The charts and layout were reused.

## How

**One request, three cards**: the section makes one request for the filters in the header. The pie, the pipeline and the trend all read from it, as the Enquiry section does.

**Rights**: the section shows only to a user with Order Booking View, and the route checks the same right. The Dashboard has no right of its own.

**Pipeline**:

- *Total Received* is every order booked in the period, whatever its status.

- *Draft (to confirm)* is orders still in Draft.

- *Confirmed* is orders with status Confirmed, which is the same set as the Confirmed tab of the Orders list.

- *Invoiced (PI Raised)* is orders that have a proforma invoice, using the same rule the Orders list uses. An order with two PIs counts once.

**Trend**: booked orders per day, week or month as bars, orders that are now Confirmed as a line, and the previous period of the same length as a dashed line. Empty buckets show as zero. The grain follows the period length, as in the Enquiry section.

**Dates**: the section counts by booking date, the same field the Orders list filters on.

**Click-through**:

- A pie slice opens the Orders list on the All tab, narrowed to that booking type through the list's search box.

- Total, Draft and Confirmed open the All, Draft and Confirmed tabs.

- Invoiced opens the All tab with the new PI Raised filter on.

- A trend bar opens the list for that bucket's dates, kept inside the period being viewed.

- The period and the party are carried over.

**Separate filters**: the Order Booking Order Type and Party are saved in the URL as `obType` and `obParty`. The Enquiry section keeps `rep` and `party`. The date range and Reset are shared.

**PI Raised on the Orders list**: with `pi=1` in the URL, the list shows a "PI Raised ×" chip. Clicking it or Reset shows all orders again.

**Approved bar (Enquiry)**: the Enquiries list already accepts several statuses, so the bar opens Approved and Converted together. That matches the count on the bar.

## Decisions made

**An Order Type filter instead of Sales Rep.** An order has no sales rep of its own. A rep filter had to use the rep of the enquiry the order came from, which hid every Direct booking. Order Type (Direct or From Enquiry) is exact and useful.

**Separate filters for Order Booking.** The two sections used to share one pair of dropdowns, so changing one changed the other. Order Booking now has its own values in the URL. The date range and Reset stay shared.

**A Draft stage was added to the pipeline.** The Enquiry pipeline has four stages, so this section has four too: Total Received, Draft, Confirmed and Invoiced.

**Confirmed counts only status Confirmed.** That keeps the number equal to the Confirmed tab of the Orders list when the user clicks it.

**Invoiced counts any proforma invoice,** whatever the PI's own status, as the Orders list does today.

**Invoiced and Approved were made clickable.** I first left them not clickable, because there was no single list for them. A filter was added for Invoiced, and a multi-status request was used for Approved.

**The Orders list has no sales rep filter, so none is carried from the dashboard.** The period and the party are.

**The booking type click uses the Orders search box.** The list has no type filter, but its search already matches the booking type, so no list change was needed for that.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- All changes were tested in the browser.
- No database change is needed. The section reads the existing order, enquiry and proforma invoice tables.

## Next steps

- Add more sections to the Dashboard (for example Work Order and Dispatch) using the same section and card components.
- Add a booking type filter to the Orders list if the search-box match is not enough.

## Test plan

**Automated tests:**
- [x] `tests/dashboard-filters.test.ts` (frontend): the URL parser, including `obType` and `obParty`, ignores bad values.
- [x] Full frontend suite: 111 passing.
- [x] Backend type check (`npx tsc --noEmit`) and frontend type check (`npx tsc -b`): no errors.
- [x] Lint on the dashboard files: no errors.

**Manual checks (local):**
- [x] Dashboard: the Order Booking section shows below Enquiry with its three cards, the shared header bar and footers. Tested in the browser.
- [x] Filters: Order Type and Party change only the Order Booking section. The Enquiry section does not move. Tested in the browser.
- [x] Click-through: pie slices, Total, Draft, Confirmed, Invoiced and trend bars open the matching Orders list; the PI Raised chip clears. Tested in the browser.
- [x] Enquiry: the Approved bar opens Approved and Converted. Tested in the browser.
- [x] Loading, error with retry and empty states. Tested in the browser.
