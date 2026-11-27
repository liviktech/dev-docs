# 2026-10-14 - api-call-cleanup (frontend)

**Backend PR**: `rubine-toughened-glass-backend` - `<backend PR link>` (adds `is_photo` and `has_receipt` to the PI table list). Merge the backend first. If the frontend goes out first, every Photos and View Receipt button on PI Status shows as disabled until the backend is deployed.

## Summary

Several screens called endpoints they did not need: a second sales rep list, a KPI "one row" request, a bulk photos list, the company list, projects, and the whole PI list. This PR removes those calls or makes them load only when the screen really uses them, and removes Master Data from the PWA menu. It also disables Save in the Edit Work Order modal until the required fields are filled and removes its Approve button.

I checked that the project builds its types and that the changed files lint clean for the new code. I did not run the app in a browser or check the network tab, so the "calls removed" claims are from reading the code. See "Test plan".

**Trimming the sales rep and party responses to only the fields shown is not included on purpose** - see "Decisions made" below. The backend endpoints have no option for a short response, and this PR is frontend plus one small backend change for photos. This was a choice, not a miss.

## What changed

### rubine-toughened-glass-frontend
| File | Change |
|---|---|
| `src/features/pwa/mobile-shell.tsx` | The PWA menu no longer shows Master Data. It lists only the modules the user can view, plus Install app and Sign out. |
| `src/components/layout/mobile-menubar.tsx` | The mobile top menu no longer shows Master Data (unused imports and state removed). |
| `src/features/enquiries/enquiry-form-modal.tsx` | Removed the `/enquiries/sales-reps` call. The sales rep list now comes only from `/master/sales-reps` and keeps just `id` and `name`. Removed an unused `approveEnquiry` import. |
| `src/features/pwa/enquiry/mobile-enquiry-form.tsx` | Same change for the mobile enquiry form. |
| `src/features/sales-reps/queries.ts` | `useSalesReps` accepts an optional `select`, so a screen can keep only the fields it uses. Other screens are unchanged. |
| `src/routes/work-orders.tsx` | The KPI cards use the counts from the list response. The separate `limit=1` request runs only while a search, party or date filter is set. |
| `src/features/pwa/pi-status/mobile-pi-list.tsx` | Same KPI change for the PWA PI list. |
| `src/features/order-booking/work-order-edit-dialog.tsx` | Approve button and its code removed. Save is disabled until Delivery Date and Dispatch Type are filled. After saving, the detail is not downloaded again. |
| `src/features/order-booking/use-work-order-approval.ts` | New `refreshWorkOrdersAfterChange`: writes the save or approve response into the detail cache and refreshes only the lists. Approving no longer re-downloads the work order. |
| `src/features/pi-status/components/pi-status-table.tsx` | Removed the bulk `/ai-extraction/photos?screen=order_booking` calls. The Photos button reads `isPhoto` and View Receipt reads `hasReceipt` from the PI list. A receipt's photos load only when View Receipt is clicked. |
| `src/features/pi-status/types.ts` | New `hasReceipt` field on the PI record. |
| `src/features/pi-status/api.ts` | Maps `has_receipt` to `hasReceipt`. |
| `src/features/invoices/create-proforma-invoice-dialog.tsx` | Removed the `/master/projects` call (the project name comes from the booking or the saved PI). The parties list loads only for the party picker step. |
| `src/features/order-booking/booking-preview-modal.tsx` | `my-company` loads only while the preview is open. |
| `src/features/order-booking/header-detail-tab.tsx` | The PI list loads only on the Confirmed and All tabs, because only confirmed bookings show the Invoice column. |
| `src/features/order-booking/booking-header-form.tsx` | Removed the `/master/companies` call and the code that filled the hidden company field. |
| `src/features/order-booking/schema.ts` | `companyId` is optional. |
| `src/features/order-booking/api.ts` | The create and update body no longer sends `companyId`. |

## Why

The pages were making requests whose results were thrown away, duplicated by another request, or only used in a view the user had not opened. On the PI list the bulk photos request returned the photos of every order just to colour a button. Each extra request slows the page and loads the server for nothing.

## How

**Use what is already loaded**: the Work Order and PWA PI KPI cards read the status counts that come with the list. A separate unfiltered request is made only while a filter is active, because the cards must ignore filters.

**Load on demand**: photos, receipts, company details, the PI list for the Invoice column, and the parties list now load when the user opens the thing that needs them.

**Do not re-download what the response already holds**: saving or approving a work order returns the full work order, so it goes straight into the cache and only the lists are refreshed.

**Drop what the server decides**: the booking API takes the company from the signed-in user, so the frontend no longer picks or sends one.

## Decisions made

**Keep `/master/sales-reps` and drop `/enquiries/sales-reps`.** They return the same list. The master one is the one the Party dialog already uses, and the enquiry one was refetched on every save.

**Order charges stay in the New PI form.** The form uses them to build the default surcharges, so removing the call would drop those charges from the PI total.

**Sales rep and party responses are not shortened.** The endpoints have no short-response option. The sales rep dropdown now keeps only `id` and `name` in the screen, but the network response is unchanged. The enquiry form's Project dropdown needs each party's projects, so parties cannot be cut to `id` and `name` without loading a party's projects after it is chosen.

**The sales rep label is the name only.** It used to add the rep code in brackets.

**The Work Order edit modal keeps using the full work order endpoint.** The modal needs about ten header fields, but the same endpoint feeds the approval review and returns items and pieces. A `view=header` option on the backend would fix that.

## Bugs found while testing (already there before, not fixed here)

**1. The PI list for the Invoice column only looks at the first 20 PIs.** `fetchPIStatusList` defaults to 20 rows, so a confirmed booking whose PI is not in that page shows "Invoice Pending". This was true before this PR.

**2. A leftover `salesRepsQueryOptions` is unused.** It is harmless and left in `src/features/enquiries/queries.ts`.

## Notes

- The dev proxy points at the deployed backend through `VITE_PROXY_TARGET`. To see the photo flags before the backend is deployed, run the backend locally and set the target to `http://localhost:5100`.
- Type errors in `src/features/pi-status/api.ts` (`advance_type`) and in the two PWA detail screens (`printWorkOrder` missing from `useWorkOrderPrint`) were reported while this work was going on. They come from other edits in progress, not from this PR, and are not fixed here.

## Next steps

- Add `?view=header` (or similar) to the work order endpoint, and short-response options for parties and sales reps, then use them in the Enquiry, Order Booking, PI Status and Work Order screens only.
- Load a party's projects after the party is chosen, so the parties list can be cut to `id` and `name`.
- Take the "limit 20" PI lookup out of the Order Booking list by returning the PI status with each booking.

## Test plan

**Automated tests (none for these screens):**
- [x] `npx tsc -b`: no errors in the files changed by this PR (earlier errors in other files are listed in Notes).
- [x] `npx eslint` on the changed edit dialog, PI list and menubar files: no new problems. Existing `any` and set-state-in-effect warnings in untouched code remain.

**Manual checks (local):**
- [ ] Network tab on Work Orders, PI Status, Order Booking, New PI and New Booking: confirm the removed requests are gone. Not done, checked by reading the code only.
- [ ] Edit Work Order: Save stays disabled until both fields are filled, then saves. Not done in the browser.
- [ ] PI Status: Photos and View Receipt enable only for rows that have them. Needs the backend change. Not done.
- [ ] PWA menu on a phone shows no Master Data. Not done.
