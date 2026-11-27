# 2026-10-28 - dropdown-lookup-apis (backend)

## Summary

Many dropdowns, filters and name lookups downloaded a whole master list (a party with all its addresses and projects, a product with all its rates and charges, a supplier with all its address columns) only to show a name. This PR adds small read-only lookup endpoints that return `id`, `code`, `name` and `isActive`, and switches the screens that only need those fields to them. It also removes a few party and project list loads that were only used to find one name, by sending the name with the booking.

The screens that need more than a name were left alone: the inventory dialogs that use a product's rates, and every screen that reads a party's addresses or GST. I checked the backend against the local database. I did not run the screens in a browser (see "Test plan").

**Rates-dependent inventory dialogs and address screens are not switched on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/master/lookup/lookup.repository.ts`, `lookup.service.ts`, `lookup.controller.ts`, `lookup.routes.ts` | New `GET /master/lookup/products`, `suppliers`, `sales-reps`, `glass-colors`, `glass-finishes`, `work-centers`, `tempering-load-directions`, `tempering-wave-limits`, `tempering-bend-limits` and `shifts`. Plain code / name / active tables share one repository function with a fixed table map, so no table name comes from the request |
| `src/modules/order-booking/order-booking.repository.ts` | `findBookingLabels` returns the party and project names of a booking; table page query also returns `party_name` |
| `src/modules/order-booking/order-booking.service.ts`, `order-booking.types.ts` | `getFullOrderBooking` and the paged table rows now carry `partyName` and `projectName` |
| `src/modules/fir/fir.repository.ts` | `/firs/lookups` also offers a deactivated customer that has FIRs |
| `src/modules/report/sales-income/sales-income-report.repository.ts` | Customer and supplier filter options also offer a deactivated one that has proforma invoices or work orders (customers) or purchases (suppliers) |

## Why

The dropdowns only read a name, but each one downloaded the whole master record. Party and product lists were the worst: the party list carries two JSON aggregates per row, and the product list carries four (sub-products, routing, every rate, charges). Suppliers carry about 11 address columns. The easy part was the endpoints. The real work was checking, screen by screen, what each dropdown actually reads, so that nothing that needs rates, addresses or projects was switched by mistake.

## How

**Lookup shape**: every lookup returns `{ id, code, name, isActive }`, where `id` is the business code (the same value the old lists used as `id`). Shifts also return `startTime` and `endTime`. All take `?activeOnly=true`.

**Active-only versus full**: pick-lists for new records ask for active only, and add back the saved record's own value if it was deactivated since (one single-record read). Filters and name lookups for old records use the full list, because old documents belong to inactive parties and suppliers.

**Cache keys**: each new lookup key sits under its list's existing key family, so the create, update and deactivate code that already refreshes the list refreshes the lookup. Products share `['master','products']`, which `invalidateMasterData` already covers.

**Kept the same on purpose**: the old list endpoints (`/master/parties`, `/master/products`, `/master/suppliers`, ...) are unchanged, and admin pages still use them. No saved data, rate, charge, total or permission logic was touched.

## Decisions made

**Same field names for every lookup.** The earlier catalog drafted `partyName`, `productCode` and similar. I kept `id`, `code`, `name`, `isActive` so no consumer needs an adapter.

**Left as they are:** glass types, glass thicknesses and currencies (tiny and cached), projects (already a slim list), and every list that is derived from documents or hard-coded.

## Notes

- The local database has not had migration `094_fir.sql` applied, so `sales.fir` does not exist there. I could not run the FIR lookups change against it. The SQL mirrors the columns used by the existing FIR queries.
- The sales-income filter change did run against the local database.
- This working tree also holds other changes made at the same time (bulk size update, Add Product edit bundle and product-selection endpoints, PI data on the order list, mobile PI badge fix). They are not described here.
- The `lookup/shifts` response has no rows locally, so the shift label was checked only by reading the code.

## Next steps

- Optional lookups for charges, roles and inventory locations.
- File tickets for bugs 1 and 2 above.

## Test plan

**Automated tests:**
- [x] Backend `npx tsx --test tests/*.test.ts`: 91 tests, 72 pass, 0 fail. No new test file was added for the lookup endpoints.
- [x] Backend `npx tsc --noEmit`: no errors.

**Manual checks (local):**
- [x] Ran the new lookup queries against the local database: suppliers 0 rows, sales reps 5, glass colours 0, glass finishes 1, work centers 3, tempering lists 2 each, shifts 0. Active-only and full lists matched in size.
- [x] Compared `/full` party name and project name for a confirmed booking: the party code, name and project name came back.
- [ ] FIR customer and department filters - not possible locally (see Notes).
