# 2026-11-25 - direct-sales-rep-removal (backend)

## Summary

"Direct" used to be a built-in row in the Sales Representatives master (code `DIRECT`, marked as a system row), and the code had to keep creating it for every company. This PR removes that row. Direct now simply means an enquiry with no sales rep (`sales_rep_code` is NULL). Reports and dashboards still show those enquiries as "Direct", but they build that line themselves instead of reading a master row. The PR also adds a rule that an enquiry from a Visit cannot be Direct.

The type check passes. The migration was written but not run, and the Visit rule was not tried against a running server (see "Test plan").

**Old data is not handled on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/183_remove_direct_sales_rep_row.sql` (new) | Sets enquiries saved under `DIRECT` to no rep, clears their `visit_code`, deletes any visits and visit days recorded against `DIRECT`, deletes the `DIRECT` row from `master.sales_rep`, and drops the `is_system` column. Not run. |
| `src/modules/master/sales-rep/sales-rep.repository.ts` | `DIRECT_SALES_REP_CODE` and `ensureDirectSalesRep` removed. The list no longer filters on `is_system`. `is_system` is gone from the selects, and the re-number query no longer skips a system row. |
| `src/modules/master/sales-rep/sales-rep.service.ts` | `assertNotSystem` and its three calls removed (delete, deactivate, recode). `isSystem` removed from the response. |
| `src/modules/master/sales-rep/sales-rep.types.ts` | `is_system` and `isSystem` removed. |
| `src/modules/enquiry/enquiry.repository.ts` | Create and update no longer ensure a Direct row. A new `toRepCode` helper turns a blank or `DIRECT` rep into NULL before saving. The rep name falls back to "Direct" in the list, detail and follow-up queries. `listSalesReps` returns real reps only, and the fallback to app users is removed. |
| `src/modules/enquiry/enquiry.service.ts` | New `assertVisitSourceHasRep`. Create and update reject a Visit enquiry with no rep or with `DIRECT`, with `SALES_REP_REQUIRED_FOR_VISIT`. On update it checks the final values (the new ones, or the saved ones if the field was not sent). |
| `src/modules/dashboard/enquiry/dashboard-enquiry.repository.ts` | The `DIRECT` filter becomes `e.sales_rep_code IS NULL`. The rep label falls back to "Direct" instead of "Unassigned". |
| `src/modules/dashboard/pi/dashboard-pi.repository.ts` | The `DIRECT` filter becomes `e.sales_rep_code IS NULL`. |
| `src/modules/report/enquiry-follow-up/enquiry-follow-up-report.repository.ts` | The `DIRECT` filter becomes `IS NULL`. The rep name falls back to "Direct". |
| `src/modules/report/sales-rep/sales-rep-report.repository.ts`, `src/modules/report/sales-rep/sales-rep-report.service.ts` | New exported `DIRECT_REP_CODE`. The `DIRECT` filter becomes `IS NULL`. The rep options list ends with a built-in "Direct" entry, and the service skips its rep-exists check for it. The "Unassigned" label becomes "Direct". |
| `src/modules/report/sales-rep-targets/sales-rep-targets-report.repository.ts` | The rep query no longer reads `is_system`. The report adds the Direct line itself, last, with no targets. |

## Why

Direct was never a real person. Keeping it as a master row meant it showed up in the master list with special rules (cannot delete, deactivate or recode), and every enquiry create had to make sure the row existed. Treating "no rep" as NULL removes all of that. The easy part was the master and enquiry code. The real work was every report and dashboard that matched on the `DIRECT` code, which would have returned nothing once the row was gone.

## How

**Direct is NULL**: An enquiry with no rep keeps `sales_rep_code = NULL`. The API still accepts `DIRECT` or a blank value for the rep and stores NULL for both, so older clients keep working.

**Filters**: Wherever a report or dashboard takes a rep filter, the value `DIRECT` is turned into `sales_rep_code IS NULL`. Any other value is matched as before.

**Names**: Queries that show the rep name use `COALESCE(sr.name, rep.full_name, 'Direct')`, so the name is "Direct" when there is no rep.

**Visit rule**: `assertVisitSourceHasRep` runs first in create and before the visit check in update. The source is compared case-insensitively.

**Kept the same on purpose**: Enquiry and report response shapes, and the sales rep list endpoints and their routes. Only the `isSystem` field is gone from the sales rep response.

## Decisions made

**Old data is not handled beyond the migration.** The request was to not think about old data. The migration only moves enquiries off the `DIRECT` row so nothing points at a row that no longer exists. It does not check other tables for a `DIRECT` rep code.

**Visit with Direct is an error, not a silent fallback.** The rule rejects the save so the user picks a real rep. It does not pick one for them.

**The migration deletes visits recorded against Direct.** Direct cannot make visits any more, so any that exist are removed instead of being left pointing at a missing rep.

## Notes

- Migration 183 was written only. It has to be run by the person who owns the database. Deploy it with this change, since the code no longer creates the Direct row.
- The migration clears `visit_code` on enquiries that came from a Direct visit, so those enquiries lose their link to the visit.
- Dropping `is_system` cannot be undone by this migration.

## Next steps

- Run migration 183 on a copy of the data first and check the enquiry and visit counts before and after.

## Test plan

**Automated tests (not added):**
- [x] `npx tsc --noEmit`: passes, no errors.
- [ ] No tests were added for the Visit rule or the `IS NULL` filters.

**Manual checks (local):**
- [ ] Migration 183 was not run, so the SQL was not tried against a database.
- [ ] The Visit rule was not tried against the running backend with a real request. The frontend blocks the same case before it is sent.
- [ ] The reports and dashboards were not opened with the `DIRECT` filter, so the `IS NULL` queries were not checked against data.
