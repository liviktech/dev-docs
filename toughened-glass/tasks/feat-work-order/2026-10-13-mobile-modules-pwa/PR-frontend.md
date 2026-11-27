# 2026-10-13 - mobile-modules-pwa (frontend)

## Summary

The first PWA PR only had a mobile screen for Enquiry. This PR adds mobile screens for **Inventory, Order Booking, PI Status and Work Orders** inside the installed app, and moves all the common mobile UI into one shared component set so every module looks and behaves the same. The mobile menu now follows the desktop sidebar (modules first, then a Master Data heading with its list). Every mobile screen uses the existing API functions and query hooks of its module, with the same fields, permissions and status rules as desktop. I checked that the production build, TypeScript and eslint on the PWA folder pass. I did not test on a real phone or in browser device emulation.

**Creating and editing an order booking is not rebuilt for mobile on purpose** - see "Decisions made" below. It opens the existing booking editor. This was a choice, not a miss.

**No backend files changed in this task, so there is no PR-backend.md for it.**

## What changed

| File | Change |
|---|---|
| `src/features/pwa/components/*` (new: `styles.ts`, `button.tsx`, `status-badge.tsx`, `card.tsx`, `feedback.tsx`, `sheets.tsx`, `form-fields.tsx`, `list.tsx`, `detail.tsx`, `index.ts`) | The shared mobile component set: buttons, badges, cards, list cards, metric tiles, banners, skeletons, empty and error states, bottom sheets, confirm, action and filter sheets, form fields, searchable picker, search bar, status chips, tabs, list states, load-more footer, detail layouts, action bar |
| `src/features/pwa/hooks/*` (new) | `use-debounced-value` (search), `use-load-more-sentinel` (load next page on scroll), `use-client-paging` (show 20 at a time for lists the desktop loads whole) |
| `src/features/pwa/screen-states.tsx`, `photo-grid.tsx`, `shared-filters.tsx`, `filter-types.ts` (new) | Whole-screen loading, not-found and error states, photo thumbnails card, party and date filter fields shared by the list screens |
| `src/features/pwa/mobile-modules.ts` (new) | One list of the modules that have a mobile screen, the desktop to mobile path mapping, and `redirectToMobileScreen()` used by the desktop routes |
| `src/features/pwa/pwa-access.ts` | One `shouldUseMobileUi()` check replaces the separate Enquiry and Work Order checks. `?pwa=true` in a URL simulates the installed app for testing |
| `src/features/pwa/mobile-shell.tsx` | Menu now follows the desktop sidebar: modules in sidebar order, then a **Master Data** heading with its list, filtered by the same permission check. Buttons use the shared components |
| `src/features/pwa/mobile-no-access.tsx` | Takes the module code and title, so every module uses the same VIEW check |
| `src/features/pwa/mobile-ui.tsx`, `mobile-styles.ts` (removed) | Replaced by `components/` |
| `src/features/pwa/enquiry/*` (moved from `pwa/`) | Enquiry list, details and form rebuilt on the shared components. Same fields, rules and API calls |
| `src/features/pwa/work-orders/*` (new) | Work Order list (cards, tabs with counts, party and date filters) and details (summary, products with stages, pieces, photos) with Approve, Edit and Cancel under the desktop permissions |
| `src/features/pwa/inventory/*` (new) | Inventory with the desktop's three tabs: Stock, Movements, Locations. Lot details and the New and Edit Stock form |
| `src/features/pwa/order-booking/*` (new) | Order Booking list and read-only details (same data as the desktop order preview) with PDF download, Replica, Create PI and Delete |
| `src/features/pwa/pi-status/*` (new) | PI Status list and details, the Party and Admin approval sheet, shared approve and delete logic, and the PI status wording |
| `src/routes/m.inventory.tsx`, `m.inventory_.new.tsx`, `m.inventory_.$lotId.tsx`, `m.inventory_.$lotId_.edit.tsx` (new) | Inventory routes |
| `src/routes/m.orders.tsx`, `m.orders_.$bookingId.tsx` (new) | Order Booking routes |
| `src/routes/m.pi-status.tsx`, `m.pi-status_.$piId.tsx` (new) | PI Status routes. The list keeps the `?createForBooking=` search param the desktop page has |
| `src/routes/m.work-orders.tsx`, `m.work-orders_.$workOrderId.tsx` (new) | Work Order routes |
| `src/routes/m.index.tsx` | `/m` opens the first mobile module the user is allowed to view |
| `src/routes/m.enquiries*.tsx` | Import paths only (no real change) |
| `src/routes/inventory.tsx`, `orders.tsx`, `pi-status.tsx`, `work-orders.tsx`, `enquiries.tsx` | One call added to each `beforeLoad`: in the installed app on a phone it redirects to the mobile screen. Browsers and desktops are unchanged. No change to the desktop pages themselves |
| `PWA.md` | Updated with the Rules section, the folder layout, and a section per module |

