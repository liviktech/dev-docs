# 2026-10-26 - shift-master-and-plan-shift (backend)

## Summary

There was no Shift master, and production plan lines were planned with a start date and an end date, with no way to say which shift the work was for. This PR adds a Shift master (same shape as Charges: list with Active, Inactive and All, search, KPI counts, create, edit, deactivate, delete) with a start time and an end time. It also changes each production plan line to one planned date plus a shift, in place of the start and end dates.

**Running the migrations is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/087_shift_master.sql` (new) | Creates `master.shift` (company, code, name, is_active, timestamps). Primary key is (company_id, code) and the name is unique per company, ignoring case. |
| `src/database/migrations/088_shift_start_end_time.sql` (new) | Adds `start_time` and `end_time` (TIME, NOT NULL) to `master.shift`. Existing rows get 09:00 and 17:00, then the defaults are dropped so a new shift must give both. A separate file because 087 was already applied on the dev database. |
| `src/database/migrations/089_production_plan_line_date_shift.sql` (new) | Adds `planned_date` and `shift_code` to `production_plan_line`. Copies the old `planned_start` into `planned_date`. Drops `planned_start`, `planned_end` and the date range check. |
| `src/modules/master/shift/` (new, 6 files) | Controller, repository, routes, schema, service and types for the Shift master, cloned from Charges. The schema checks start and end time as `HH:MM`. Required on create, optional on update. |
| `src/modules/master/master.routes.ts` | Mounts the shift router at `/master/shifts`. |
| `src/modules/production-plan/production-plan.schema.ts` | Line fields `plannedStart` and `plannedEnd` become `plannedDate` and `shiftId` (the shift code). |
| `src/modules/production-plan/production-plan.types.ts` | Row and DTO types follow the new fields and add `shiftName`. |
| `src/modules/production-plan/production-plan.repository.ts` | Reads `planned_date`, `shift_code` and the shift name (joined from `master.shift`). Saves the new columns. |
| `src/modules/production-plan/production-plan.service.ts` | Returns `plannedDate`, `shiftId` and `shiftName`. Removed the "end date before start date" check because the range is gone. |

## Why

Plans are made per day and per shift, so a date range does not fit how the work is scheduled. The shift list has to be a master so it can be managed in one place and reused by other screens.

## How

**Shift master**: works like Charges. The API id is the shift code, rows are soft deactivated with `is_active`, and the list returns items, totals and active and inactive counts when paged. A plain `GET /master/shifts` (no page) returns the full list, which the plan screen uses for its dropdown.

**Times**: stored as `TIME` and sent back as `HH:MM` using `to_char`. An end time earlier than the start time is allowed on purpose, for a shift that crosses midnight.

**Plan lines**: `shift_code` has no foreign key, the same as `work_center_code`. The shift name is joined at read time, and a line keeps its shift code even if the shift is deactivated later. Both new line fields are optional.

**Kept the same on purpose**: the plan submit, approve and reject flow, and the owner rules, did not change.

## Decisions made

**Times went into new migration 088, not into 087.** 087 was already applied on the dev database, so editing it would not reach that database.

**Old plan lines keep their start date.** The old end date is dropped and not kept anywhere. A date range cannot be mapped to one date and one shift any other way, so existing lines have no shift until someone picks one.

**Duplicate and constraint errors are mapped to clear messages.** The constraint names `uq_shift_company_code`, `shift_pkey` and `uq_shift_company_name` give the same style of message as Charges.

## Test plan

**Automated tests (no test files for this area, type check only):**
- [x] Backend `tsc --noEmit`: passes with no errors (run after the shift changes and again after the plan line changes).

**Manual checks (local):**
- [ ] Migrations 088 and 089, the shift API and saving a plan with a date and shift were not run, because the migrations were not applied. Type checking is the only thing that covers them for now.
