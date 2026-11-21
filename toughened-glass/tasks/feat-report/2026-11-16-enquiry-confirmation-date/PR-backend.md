# 2026-11-16 - enquiry-confirmation-date (backend)

## Summary

An enquiry had no field for the date the customer is expected to confirm it. This PR adds a nullable Date of Confirmation to the enquiry: a new column, accepted on create and update, and returned when an enquiry is read. An enquiry saved without the date, or with the date cleared, keeps working and returns `null`. This was type checked only. The migration and the API were not run against a live database for this change.

**Follow-up notifications, the in-app follow-up list and the follow-up report are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/138_enquiry_confirmation_date.sql` (new) | Adds `confirmation_date DATE` (nullable) to `sales.enquiry`, plus a partial index on open enquiries that have a date. |
| `src/modules/enquiry/enquiry.schema.ts` | New optional, nullable `confirmationDate` (`YYYY-MM-DD`) on the create schema. The update schema is the partial of it, so it is accepted on update too. |
| `src/modules/enquiry/enquiry.repository.ts` | Create inserts `confirmation_date`. Update sets it only when the field is sent, so an edit that does not send it leaves the stored date alone, and `null` clears it. |
| `src/modules/enquiry/enquiry.service.ts` | Maps `confirmation_date` to `confirmationDate` in the enquiry response. |
| `src/modules/enquiry/enquiry.types.ts` | Adds `confirmation_date` to the row type and `confirmationDate` to the API type. |

## Why

The business wants to track when each customer is expected to confirm, so the sales team knows when to follow up. This PR is only the data part: store the date and return it. The date has to be reliable first, before anything uses it.

## How

**Same shape as delivery date**: the field follows the existing `delivery_date` field in every layer (validation, insert, update, response mapping). Nothing new was invented.

**Nullable end to end**: the column allows null, the schema allows `null` and a missing value, and the response returns `null` when there is no date. Sending `null` on update clears the date.

**Read paths**: the single enquiry read selects `e.*`, so the new column comes back without a query change. The lean table list query was left as it is, because the list does not show the date.

## Decisions made

**Notification work was built and then taken out.** An earlier version of this task also added a scheduled job that queued follow-up messages through the existing notification queue, a follow-up list endpoint for the in-app panel, and an image upload for the message. All of it was reverted before this PR, so none of it is in this diff. Only the date field is in. The notification and in-app panel work should be done as its own PR once the recipient (customer or sales rep) is settled.

## Notes

- The migration file number 138 is also used by another migration on this branch (`138_production_plan_date_index.sql`). Migrations are tracked by file name, so both still apply. Renaming one is only worth doing if it has not been applied anywhere yet.

## Next steps

- Follow-up notifications and the in-app follow-up panel, as a separate PR.
- The follow-up report page, planned for a later phase.

## Test plan

**Automated tests (no new tests added):**
- [ ] No unit or API test was added for this field. The enquiry module has no existing test file for it.
- [x] `npx tsc --noEmit`: passes with no errors.

**Manual checks (local):**
- [ ] Run `npm run migrate` and confirm `sales.enquiry.confirmation_date` exists - not done in this session.
- [ ] Create an enquiry with a date, edit it, clear it, and read it back through the API - not done against the running API. The frontend PR covers the screens that call it.
