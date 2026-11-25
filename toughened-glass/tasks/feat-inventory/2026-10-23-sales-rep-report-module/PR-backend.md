# 2026-10-23 - sales-rep-report-module (backend)

## Summary

The app had no Reports area. This PR adds the Sales Representative report to the backend as a new `report` module. For one sales rep, or for every rep when none is chosen, it returns the enquiry counts by status, how many order bookings, work orders and final invoices came from those enquiries (and the status of each), enquiries per rep, and an enquiry trend grouped by day, week, month or year. It also has an unpaged export for CSV and PDF, and it registers a `reports` module and a `reports_view` right so the page can be given to roles from Roles & Access.

The numbers, filters, grouping, bad input and the logged out case were checked live against the running API. Response time was measured with `curl`. A role that has only the Reports right (not admin) was **not checked**, only the admin user was used.

## What changed

| File | Change |
|---|---|
| `src/modules/report/report.routes.ts` (new) | `reportRouter` with one `requirePermission("reports", "VIEW")` guard. Mounts the sales rep routes: `/sales-reps`, `/` and `/export`. |
| `src/modules/report/sales-rep/sales-rep-report.types.ts` (new) | Types for the report row, summary (including the status breakdowns, by-rep list and trend), page and filters. |
| `src/modules/report/sales-rep/sales-rep-report.schema.ts` (new) | Zod checks for the query string. The sales rep is optional (empty means every rep). Dates must be valid and in order, `status` must be one of the four report statuses, `groupBy` is `day`, `week`, `month` or `year` (default `month`), `summaryOnly` is `"true"` or `"false"`, and `limit` is capped at 200. |
| `src/modules/report/sales-rep/sales-rep-report.repository.ts` (new) | The SQL. `getSummary` is one statement that reads the enquiries once and hangs the bookings, work orders and invoices off them, returning every part as a JSON list. `listRows` returns the rows and the total for the table and the export. |
| `src/modules/report/sales-rep/sales-rep-report.service.ts` (new) | Checks the rep exists when one is given. With `summaryOnly=true` it returns the summary only and skips the row list. |
| `src/modules/report/sales-rep/sales-rep-report.controller.ts` (new) | Thin handlers for the three routes. |
| `src/api/route-mapper.ts` | Added `apiRouter.use("/reports", reportRouter)`. |
| `src/database/seeds/027_reports_rbac.sql` (new) | Seed that adds the `reports` module to `core.module` and the `reports_view` right to `core.right`. Both inserts use `ON CONFLICT`, so it is safe to run again. |

## Why

The ask was a scalable Report module, Sales Representative report first: summary numbers, the order booking and final invoice flow, filters, and CSV and PDF export. The easy part was the status counts. The real work was following an enquiry to its order bookings, work orders and final invoices (these tables now link by code, not by id), and then making the page switch quickly, because the database is hosted far away.

## How

**Status mapping**: Enquiry status is mapped to the report status in SQL. OPEN is Pending, APPROVED is In Progress, CONVERTED is Completed, CLOSED is Cancelled. For a converted enquiry the completed date is its `updated_at`. "Completed" here means the enquiry was converted to an order, not that the work is finished or delivered.

**Linking the documents**: Enquiry to order booking is by `enquiry_code`. Order booking to work order is by `order_code`. Work order to final invoice is by `work_order_id`. Bookings, work orders and invoices are each counted by their own status.

**Common report**: With no `salesRepCode` the report covers every rep, and the by-rep list is filled (enquiries with no rep show as "Unassigned"). With a rep selected the by-rep list is empty.

**Summary vs rows**: The summary ignores the status filter, so the charts always show the full picture for the rep and date range. The status filter only narrows the rows.

**One query**: The summary used to be about ten queries one after another. Because each one costs a network round trip, it took about 2.2 seconds. It is now one statement, about 0.75 to 1 second. About half a second of that is the shared login and permission check, which is **not changed**.

**Trend grouping**: `groupBy` is checked against a fixed list before it goes into the SQL, so it cannot carry anything else. The bucket label is `YYYY-MM-DD` for day and week (a week shows its Monday), `YYYY-MM` for month and `YYYY` for year.

**Export**: `/export` runs the same filters without paging so the CSV and PDF carry every row.

**Access**: One `reports` module with a `VIEW` right covers opening and exporting. Admin and super admin bypass the check, so they need no rows. Other roles get the right from Roles & Access.

**Seed, not migration**: The `027` file is a seed because it only inserts rows. It creates no tables. Run it with `npm run seed`.

## Decisions made

**Overdue was removed.** An early version had an Overdue bucket. It was dropped everywhere (backend, API shape, UI, PDF) because it was not wanted.

**One statement instead of several parallel ones.** Running the queries together in parallel was tried first and only got to about 1.2 seconds. Merging them into one statement was faster again.

**`summaryOnly` flag.** The page now shows charts and no row list, so it asks for the summary only. The row list stays for the export and for any later screen that needs it.

**Seed instead of a migration.** Nothing here changes the table structure, so there is no migration file.

## Notes

- Sample data: reps "Bavin" (14 enquiries) and "Arun Kumar" (12), plus 36 enquiries with no rep. The whole company shows 73 enquiries, 61 converted, 57 order bookings, 33 work orders and 18 final invoices. Arun Kumar shows 12, 10, 10, 4 and 2.
- The seed is recorded in `public.schema_seeds`. Deleting the file after it has run does nothing to the database. Deleting the rows would remove Reports from Roles & Access and block non-admin roles.
- Checked with a test server on port 5199, started separately from any dev server.

## Next steps

- Check the Reports right with a non-admin role.
- If the page still feels slow, look at the shared permission check, which adds about half a second to every request.
- Add the next reports under `src/modules/report/` using the same folder pattern.

## Test plan

**Automated tests:**
- [ ] No automated tests were added for this module.
- [x] `npx tsc --noEmit`: passes.

**Manual checks (local):**
- [x] Whole company and one rep: counts, booking, work order and invoice status lists, by-rep list and trend all returned. The status lists add up to the totals (57, 33 and 18).
- [x] Same numbers before and after the one-statement rewrite, for the whole company and for Arun Kumar.
- [x] `groupBy` day, week, month and year: each returned the right bucket labels and the totals stayed at 73 and 61. A bad value was rejected with a validation error.
- [x] `summaryOnly=true`: returned the summary with an empty row list. Response time about 0.75 to 1 second, down from about 2.2.
- [x] Status filter: rows narrowed, summary counts stayed the same. Missing and unknown rep, reversed dates and a bad status each returned a clear error. No token returned 401.
- [x] Search, including a SQL injection style string: returned normal results and no error.
- [ ] A non-admin role with only the Reports right - not checked, only the admin user was used.
