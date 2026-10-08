# 2026-10-08 - work-order-and-list-polish (backend)

## Summary

Creating a company from the Super Admin screen returned a 500 error, and bad request bodies that failed validation also came back as a generic 500. This PR fixes the company clone so it works, and makes validation errors return a 400 with the field name.

I could not run the full company creation against the live database. The cause was found from the real error in the server log (`column "id" does not exist`). See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/modules/superadmin/superadmin.repository.ts` | `cloneMasterDataForCompany` no longer selects an `id` column. `master.work_center` and `master.charge` are keyed by `(company_id, code)`, so they are now copied by code (`code, name, is_active`). The old id maps were removed. |
| `src/middlewares/error.middleware.ts` | A `ZodError` now returns a 400 `VALIDATION_ERROR` with a `details` map and a message like `field: message`, instead of a generic 500. |
| `src/modules/order-booking/order-booking.repository.ts` | No real change. An experiment for the HOLE position was reverted to the original code. |

## Why

Both tables used by the clone have no `id` column, so the copy failed on the first query and the whole company creation rolled back with a 500. The validation change makes it clear what is wrong when a request body is rejected, which would also have helped find this sooner.

## How

**Clone by code**: the new company gets its own rows with the same `code`, `name` and `is_active` as the source company. Nothing else about the clone changed.

**Validation errors**: Express 5 forwards async errors to the error middleware, so a thrown `ZodError` reaches the new handler and is turned into a 400.

## Decisions made

**HOLE is not moved in the routing.** I first tried placing HOLE at a fixed position among the work centers, but it was a hard-coded list of codes, so it was reverted. The printed route hides HOLE in the frontend instead.

## Bugs found while testing (already there before, not fixed here)

**1. Deleting an approved PI is not blocked without a work order.** The backend only refuses the delete once a work order exists. The frontend now hides the button, but the API still allows it.

**2. Existing items still have HOLE last in their routing.** There was no migration, so only new items follow the current rule.

## Notes

- A repro script I wrote first did not select `id`, so it missed the real cause. The real error came from the server log.
- The `.env` was printed to the terminal once while checking the database settings, which exposed the database URL and password. The password should be rotated.

## Next steps

- Block deleting approved PIs in the API.
- Rotate the database password.

## Test plan

**Automated tests (no test files for these changes):**
- [x] `npx tsc --noEmit -p .`: passed after each change.

**Manual checks (local):**
- [ ] Create a company from Super Admin and check that the work centers and charges are copied. Not run live after the fix.
- [ ] Send a bad body to a validated route and check it returns a 400 with the field name. Not run live.
