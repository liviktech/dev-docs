# 2026-10-15 - pwa-attendance-invoice-dispatch (backend)

## Summary

The app had no way to record employees, daily attendance or attendance based payroll. Separately, completing the last stage of a piece returned a 500 (Internal server error) on the dev database, and a piece page that was out of date on one device (phone or desktop) could start or complete a stage that the other device had already finished.

This PR adds a new `attendance` module (employees, daily attendance, payroll), makes the Final Invoice auto create step safe when the database has no unique constraint on `work_order_id`, and makes the production scan Start and Complete calls tell a stale caller that the stage was already handled instead of acting on the wrong stage.

The 500 fix was checked live against the dev database with a script that was rolled back. The payroll and attendance SQL was checked in an in memory Postgres (pglite). **Migrations 062 and 063 and seed 019 were not run against the shared dev database by me** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/062_employee_attendance.sql` (new) | Creates the `hr` schema with `hr.employee`, `hr.attendance` and `hr.payroll_record`. Unique keys: employee code per company, one attendance row per employee/date/status, one payroll record per employee/month/year. |
| `src/database/migrations/063_final_invoice_work_order_unique.sql` (new) | Adds `uq_final_invoice_work_order` on `invoice.final_invoice(work_order_id)` if it is missing. Later changed by PR #96, see "Notes". |
| `src/database/seeds/019_attendance_rbac.sql` (new) | Adds the `attendance` module with VIEW and CONTROL rights so it shows in Roles & Access and the menu. |
| `src/modules/attendance/attendance.routes.ts` (new) | Routes under `/attendance`: employees (list, create, update, toggle status), daily attendance (get, save), payroll (get, save, change record status). VIEW for reads, CONTROL for writes and for payroll. |
| `src/modules/attendance/attendance.schema.ts` (new) | Zod validation for the bodies and query strings. |
| `src/modules/attendance/attendance.types.ts` (new) | Types for employees, attendance rows and payroll rows. |
| `src/modules/attendance/attendance.controller.ts` (new) | Thin handlers that call the service. |
| `src/modules/attendance/attendance.service.ts` (new) | Business rules: payroll maths, locking of finalized records, validation of status combinations. |
| `src/modules/attendance/attendance.repository.ts` (new) | All SQL. Only the columns each screen needs are selected. |
| `src/api/route-mapper.ts` | Mounts `attendanceRouter` at `/attendance`. |
| `src/modules/final-invoice/final-invoice.repository.ts` | The auto create insert no longer uses `ON CONFLICT (work_order_id)`. It uses `INSERT ... SELECT ... WHERE NOT EXISTS (...)` instead. |
| `src/modules/production-scan/production-scan.service.ts` | `scanWorkOrder` and `completeScanStage` accept the stage the caller is looking at. If that stage already moved on, nothing is changed and the response carries `alreadyCompleted: true`. A finished work order no longer throws a 400 for a caller that names a stage. |
| `src/modules/production-scan/production-scan.schema.ts` | The scan body accepts an optional `stageId` (uuid). |
| `src/modules/production-scan/production-scan.controller.ts` | Passes `stageId` from the body to the service. |

## Why

The business asked for employee and attendance tracking with payroll, so the backend needed its own data model and API. The 500 on the last stage was a real blocker for production: finishing the last stage creates the Final Invoice, and that insert used `ON CONFLICT (work_order_id)`, which fails with Postgres error 42P10 when the unique constraint does not exist on that database. The stale page problem came from phone and desktop users working on the same piece: the second user could complete or start a stage that was already done.

## How

**Attendance data model.** One row per employee, date and status. Day plus Night on the same date is a double shift. Half Day, Absent and Company Holiday are exclusive and cannot be mixed with anything else.

**Saving attendance.** The daily save sends only the rows that changed. The repository deletes the old statuses and inserts the new ones for those employees in one go using `unnest`, so the call is one round trip no matter how many employees changed.

**Employee codes.** Codes like `EMP-001` are generated inside a transaction with `pg_advisory_xact_lock` so two people adding an employee at the same time do not get the same code.

**Payroll.** One SQL query counts the attendance for the month (CTE plus `generate_series`). Net pay is base salary, minus loss of pay, plus the double shift bonus, rounded to whole rupees. Days before the joining date are not counted as loss of pay. `getPayroll` returns the saved records if the month was already saved, otherwise a live calculation. Once any record in a month is marked PAID, the saved records for that month are locked.

**Stale stage protection.** `scanWorkOrder` and `completeScanStage` compare the stage the caller named with the stage that is really current. If they differ, or the work order is already finished, the call returns the current state with `alreadyCompleted: true` and changes nothing. Callers that do not send `stageId` behave exactly as before.

**Final Invoice insert.** The `WHERE NOT EXISTS` check gives the same "do not create twice" result without needing the constraint, so it works on any database.

## Decisions made

**Did not rely on migration 063 to fix the 500.** The code now works with or without the constraint. Running a migration on a shared database is a separate decision, and the app should not break because a migration was missed.

**Stale callers get a normal success response with a flag, not an error.** A 400 would show up as a red error toast on the phone. The clients show a confirmation message instead, and the current state comes back in the same response, so they do not need a second request.

**No `search` parameter on the daily attendance call.** The screen already has the whole day in memory, so filtering is done in the browser and typing does not send requests.

## Bugs found while testing (already there before, not fixed here)

**1. Final Invoice 500 on the last stage.** The dev database was missing the unique constraint that the old insert depended on. This PR avoids the dependency but does not add the constraint to that database.

**2. Two migrations are numbered 062.** `062_employee_attendance.sql` and `062_final_invoice_items.sql` now exist side by side after merging branches. Whether both run in order depends on the migration runner. Nobody should rename them without checking what has already run on each environment.

## Notes

- Migration 063 adds the unique constraint on `work_order_id`, and migration 064 (`064_final_invoice_work_order_not_unique.sql`, from PR #96 for manual partial invoices) removes it again, because one work order can now have several Final Invoices. That is why the insert in this PR does not use `ON CONFLICT`. If someone reads 063 on its own it can look wrong.
- The shared dev database (Neon) has not had `npm run migrate` or `npm run seed` run for 062, 063 and seed 019 by me. The Employee menu and rights will not appear until they are run.
- The 500 was reproduced and the fix was checked with a script against the real dev database that was rolled back, so no data was written. The payroll and attendance counts (double shift, half day, absent, mid month joiner, saving twice, code generation) were checked in pglite.

## Next steps

- Run `npm run migrate` and `npm run seed` on the dev database and tick the new Attendance rights for the roles that need them.
- Decide whether duplicate migration number 062 should be cleaned up.

## Test plan

**Automated tests (no test files added or changed in this PR):**
- [x] Type check (`tsc`) on the backend: passed when the attendance code was written.
- [ ] No automated tests exist for the attendance module or the stale stage flow. They were not added in this PR.

**Manual checks (local):**
- [x] Payroll and attendance SQL in pglite: double shift counts as 2, half day as 0.5, absent and holiday as expected, a joiner mid month has no loss of pay before the joining date, saving the same day twice replaces the rows, and employee codes count up without gaps.
- [x] Completing the last stage on the dev database through a rolled back script: the old insert failed with 42P10, the new insert created the Final Invoice once and did nothing on a second run.
- [ ] Stale Start and Complete from two real devices - not checked here. Covered only by reading the code paths and by the frontend checks in the frontend PR.
- [ ] Attendance API against the real dev database - not checked, because the migration has not been run there.
