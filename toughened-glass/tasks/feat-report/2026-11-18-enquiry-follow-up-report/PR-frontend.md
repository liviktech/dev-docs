# 2026-11-18 - enquiry-follow-up-report (frontend)

## Summary

Enquiries had a Date of Confirmation but nothing that reminded anyone to follow up on it. This PR adds a notification bell for follow-ups (desktop Enquiries page and the mobile Enquiries screen) and a new Enquiry Follow-up report (desktop and mobile app). It also makes small changes to the FIR report and the Raw Glass reports, and removes the Party Outstanding banner from the Proforma Invoice Details modal.

The type check and Prettier pass. None of the screens were opened in a browser for this change, so nothing here was checked visually.

**A formatting only rewrite of `enquiries-page.tsx` is included, not a real change** - see "Notes" below.

## What changed

| File | Change |
|---|---|
| `src/features/enquiries/api.ts` | New `EnquiryFollowUp`, `EnquiryFollowUps` and `EnquiryFollowUpStatus` types and a `getEnquiryFollowUps` call to `/enquiries/follow-ups`. |
| `src/features/enquiries/queries.ts` | New `enquiryFollowUpsQueryOptions`. It sits under the `enquiries` key, so saving or closing an enquiry refreshes it, and it is read again every 5 minutes. |
| `src/features/enquiries/enquiry-follow-up-bell.tsx` (new) | The bell. A badge counts Overdue and Due Today (Upcoming is listed but not counted). The panel lists each follow-up with a status label and opens that enquiry on click. Shown only to users who can view enquiries. |
| `src/features/enquiries/enquiries-page.tsx` | Shows the bell, and opens the view dialog for the enquiry code in the new `view` URL param. The rest of the diff is formatting only (see Notes). |
| `src/routes/enquiries.tsx` | Adds `view` to the allowed search params, so the bell can link to an enquiry. |
| `src/features/pwa/enquiry/mobile-enquiry-list.tsx` | Shows the bell in the mobile top bar, next to the refresh button. |
| `src/features/reports/enquiry-follow-up/api.ts` (new) | Types, status options and labels, the status to tone map (`FOLLOW_UP_TONE`) shared with the bell, and `fetchEnquiryFollowUpReport`. |
| `src/features/reports/enquiry-follow-up/enquiry-follow-up-report.tsx` (new) | The desktop report. Date range, Party, Sales Rep and Follow-up filters, reset, XLSX and PDF export, and server paging. |
| `src/features/pwa/reports/enquiry-follow-up-report.tsx` (new) | The mobile version, built on the shared `MobileTableReport`. |
| `src/features/reports/report-menu.ts`, `src/features/reports/registry.ts`, `src/features/pwa/reports/registry.ts` | Register the new "Enquiry" group with the "Enquiry Follow-up" report on desktop and mobile. |
| `src/features/reports/fir/fir-report.tsx` | Search bar removed. Party names use `formatPartyName`. The Rework and Replacement counts show as two KPI cards (the same `KpiCard` as the Enquiries page) between the filters and the table, sharing the full width. |
| `src/features/pwa/reports/sales-rep-reports.tsx` | The mobile FIR report Party column uses `formatPartyName`. |
| `src/features/reports/inventory/batch-consumption-report.tsx`, `src/features/reports/inventory/offcut-register-report.tsx` | Search bar removed, and the "Search:" line removed from the PDF. The `search` filter is still in the state and API, only the input is gone. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | The red "Party Outstanding" banner is removed from the Proforma Invoice Details modal, along with its query and the now unused imports. |

## Why

Sales need to see the open enquiries whose Date of Confirmation is late or due today. The bell gives that at a glance and the report gives the full list with filters and downloads. The FIR and Raw Glass changes follow review feedback on the report screens.

## How

**One status style**: The report and the bell both use the shared `StatusLabel` with the same `FOLLOW_UP_TONE` map (Overdue is rejected, Due Today is pending, Upcoming is open). There is no separate badge component.

**Open enquiries only**: The report shows only open enquiries, and the Start and End Date filters apply to the Date of Confirmation. The follow-up status is worked out by the server, so the screen never compares dates itself.

**Lookups load on demand**: The Party and Sales Rep dropdowns load their lists when first used, or at once if one is already selected so its name can be shown.

**Counts are not hardcoded**: The FIR KPI cards show the counts the server returns for the current filters. While loading they show "...".

**Left the same on purpose**: The Raw Glass search filter is still in the state and API. The FIR PDF still prints the Rework, Replacement and Total Processed line. The Party Outstanding banner in the Admin Approval modal and in the create Proforma Invoice screen was not touched.

## Decisions made

**The Party Outstanding banner was removed, not hidden.** The request was to remove it from the view modal. The query that fed it was removed too, so the modal no longer makes that call.

**The FIR counts went through several layouts.** Tabs, a footer strip and counts inside the filter bar were tried and dropped. The final layout is two KPI cards between the filters and the table.

## Notes

- `src/features/enquiries/enquiries-page.tsx` shows a large diff because Prettier rewrote the quote and semicolon style of the whole file. The real change is the bell, the `view` param and its import. Review that part only.
- Sending the enquiry queue content to customers (WhatsApp or SMS) is not part of this PR. Nothing was built for it.

## Next steps

- Open the new report, the bell and the FIR KPI cards in a browser, on desktop and in the mobile app.
- Settle the open questions for customer notifications (when PI ready is sent, SMS provider, template approval) before that work starts.

## Test plan

**Automated tests (not added):**
- [x] `npx tsc -b`: passes, no errors.
- [x] `prettier --check` on the new and changed report files: passes.
- [ ] No component tests were added for the bell or the report.

**Manual checks (local):**
- [ ] Bell (desktop and mobile), Enquiry Follow-up report (desktop and mobile), XLSX and PDF export, the FIR KPI cards and the Proforma Invoice Details modal were not opened in a browser, so nothing was seen on screen.
