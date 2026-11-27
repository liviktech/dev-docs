# 2026-11-24 - rep-enquiry-view-modals (backend)

## Summary

Merging `dev` into this branch left two merge conflicts in the order booking list query. Both sides added new columns to the same select list and to the `GROUP BY`. This PR resolves them by keeping both sets of columns, so the table still returns Created By (from this branch) and Manager Approved By (from `dev`).

There is no other backend change in this task. I did not run the backend or its tests after the merge, so the query has not been run against the database.

## What changed

| File | Change |
|---|---|
| `src/modules/order-booking/order-booking.repository.ts` | Two conflicts fixed in the paged booking list query. In the select list, `created_at` and `created_by_name` now sit beside `manager_approved_at` and `manager_approved_by_name`. In the `GROUP BY`, `created_at`, `created_by`, `manager_approved_at` and `manager_approved_by` are all listed. |

## Why

The select list and the `GROUP BY` have to match. A column in the select that is missing from the `GROUP BY` makes Postgres reject the query. The service and types already read both pairs of fields (`createdByName` and `managerApprovedByName`), so keeping only one side would have broken the other feature.

## How

**Both sides kept**: nothing was dropped from either branch. The two sets of columns do not overlap, so the merge was a union.

**Kept the same on purpose**: the filters, the paging, the totals and the row shape are untouched.

## Notes

- A search of `src` shows no conflict markers left.
- The duplicate username login problem is in the backend (`src/modules/auth/auth.repository.ts`, `findUserByUsername`). It is described in the frontend PR and is not changed here.

## Test plan

**Automated tests (not run):**
- [ ] Backend build and tests were not run after the merge. The table query was not run against the database either, so this is only checked by reading the select list against the `GROUP BY`.

**Manual checks (local):**
- [ ] Open the Order Booking table and check it loads with both Created By and the manager approval fields. Not done.
