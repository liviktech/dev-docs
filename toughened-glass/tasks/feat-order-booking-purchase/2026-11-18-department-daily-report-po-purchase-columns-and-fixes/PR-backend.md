# 2026-11-18 - department-daily-report-po-purchase-columns-and-fixes (backend)

## Summary

This PR is the work from one day. The main piece is a new **Department-wise Daily Report** under Reports. The rest is Purchase Order and Purchase screen work (a rename, new table columns, a clickable Purchase Order link, a new layout for amounts and charges) plus two fixes: a server error on Admin Approval and a section that should not show in Party Approval.

Before this PR there was no report showing how many Enquiries, Order Bookings, PIs, Work Orders, Final Invoices and Dispatches a day has. "Assign Supplier" was the name of the Purchase Order action that really records the order. The Purchase table could not show the Purchase Order behind a purchase or its additional charges and last payment date. The Indent and Purchase Order view modals left gaps in their General Details. Admin Approval failed with "Internal server error" when the promised delivery date was today. Party Approval showed Delivery Details & Surcharges, which only the admin needs.

Each change keeps the existing flows working. All changes were tested in the browser. Frontend and backend type checks pass apart from one error that was already there (see "Bugs found while testing"). The new report's query and the Admin Approval query were also run on a scratch local Postgres built from all project migrations.

**Purchase Return is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/report/department-daily/department-daily-report.schema.ts` (new) | Validates the `fromDate` and `toDate` filters. |
| `src/modules/report/department-daily/department-daily-report.repository.ts` (new) | One query that counts each department by status and date for the company. |
| `src/modules/report/department-daily/department-daily-report.service.ts` (new) | Builds the six rows and works out Pending as the rest. |
| `src/modules/report/department-daily/department-daily-report.controller.ts` (new) | Thin HTTP layer. |
| `src/modules/report/report.routes.ts` | New `/reports/department-daily` route with the same Reports view permission as the other reports. |
| `src/modules/procurement/procurement.repository.ts` | Purchase Order list query now returns `totalArea`. `findProcurementInfo` also returns every order of a purchase (listed once each); the old single-order fields are unchanged. |
| `src/modules/procurement/procurement.types.ts` | `totalArea` and `purchaseRequests` types. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | Fixed the same-day capacity query used by Admin Approval: it read a `charge_name` column that does not exist, so it now takes the name from `master.charge`. |

## Why

The Department-wise Daily Report is a new reporting need: a daily count per department, without the production stages, which keep their own reports. The Order Status rename, the new columns and the Purchase Order link came from review feedback on the Purchase and Purchase Order screens. The Admin Approval error was reported while testing, and the Party Approval change is because only the admin sets delivery details.

Most of the work was in the report: deciding which date and which statuses mean "completed" for each department, and checking them against how the app really sets those statuses. The Admin Approval fix was a one-line query fix, but finding it needed a reproduction.

## How

**Completed, Pending and Cancelled**: Completed and Cancelled come from a fixed list of statuses per department. Pending is everything else, so the three always add up to "Entries Created". Days with no records show 0.

**Admin Approval fix**: a booking charge stores only a `charge_code`, so the capacity query now joins `master.charge` for the name, the same join the order booking code already uses.

## Decisions made

**New "Department Daily" menu group.** There is already a "Department-Wise" report under Production Plan for production stages. A separate group avoids confusion with it.

**No pagination footer on the Department report.** It always returns six fixed rows, so the table shows them all on one page (`showAllRows`). The server also returns all six rows together, so there is no server paging.

**Same-day capacity rule left as is.** The server treats a promised date of today as same-day, even for Express. The fix only repairs the broken query. Changing the rule was not asked for.

**Purchase Return is not included.** It was planned but is on hold until the team decides whether received, accepted and rejected quantities belong per product or per purchase.

## Bugs found while testing (already there before, not fixed here)

**1. A migration comment says a user can close a PI, but the code does not.** The comment in `075_pi_close.sql` says a PI can be closed by a user. The code only sets Closed automatically, when the PI is complete. A user cancel sets Cancelled. This misled the first version of the report.

## Notes

- The new report's counts and the Admin Approval query were checked on a scratch local Postgres, built from all project migrations, with synthetic data. The data included records at IST midnight, every status of every department and a second company.
- Two read-only checks were made against the database in the backend `.env`: today's PI (it was Closed, closed automatically as completed) and the list of applied migrations (none missing). Nothing was written there.
- The same-day capacity query now runs. It was not tested with a PI that has machining charges on a scratch database.
- Deleted Order Booking, PI and Work Order records cannot appear in the report. Showing them would need a deletion log.

## Next steps

- Decide whether Express should skip the same-day capacity check, and whether the screen should warn before the server refuses.
- Revisit Purchase Return once the QA quantity question (per product or per purchase) is answered. A debit note is for another team member.
- Update the stale PI close comment in `075_pi_close.sql`.

## Test plan

**Automated tests:**
- [x] Backend type check (`npx tsc --noEmit`): no errors.
- [x] Department report query on a scratch Postgres: records at IST midnight land on the right day, a day with no records is all zeros, a second company never appears, and every status of every department lands in the intended column. Completed + Pending + Cancelled equals the total in every case.
- [x] Admin Approval capacity query on a scratch Postgres: failed with "column obc.charge_name does not exist" before the fix, runs after it.

**Manual checks (local):**
- [x] Admin Approval with a promised date of today: approves without the server error. Tested in the browser.