## Why

Users want to check stock, bookings, invoices and work orders from a phone, not only enquiries. The risk with adding four modules was four different looking UIs and four copies of the same buttons and sheets, so the common parts were built once and used everywhere. The real work was matching each desktop screen's fields, filters and rules without touching any API call or desktop page.

## How

**Shared components**: A screen is built from `components/` (list card, status chips, bottom sheet, form field and so on) instead of its own markup. A new module is a row in `mobile-modules.ts`, its `/m/...` routes, and one redirect line in the desktop route.

**Same requests as desktop**: Each list uses the module's own list request (Enquiry page, Work Order list, Order Booking table, PI Status page, Inventory lots and movements) and the same status tabs and counts. Server-paged lists load 20 at a time and keep loading as the user scrolls. Inventory lists are loaded whole by the desktop, so mobile does the same and only shows 20 at a time. List query keys sit under each module's existing key prefix, so a change made on desktop refreshes the mobile list too.

**Forms match desktop**: The New and Edit Stock form has the same fields in the same order as the desktop Receive Stock dialog, the same Type to Thickness dependency, the same SQM calculation, the same messages and the same request body. The PI approval sheet has the same fields, checks and messages as the desktop approval dialog.

**Menu**: Both menus (the `/m` screens and the top bar on desktop pages) read the one `nav-items.ts` list the sidebar uses, so they cannot drift apart.

**Kept the same on purpose**: The desktop pages, the API functions, the backend, and the desktop sidebar from 768px up.

## Decisions made

**Order booking create and edit use the existing editor.** The booking editor (header, products, sizes, charges, review) is more than 5,000 lines of dense table forms. Rebuilding it field by field would risk breaking booking rules. "New booking" and "Edit booking" open the existing editor in its desktop layout. The mobile screens cover the list and a read-only details view with PDF download, Replica, Create PI and Delete.

**PI form and preview reuse the desktop dialogs.** New PI, Edit PI and Preview open the existing desktop dialogs, so every field is the same as desktop. They are not phone-optimised. I did not rewrite the 2,700 line PI form.

**Approval logic is copied, not shared.** The desktop PI Status page keeps its approve and delete logic inside the page. To avoid editing the desktop page, `use-pi-actions.ts` repeats the same checks, request, advance cache and messages, with a comment saying so. If the desktop page is refactored later, both should share one hook.

**Not added on mobile:** Export CSV and Decline on PI Status (the desktop approval dialog has no Decline button either), and "Record movement" on Inventory (the desktop page has no button for it, only the unused dialog).

**New PI button is not permission-gated.** The desktop page shows New PI to everyone and the server enforces the right, so mobile does the same to stay identical.

## Bugs found while testing (already there before, not fixed here)

**1. PI detail errors look like "not found".** `fetchPIStatusDetail` catches every error and returns `null`, so the mobile PI details screen cannot tell a network failure from a missing invoice. It shows "not found" with a back button for both. It affects any user opening a PI on a bad connection. The desktop drawer has the same behaviour.

**2. Lint errors in desktop files.** `src/routes/inventory.tsx` has several `any` errors and `src/routes/work-orders.tsx` has a state-in-effect error. They were there before this PR and were not touched.

## Notes

- The earlier request to keep Work Orders free of mobile changes was replaced by a later request to add a Work Order mobile screen to the PWA. The regular browser app and desktop Work Orders page are still unchanged.
- `eslint` still reports `react-refresh/only-export-components` on the thin `m.*` route files. The same pattern exists in the older route files.
- Checked with `vite build`, `tsc -b` and eslint on `src/features/pwa` (clean). Not run on a device or in emulation.

## Next steps

- Test every mobile screen at 320px, 375px and 430px and on a real Android phone, including the PDF download and the payment slip upload.
- Decide whether to rebuild the order booking editor and the PI form for phones as a separate task.
- Add the backend company-level PWA setting (see `PWA.md`), which is still not available.
- Make `fetchPIStatusDetail` return real errors so the mobile screen can show Retry.

## Test plan

**Automated tests (none exist for this area):**
- [x] `vite build`: passes
- [x] `tsc -b`: passes, no errors
- [x] `eslint src/features/pwa`: no errors or warnings
- [ ] There is no `test` script in `package.json`, so no unit tests were run

**Manual checks (local):**
- [ ] Menu shows the sidebar items and the Master Data heading with its list - not checked in a browser
- [ ] Inventory, Order Booking, PI Status and Work Order screens at 320px, 375px and 430px - not checked
- [ ] Party approve, admin approve, edit and delete on a PI - not checked against the running backend
- [ ] New and Edit Stock saves the same data as the desktop form - not checked against the running backend
- [ ] Installed app opens the mobile screens and the browser still opens the desktop pages - not checked, needs a deployed HTTPS build or `?pwa=true`
- [x] No backend files changed (checked with `git status` in the backend repo)
