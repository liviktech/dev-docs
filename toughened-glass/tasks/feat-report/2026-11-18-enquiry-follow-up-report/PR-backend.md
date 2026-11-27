# 2026-11-18 - enquiry-follow-up-report (backend)

## Summary

The Date of Confirmation on an enquiry had no way to be followed up. This PR adds two read only endpoints: one for the notification bell (a short list with counts) and one for a new Enquiry Follow-up report (filtered and paged, and downloaded by the frontend). Both list only OPEN enquiries that have a Date of Confirmation, and both judge Overdue, Due Today and Upcoming against today in India time (`Asia/Kolkata`). No tables or columns were added and no migration is needed.

The type check passes. The endpoints were not called from a browser or a REST client for this change, and the SQL was not run against live data.

## What changed

| File | Change |
|---|---|
| `src/modules/enquiry/enquiry.routes.ts` | New `GET /enquiries/follow-ups` with `requirePermission("enquiry", "VIEW")`. It is registered before `/:id` so it is not read as an enquiry code. |
| `src/modules/enquiry/enquiry.controller.ts` | New `listFollowUps` handler. |
| `src/modules/enquiry/enquiry.service.ts` | New `listFollowUps`. Counts the rows per status and maps them to the response shape. The list is capped at 200 rows. |
| `src/modules/enquiry/enquiry.repository.ts` | New `findEnquiryFollowUps` query. Overdue first (oldest first), then due today, then upcoming (soonest first). |
| `src/modules/enquiry/enquiry.types.ts` | New `EnquiryFollowUpStatus`, `EnquiryFollowUpRow`, `EnquiryFollowUp` and `EnquiryFollowUps` types. |
| `src/modules/report/enquiry-follow-up/enquiry-follow-up-report.schema.ts` (new) | Zod query schema: `fromDate`, `toDate`, `partyId`, `salesRepId`, `followUpStatus` and paging. `fromDate` must not be after `toDate`. |
| `src/modules/report/enquiry-follow-up/enquiry-follow-up-report.repository.ts` (new) | The report query and its filters, paged with the shared `pagingSql`. |
| `src/modules/report/enquiry-follow-up/enquiry-follow-up-report.service.ts` (new) | Maps rows to the response: enquiry code, enquiry date, customer name, contact number, sales rep name, confirmation date and follow-up status. |
| `src/modules/report/enquiry-follow-up/enquiry-follow-up-report.controller.ts` (new) | The `getEnquiryFollowUpReport` handler. |
| `src/modules/report/report.routes.ts` | Mounts `GET /reports/enquiry-follow-up` with `requirePermission("reports", "VIEW")` and the query validation. |

## Why

Sales needs to see which open enquiries are due for a call today or are already late. The bell gives a quick list, and the report gives the full list with filters and downloads. Both use the same rule for the status so the two screens do not disagree.

## How

**One rule for the status**: The status is worked out in SQL against `(now() AT TIME ZONE 'Asia/Kolkata')::date`. A confirmation date before today is Overdue, equal to today is Due Today, and after today is Upcoming. It does not depend on the server or browser time zone.

**Who is listed**: Only enquiries with status `OPEN` and a confirmation date. Closed, approved and converted enquiries have moved on from the customer follow-up, and an enquiry with no date has nothing scheduled.

**Report filters**: Start and End Date apply to the Date of Confirmation, not the enquiry date. Party and Sales Rep match the enquiry's party code and sales rep code. The follow-up status filter uses the same CASE as the status column. Every query is limited to the company of the signed in user.

**Contact number**: The party phone, or the party WhatsApp number when the phone is blank.

**Sales rep name**: The sales rep master name, or the app user's full name when the enquiry has no sales rep master row.

**Paging**: The report uses the shared `pagingShape` and `pagingSql`. Without paging params it returns every matching row, which is what the frontend downloads use.

## Decisions made

**The bell and the report are two endpoints.** The bell needs counts and a short capped list for every user who can view enquiries. The report needs paging and filters and sits behind the Reports permission. Sharing one endpoint would have mixed both permission checks and both response shapes.

**The bell list is capped at 200.** Overdue rows come first, so they are never the ones cut. The counts are taken from the returned rows, so on a very large list they would stop at 200. This was accepted because a bell is meant to be short.

## Test plan

**Automated tests (not added):**
- [x] `npx tsc -b`: passes, no errors.
- [ ] No unit or integration tests were added for the new endpoints.

**Manual checks (local):**
- [ ] `GET /enquiries/follow-ups` and `GET /reports/enquiry-follow-up` were not called, so the SQL was not run against real data. The status split around midnight India time was not tested.
