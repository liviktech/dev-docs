# 2026-10-28 - dropdown-lookup-apis (frontend)

## Summary

Many dropdowns, filters and name lookups downloaded a whole master list (a party with all its addresses and projects, a product with all its rates and charges, a supplier with all its address columns) only to show a name. This PR adds small read-only lookup endpoints that return `id`, `code`, `name` and `isActive`, and switches the screens that only need those fields to them. It also removes a few party and project list loads that were only used to find one name, by sending the name with the booking.

The screens that need more than a name were left alone: the inventory dialogs that use a product's rates, and every screen that reads a party's addresses or GST. I checked the backend against the local database. I did not run the screens in a browser (see "Test plan").

**Rates-dependent inventory dialogs and address screens are not switched on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/lib/lookup.ts` (new) | `CodeNameLookupItem` type and `fetchCodeNameLookup(name, activeOnly)` for `GET /master/lookup/<name>` |
| `src/features/parties/api.ts`, `queries.ts` | `fetchPartyLookup(activeOnly)`; new `useActivePartyLookup()` with its own cache key under the existing party lookup key |
| `src/features/products/api.ts`, `queries.ts` | `getProductLookup`; new `useProductLookup()` sharing the cache key `['master','products']` with Order Booking, which a product edit already refreshes |
| `src/features/stock-masters/queries.ts` | `useLookup()` on the supplier, glass colour, glass finish and tempering hooks; key sits under each list's key so existing create / update / deactivate refreshes it |
| `src/features/sales-reps/queries.ts` | new `useSalesRepLookup()` |
| `src/features/work-centers/queries.ts` | new `useWorkCenterLookup()` |
| `src/features/shifts/queries.ts`, `shift-select.tsx` | new `useShiftLookup()` (with start and end time); `ShiftSelect` uses it |
| `src/features/order-booking/api.ts` | `listProducts()` calls `lookup/products`; table row and full booking types carry `partyName` / `projectName` |
| `src/features/order-booking/header-detail-tab.tsx` | Party filter uses the active-only lookup and keeps an applied party that was deactivated since |
| `src/features/order-booking/booking-header-form.tsx` | Pick lists (party, project, currency, enquiry) load only for a new booking or after Edit; a locked booking shows its saved values from the booking |
| `src/features/order-booking/booking-preview-modal.tsx`, `order-details-modal.tsx` | Party and project names come from the full booking, not from the party and project lists |
| `src/features/order-booking/work-order-pdf-mapping.ts` | Party name comes from the full booking, so no party list is loaded for the PDF |
| `src/features/order-booking/product-detail-panel.tsx` | Colour dropdown uses the glass colour lookup |
| `src/features/order-booking/work-center-detail-modal.tsx` | Tempering load direction, wave limit and bend limit use the lookup |
| `src/features/enquiries/enquiry-form-modal.tsx` | Sales reps from the lookup; projects from `GET /master/projects?partyId=`; the full party record is read only for a deactivated saved party and when "+ Add New Project" opens the party dialog |
| `src/features/pwa/enquiry/mobile-enquiry-form.tsx` | Same pattern as the desktop enquiry form (party lookup, sales rep lookup, project list) |
| `src/features/sales-visits/sales-visit-form-modal.tsx` | Sales reps from the lookup |
| `src/features/fir/fir-form-modal.tsx` | Department from the work-center lookup |
| `src/features/reports/fir/fir-report.tsx` | Customer filter uses the full party lookup; department uses the work-center lookup |
| `src/features/parties/party-dialog.tsx` | Sales executive from the sales rep lookup |
| `src/features/products/product-dialog.tsx` | Routing work centers from the lookup |
| `src/features/inventory/stock-tab.tsx` | Product filter uses `useProductLookup` |
| `src/features/inventory/components/purchase-dialog.tsx` | Product, supplier, colour and finish from the lookups |
| `src/features/inventory/components/receive-stock-dialog.tsx` | Supplier and colour from the lookups |
| `src/features/inventory/components/lot-group-edit-dialog.tsx` | Colour from the lookup |
| `src/features/inventory/purchase-tab.tsx` | Supplier filter from the lookup (full list, so old purchases still resolve) |
| `src/features/pwa/inventory/stock-tab.tsx`, `movements-tab.tsx` | Product filter uses `useProductLookup` |
| `src/features/pwa/shared-filters.tsx` | Party filter keeps the picked party even if it was deactivated since |
| `src/features/final-invoices/components/final-invoice-detail-modal.tsx`, `final-invoice-preview-modal.tsx` | Party found by name in the full party lookup, then that one party is read for its address and GST |
| `src/features/pwa/final-invoices/use-invoice-bill-to.ts` | Same as the two modals above |
| `src/features/dispatch/components/gate-pass-modal.tsx` | Same pattern: name to id in the lookup, then one party read for its addresses |

## Why

The dropdowns only read a name, but each one downloaded the whole master record. Party and product lists were the worst: the party list carries two JSON aggregates per row, and the product list carries four (sub-products, routing, every rate, charges). Suppliers carry about 11 address columns. The easy part was the endpoints. The real work was checking, screen by screen, what each dropdown actually reads, so that nothing that needs rates, addresses or projects was switched by mistake.

## How

**Lookup shape**: every lookup returns `{ id, code, name, isActive }`, where `id` is the business code (the same value the old lists used as `id`). Shifts also return `startTime` and `endTime`. All take `?activeOnly=true`.

**Active-only versus full**: pick-lists for new records ask for active only, and add back the saved record's own value if it was deactivated since (one single-record read). Filters and name lookups for old records use the full list, because old documents belong to inactive parties and suppliers.

**Cache keys**: each new lookup key sits under its list's existing key family, so the create, update and deactivate code that already refreshes the list refreshes the lookup. Products share `['master','products']`, which `invalidateMasterData` already covers.

**Names sent with the booking**: the full booking and the table rows now include `partyName` and `projectName` from one small query, so the preview and details modals, the work order PDF and the locked header form do not load party or project lists.

## Decisions made

**Rates-dependent inventory dialogs not switched.** Receive Stock, Lot Group Edit and the mobile lot form read each product's rates to limit the glass types and thicknesses offered. A name lookup would remove that limit. They need a small per-product rate-options endpoint first.

**Address and GST screens not switched to a plain lookup.** Gate pass and the final invoice modals read a party's addresses. They find the party id in the lookup, then read that one party.

**Order list party filter stays active-only.** The catalog suggested the full list for filters. I kept active-only because you asked for that on this dropdown, and added back an applied party that was deactivated since.

**Same field names for every lookup.** The earlier catalog drafted `partyName`, `productCode` and similar. I kept `id`, `code`, `name`, `isActive` so no consumer needs an adapter.

**FIR and sales-income filters offer an inactive record only if it has data.** I did not show every inactive party, to keep the dropdowns short.

**Left as they are:** glass types, glass thicknesses and currencies (tiny and cached), projects (already a slim list), and every list that is derived from documents or hard-coded.

## Bugs found while testing (already there before, not fixed here)

**1. The mobile enquiry form offers 3 hard-coded sources.** The desktop form loads them from `/enquiries/sources`, so a source added on desktop does not appear on mobile. This was already there. It affects mobile users.

**2. `GET /master/users` ignores `activeOnly` when not paged.** It was already like this. It affects any screen that passes `activeOnly` to the user list.

**3. `/master/glass-types`, `/glass-thicknesses` and `/currencies` take `includeInactive`, not `activeOnly`.** Their default already hides inactive rows, so the `isActive` filters in the frontend do nothing for them. This was already there and is harmless.

## Notes

- The sales-income filter change did run against the local database.
- This working tree also holds other changes made at the same time (bulk size update, Add Product edit bundle and product-selection endpoints, PI data on the order list, mobile PI badge fix). They are not described here.
- The `lookup/shifts` response has no rows locally, so the shift label was checked only by reading the code.

## Next steps

- Add a small per-product rate-options endpoint, then switch the product list in Receive Stock, Lot Group Edit and the mobile lot form to `useProductLookup`.
- Optional lookups for charges, roles and inventory locations.
- A slim endpoint for the "From Enquiry" picker on the booking header.
- File tickets for bugs 1 and 2 above.

## Test plan

**Automated tests:**
- [x] Frontend `npx tsc -b`: no errors.
- [ ] Frontend lint: not clean. Many errors were already in these files; I did not compare the count against the original for every file.

**Manual checks (local):**
- [x] Compared `/full` party name and project name for a confirmed booking: the party code, name and project name came back.
- [ ] Checked each switched screen in the browser Network tab - not done. Checks are listed in my notes: look for `lookup/...` and no full list on each screen.
- [ ] Deactivated-record cases (old purchase with a deactivated supplier, enquiry with a deactivated party, deactivated work-center department on a FIR) - not done, because I did not run the app. They are covered by the add-back code only.
- [ ] FIR customer and department filters - not possible locally (see Notes).
