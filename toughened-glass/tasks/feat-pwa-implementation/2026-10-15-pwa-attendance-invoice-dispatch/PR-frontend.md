# 2026-10-15 - pwa-attendance-invoice-dispatch (frontend)

## Summary

This PR adds the Employee module (employees, daily attendance, payroll), brings Final Invoice and Dispatch to the mobile (PWA) app, adds Previous / Next pagination to every mobile list, fixes mobile cards that slid off to the right on narrow phones, and makes the piece stages page safe when a phone and a desktop work on the same piece.

Before this, there was no employee or attendance screen, Final Invoice and Dispatch had no mobile screens, mobile lists loaded more cards as you scrolled, the PI Status cards and the Draft tab could be pushed past the right edge on some phones, and the stages page on mobile did not show the same stage details as desktop. If one device finished a stage, the other device could still click Start or Complete on the old stage.

Checked: the production build (`vite build`), the TypeScript build (`tsc -b`) and ESLint on `src/features/pwa` and `src/features/production-scan` all pass. **Nothing was tested in a real browser, in device emulation or on a phone**, see the test plan.

**Print label barcode scanning is not changed in this PR on purpose** - see "Bugs found while testing" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/attendance/types.ts`, `api.ts`, `queries.ts` (new) | Types, API calls and query hooks for employees, daily attendance and payroll. Saving attendance and payroll writes the result straight into the cache. |
| `src/features/attendance/use-attendance-sheet.ts` (new) | Shared hook for the attendance sheet of one date. Keeps edits locally and sends only the rows that changed. Used by desktop and mobile. |
| `src/features/attendance/attendance-page.tsx` (new) | The Employee page with three tabs in this order: Employees, Attendance, Payroll. Payroll shows only for users with CONTROL. |
| `src/features/attendance/daily-attendance-tab.tsx` (new) | Attendance tab: date, search, status buttons per employee, "Mark unmarked as Day", save. |
| `src/features/attendance/payroll-tab.tsx` (new) | Payroll tab: month and year, calculation, save, and status change per record. |
| `src/features/attendance/employee-dialog.tsx` (new) | Add and edit employee form. Editing uses the table row, so it makes no extra request. |
| `src/features/pwa/attendance/mobile-attendance.tsx` (new) | Mobile attendance marking only (no payroll, no employee editing). |
| `src/routes/attendance.tsx`, `src/routes/m.attendance.tsx` (new) | Desktop route with the permission guard and the phone redirect, and the mobile route. |
| `src/components/layout/nav-items.ts`, `app-sidebar.tsx` | "Employee" is in its own block below Master Data. Master Data is collapsed by default. |
| `src/features/auth/accessible-routes.ts` | Adds `/attendance` to the routes a user can be sent to. |
| `src/features/production-scan/stage-utils.ts` | New `getColumns` for the stage tables (grinding, holes, fabrication, and the cutting and tempering columns merged from another branch), stage numbers, and the stale stage check: `useStageChangeNotice` with `isStale` and `acknowledge`. |
| `src/features/production-scan/scan-detail-page.tsx` | Desktop stages page. Before Start or Complete it checks that the piece did not move on. Shows a confirmation dialog (title, message, OK) when it did. Sends `stageId` with the scan. |
| `src/features/production-scan/components/stage-work-panel.tsx` | Uses the shared `getColumns`. The Logo and Remark columns are removed from the fabrication table. |
| `src/features/production-scan/api.ts`, `queries.ts` | The scan call sends `stageId`. The piece detail query refetches every 30 seconds. |
| `src/features/production-scan/scan-dialog.tsx` | After a barcode lookup on a phone, opens the mobile piece or mobile work order page instead of the desktop one. |
| `src/features/pwa/work-orders/mobile-piece-detail.tsx` | Mobile stages page shows the same stage details as desktop (holes, grinding, cutting and so on) in the quantity list. Same stale stage check, with the message shown in a bottom sheet with an OK button. |
| `src/features/pwa/pi-status/mobile-pi-preview.tsx` | Uses the real invoice sheet (the same as desktop) through the new shared `SheetPreview`. Portrait and Landscape, zoom, Download and Print. |
| `src/features/pwa/components/sheet-preview.tsx` (new) | Shared component that scales an A4 sheet to the phone width, with zoom and an optional orientation switch. |
| `src/features/pwa/use-sheet-pdf.ts` (new) | Shared Download and Print for a sheet. Print opens the PDF in the phone viewer. |
| `src/lib/pdf/sheet-to-pdf.ts` | Split into `renderSheetToPdf` (returns the PDF) and `downloadSheetAsPdf` (calls it and saves). Desktop behaviour is the same. |
| `src/features/pwa/final-invoices/*` (new, 6 files) | Mobile Final Invoice list, details, preview, approval sheet, bill-to lookup and small format helpers. |
| `src/features/pwa/dispatch/*` (new, 5 files) | Mobile Dispatch list, details, approval sheet, gate pass screen and format helpers. |
| `src/routes/m.final-invoices*.tsx`, `src/routes/m.dispatch*.tsx` (new, 6 files) | The six mobile routes. Each only wraps a screen in `MobileNoAccess`. |
| `src/routes/final-invoices.tsx`, `src/routes/dispatch.tsx` | Only one added step: redirect to the mobile screen in the installed app on a phone. |
| `src/features/pwa/mobile-modules.ts` | Adds Final Invoice, Dispatch and Employee to the mobile menu registry. |
| `src/features/pwa/components/list.tsx`, `index.ts` | New `Pagination` component (Previous, Page x of y, Next) replaces `LoadMoreFooter`. The card grid is now `grid-cols-1` with `min-w-0` on each card, which fixes the sideways overflow. |
| `src/features/pwa/hooks/use-paged-query.ts` (new) | Shows one page at a time from a server paged list. Visited pages stay cached. |
| `src/features/pwa/hooks/use-client-paging.ts` | Now returns page controls instead of a "load more" count. |
| `src/features/pwa/hooks/use-load-more-sentinel.ts` (removed) | No longer used. |
| `src/features/pwa/enquiry/mobile-enquiry-list.tsx`, `order-booking/mobile-order-list.tsx`, `pi-status/mobile-pi-list.tsx`, `work-orders/mobile-work-order-list.tsx` | Use `usePagedQuery` and `Pagination`. |
| `src/features/pwa/inventory/stock-tab.tsx`, `movements-tab.tsx` | Use `Pagination` with the updated `useClientPaging`. |
| `src/features/pwa/shared-filters.tsx` | The party filter can filter by party code (needed by Final Invoice). Default is still the party id. |
| `PWA.md` | Updated for the new modules, the pagination rule and the new folder layout. |

## Why

The Employee module and the mobile Final Invoice and Dispatch screens were asked for by the business, following the rules in `PWA.md`: reuse the shared mobile components, keep the same fields and permissions as desktop, make no new API endpoints, and avoid extra requests. The pagination change was asked for after the first PI Status review. The stale stage check is there because a phone user and a desktop user often work on the same piece, and the second person should never be able to redo a stage the first person finished. The overflow bug came from a card grid with no fixed column width, so one long party name or PI number stretched the whole column.

## How

**Employee module.** Desktop and mobile share the same query hooks and the same sheet hook. Search on the attendance sheet runs on the loaded day, so typing does not send requests. Only changed rows are sent on save.

**Mobile Final Invoice.** The list uses the same server paged request as desktop with the lean `view: 'table'` option. The summary tiles come from the same response as the list. Party and Admin approval follow the same order and rules as desktop (only the next step is offered, nothing once declined or fully approved). The details screen reuses the desktop detail request and the Bill-To lookup copied from the desktop dialog.

**Mobile Dispatch.** Like desktop, the whole list is loaded once and the tab and party filters run on screen. The unfiltered list for the tiles and party options is only requested while a search or a date is set. Approve Dispatch has the same required fields and messages as desktop. The gate pass screen shows the address form first, then the same sheet as desktop.

**Pagination.** Server paged lists keep `useInfiniteQuery` under the hood but show one page at a time, so a page that was already visited does not call the server again. Pages needed for a jump are fetched one by one.

**Stale stage check.** Clicking Start or Complete first reloads the piece. If its stage progress is different from what the screen shows, the confirmation dialog opens (all stages completed, one stage completed, several stages completed, or the stage was started on the other device) and no request is sent. The page then already shows the new state. When the screen's own Start or Complete succeeds, that result is marked as seen so it does not raise the dialog. Any other change that the page receives (30 second refresh, coming back to the window) opens the dialog too. The backend also refuses the stale call, so this is a second line of defence.

## Decisions made

**Pagination uses Previous and Next, not page numbers, and replaces scroll loading.** It is simple to use with one thumb and works for both server paged and fully loaded lists. This changes the older `PWA.md` rule about auto loading, which is updated.

**Print on mobile opens the PDF in the phone viewer.** The desktop print uses a hidden frame, which phones do not support well. Download saves the same PDF.

**Approving a Final Invoice is done from the invoice screen, not from the list row.** This follows how PI Status works on mobile.

**A freshness request before every Start or Complete.** It costs one extra read per click, in return for never acting on an old stage.

**The merge from `df6b5ea` was taken for the stage table columns.** Another branch changed the grinding table to show the product level values and added cutting and tempering columns. Those replaced the earlier grinding edge columns from this branch.

## Bugs found while testing (already there before, not fixed here)

**1. Print label barcode scanning is still unconfirmed.** A scanned label should open the piece. The decoder reads Code 128 correctly in a test, and the scan dialog now opens the mobile page on a phone. A small fix that draws the barcode image on a white background (so a PDF viewer does not show it as a black block) **is not in this branch**, so printed labels may still have a transparent barcode. Dense 25 character codes on small labels may also be hard for a phone camera. Nobody has confirmed scanning on a real device.

**2. Lint errors that were there before.** `src/lib/pdf/proforma-invoice-pdf.ts` has several `any` errors. The new `m.*.tsx` route files get the same `react-refresh/only-export-components` error as the existing mobile routes (already listed in `PWA.md`).

## Notes

- The Employee menu and rights need migration 062 and seed 019 from the backend PR to be run first. Until then the Employee page will not load for roles that do not have the right.
- `routeTree.gen.ts` is generated and ignored by git. It is rebuilt when the dev server starts or when `vite build` runs.
- The web app and the installed app are both affected by the sideways overflow fix, because the card grid is shared.

## Next steps

- Test on real phones: iPhone and Android, widths from 320 to 430, for the PI Status, Final Invoice and Dispatch lists.
- Test the stage dialog with a phone and a desktop on the same piece, in both directions.
- Print a label, scan it from a phone, and decide on the white background and label density fixes if it fails.

## Test plan

**Automated tests (no test files added or changed in this PR):**
- [x] `npx tsc -b`: passes.
- [x] `npx eslint src/features/pwa src/features/production-scan`: passes. The route files above and `proforma-invoice-pdf.ts` have the older errors listed.
- [x] `npx vite build`: builds.
- [ ] No component or unit tests exist for the new screens. They were not added.

**Manual checks (local):**
- [ ] Mobile Final Invoice, Dispatch and Employee screens in a browser or on a phone - not checked. Only type check, lint and build were run.
- [ ] Narrow phone (320 to 430 px) PI Status page after the overflow fix - not checked on a device.
- [ ] Stage dialog with a phone and a desktop on the same piece - not checked. Covered only by reading the code and by the backend refusing stale calls.
- [ ] Barcode label scan from a phone camera - not checked, see bug 1.
