# 2026-10-12 - paged-lists-and-pi-advance (frontend)

## Summary

This PR makes the Work Order, Order Booking and PI Status lists load one page at a time from the server with only the columns they show, adds a reusable date range filter, moves the scan widget into the page header, cuts the number of calls on the Enquiry tab, and shows the real advance and balance on the PI Status table.

None of this was checked in a browser. I ran the TypeScript check, which still reports errors in files I did not change. See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/features/pi-status/pi-status-page.tsx` | Server-side paging with `view: 'table'`. Export CSV now fetches the full rows of the current page when clicked. |
| `src/features/pi-status/api.ts` | `fetchPIStatusPage` sends `view`. The record mapper takes advance type, value, percent, amount and balance from the API when the row has them, and keeps the browser cache only for rows that do not (such as the detail). The 50% default is no longer applied to list rows. |
| `src/features/pi-status/components/pi-status-table.tsx` | Advance Amt and Balance Amt just show the mapped values. A PI with no stored advance shows "-" and a balance equal to the total. |
| `src/features/pi-status/components/print-options-modal.tsx` | Print loads the full PI first, because the list row no longer carries all fields. |
| `src/features/order-booking/*` | Booking table uses server paging and a lean query (`bookingTableQueryOptions`, `listBookingTablePage`). |
| `src/features/order-booking/components/header-detail-tab.tsx` | Server-side pagination. |
| `src/components/date-range-filter.tsx` (new) | Reusable From and To filter. Used on the Work Order page. |
| `src/features/order-booking/work-orders.tsx` | Date filter and server paging. Starting an approval fetches the full work order. |
| `src/features/production-scan/scan-widget.tsx`, `src/components/page-header.tsx` | The scan widget is a button in the page header (`HeaderWithScan`), with a curved notch, instead of a footer widget. Used on the Order Booking page and the scan pages. |
| `src/features/enquiries/*` | Fewer calls on load. Sales reps are only fetched when needed. |

## Why

The lists fetched every record with every field, so loading slowed down as data grew. The PI Status advance and balance were calculated in the browser from a saved percentage or a 50% guess, so they could differ from what was approved.

## How

**Paging**: each list keeps the page, page size, filters and search in its query key, uses `keepPreviousData` so the table does not blank between pages, and passes `manualPagination` to the shared table.

**Lean rows**: the table asks for `view=table`. Anything that needs more (Print, Export, the PI view, starting an approval) loads the full record when it is used.

**Advance**: the mapper checks whether the row has `advance_type`. If it does, the server values are used as they are. If not, the old cache and fallbacks apply, so the PI detail keeps working. `advanceReceived` no longer treats the new advance amount as a received payment.

## Decisions made

**No 50% guess on the list.** A PI without a stored advance shows "-". This will look different for the 33 PIs that previously showed 50%.

**Export fetches on click.** The lean rows do not have the export columns, so the export asks for the full page when the button is pressed.

## Notes

- The backend needs a restart, otherwise the table shows no advance and the paging calls are ignored.
- The browser cache of advances (`rubine_pi_advance_terms`) is still written by the approve flow and still used by the PI detail.
- Teammate changes in other files are not part of this document.

## Next steps

- Return the advance on the PI detail from the backend and remove the browser cache.
- Fix the existing TypeScript errors in the invoices, inventory and pi-status files.

## Test plan

**Automated tests (none for these screens):**
- [ ] `npx tsc --noEmit -p tsconfig.app.json`: still reports errors in files not changed here (`create-proforma-invoice-dialog.tsx`, `pi-status-drawer.tsx`, `invoice-preview-modal.tsx` and others) and an existing `pi_number` error in `pi-status/api.ts`. No new error was found in the changed lines.

**Manual checks (local):**
- [ ] PI Status: page through, change filters and tabs, check counts and the Advance and Balance columns for a percentage, a fixed value and a PI with no advance. Not checked.
- [ ] PI Status: Print and Export CSV give the same content as before. Not checked.
- [ ] Order Booking and Work Orders: paging, search, date filter. Not checked.
- [ ] Scan widget in the header on every page, and on small screens. Not checked.
- [ ] Enquiry tab: network calls on load. Not checked.
