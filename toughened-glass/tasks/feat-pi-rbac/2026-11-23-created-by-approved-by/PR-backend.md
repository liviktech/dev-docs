# 2026-11-23 - created-by-approved-by (backend)

## Summary

List screens had no way to show who created a record or who approved it. Some tables already stored it (dispatch, final invoice, production plan, purchase, purchase return, indent), many did not (enquiry, order booking, work order, proforma invoice, employee and about 25 master tables). This PR adds the missing `created_by` columns, adds `approved_by` to the work order, stores the user on every create and approval path of those tables, and makes the list queries return `created_by_name` and `approved_by_name` for the screens.

Records that already exist keep a blank creator. Nothing is guessed, except where the database already holds a real record of who did it (see migration 176 under "How").

**Approved By on master data is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

**Sales Visits Created By is not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/174_created_by_approved_by_columns.sql` (new) | Adds a nullable `created_by UUID REFERENCES core.app_user(id)` to 37 tables (documents, `hr.employee` and the master tables). Adds `approved_by` to `production.work_order`. A table that does not exist is skipped. Safe to run again. |
| `src/database/migrations/176_backfill_created_approved_by.sql` (new) | Fills `work_order.approved_by` and the creator of customer material lots only where stock movements already record the user. Never overwrites a value. |
| `src/modules/enquiry/*` | The create path stores `created_by`. The list and table selects return `created_by_name` and `approved_by_name` (the approver column already existed). |
| `src/modules/order-booking/*` | Create and Copy booking store `created_by`. The paged table returns `created_by_name`. |
| `src/modules/work-order/*` | Create stores `created_by`. Both approval paths store `approved_by` (the first approver is kept). The work order made automatically when a PI is approved records the PI approver as creator. The list returns both names. |
| `src/modules/proforma-invoice/*` | Create stores `created_by` (same user as `prepared_by`). The lean PI Status query and the two full selects return `created_by_name` (falls back to the preparer for old rows) and `approved_by_name`. |
| `src/modules/attendance/*` | Employee create stores `created_by`. The employee list returns it. |
| `src/modules/master/stock-option/*` | One shared path that covers 14 masters (supplier, glass colour, finish, layer, port, destination, logo, designation, cutting, edge and grinding types, three tempering lists). Create stores the user, the list returns the name. |
| `src/modules/master/party`, `product`, `item`, `charge`, `order-charge`, `purchase-charge-type`, `tax-profile`, `payment-term/mode`, `invoice-term`, `shift`, `work-center`, `inventory-location`, `sales-rep`, `role` | Each one: the controller passes `req.user.id` to the service, the service to the repository, the insert writes `created_by`, the list query returns `created_by_name`, and the types carry it. |
| `src/modules/inventory/customer-lot.repository.ts`, `inventory.service.ts`, `inventory.controller.ts` | Customer material lots store `created_by` on create. The Customer Material table returns the name. |
| `src/modules/dispatch/*`, `src/modules/final-invoice/*` | The list queries return `created_by_name`. |
| `src/modules/production-plan/*` | The list returns `approved_by_name` (the creator name was already there). |
| `src/modules/fir/*` | The list returns `created_by_name` (FIR already stored it). |

## Why

The frontend list tables needed two last columns, Created By and Approved By. Where the database already had the data, only the list queries needed the names. Where it did not, the column, the create path and the list query all had to change. The real work was the long tail of master modules, each of which passed no user id down from the controller.

## How

**Same pattern everywhere**: the controller passes `req.user!.id` as a new optional last argument. The service and repository pass it on. The insert adds `created_by` and the list select adds a scalar subquery on `core.app_user` for the name. Every new argument is optional, so existing callers keep working.

**Scalar subqueries, not joins**: the name comes from `(SELECT u.full_name FROM core.app_user u WHERE u.id = ...)`. The list queries already have grouping, counts and search filters, and a join would have changed their column names and group by rules.

**Migration 176 only uses real evidence**: approving a work order issues its glass from stock in the same step, and every issue movement stores the user. So the user on the work order's earliest ISSUE movement is the approver. A customer material lot takes the user on its RECEIPT movement. Rows that already have a value are never changed.

**Kept the same on purpose**: column names, response shapes of existing fields, and the `lot_code`, `pi_code` and other business codes. Only new fields were added.

## Decisions made

**Approved By only where an approval step exists.** Master data has no approval, so those tables only get Created By. Adding an always blank column was the alternative, and it was not picked.

**Work order keeps the first approver.** The second approval path uses `COALESCE(approved_by, $user)`, so approving again does not change who is shown.

**PI Created By falls back to the preparer.** `proforma_invoice` already had `prepared_by`. New PIs store `created_by` too, and the list shows `COALESCE(created_by, prepared_by)` so old PIs are not blank.

**Sequences were tried and reverted.** I started moving `public.daily_business_sequence` to `master` with a `created_by` column (a draft migration 175). It was reverted before it was ever run, so there is no 175 and the table stays in `public`.

**Old rows are not filled with a made up name.** Migration 176 fills only what the movement history proves. The rest stay blank and the screens show a dash.

## Bugs found while testing (already there before, not fixed here)

**1. The first customer material insert does not store a creator.** In `customer-lot.repository.ts` there are two inserts into `inventory.inventory_lot`. The older one (lots tied to an order item) is not used by Add Customer Material anymore, so I left it. It was already like this.

## Notes

- I ran the real create and list code against the dev database for shift, work center, tax profile, charge, purchase charge, invoice term, payment mode, storage location, sales rep and role. Each stored the creator and the list returned the name. The test rows were removed afterwards and a follow up query found none left.
- That check found one real bug (the Invoice Term list used its own mapper that was missing the name). It is fixed in this PR.
- Migration 174 is applied on the dev database I used. Migration 176 was written but I did not run it. It would fill 24 of the 60 approved work orders there. It would fill no customer lots, because none has a receipt movement.
- Another branch has a migration called `174_pi_manager_approval.sql`, so two 174 files exist across branches. The file names differ, so both run. You may want unique numbers.
- Existing rows stay blank, for example 73 parties, 114 items and 93 work centers on the dev database. Only records created after the backend restarts carry a name.

## Next steps

- Sales Visits Created By. Its list is built from a combined day and visit register in three queries and needs its own change.
- Decide what old blank rows should show (leave the dash, or fill a label such as "System").
- Give the two 174 migrations unique numbers.
- Add a test that creates a record and checks the list returns the creator name. The existing tests do not cover this.

## Test plan

**Automated tests (all passing, but none cover this change):**
- [x] `npx tsc --noEmit`: passes with no errors.
- [x] `npm test` (`src/**/*.test.ts`): 8 passing, 0 failing.

**Manual checks (local):**
- [x] A throwaway script created a test row through the service for 10 masters, read the list back, and removed the row. All 10 showed the creator after the fix.
- [x] Counted rows with and without a creator on 15 tables on the dev database: only the records made after the change carry one.
- [ ] Create a Party, Item, Product, Customer Material lot, Enquiry, Order Booking and PI from the screens and read the new column. Not done, because I did not run the app.
- [ ] Approve a work order and read Approved By. Not done for the same reason.
- [ ] Call the running server over HTTP and read `createdByName` in the response. I tried and the request was blocked, so it was not done.
- [ ] Run migration 176 and check the counts (24 work orders expected). Not run.
