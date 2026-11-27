# 2026-10-16 - insurance-pi-charges-mobile-fixes (backend)

## Summary

Insurance was charged for each product inside the Order Booking (it came from the product's charge rules). It should be charged once on the whole Proforma Invoice instead. Also, when a Proforma Invoice was saved, the backend did not keep which order charge row each saved charge belonged to, so reopening the PI could show the same charge twice.

This PR stops the Order Booking from calculating Insurance per product, removes the Insurance rows already calculated on draft bookings, and saves a small "row key" with every PI charge so the frontend can put each charge back on the right row.

**Confirmed bookings keep their old Insurance rows on purpose.** They were booked with that amount, so changing them would change a finished order. This was a choice, not a miss.

**Migration 065 was not run against the shared dev database by me.** It has to be run by whoever deploys this, see "Notes" below.

## What changed

| File | Change |
|---|---|
| `src/modules/order-booking/order-booking.repository.ts` | In `recomputeChargesForItemInTx`, the product charge rules now skip any charge whose master charge has code `INSUR` or a name containing "insurance". Everything else in the calculation is the same. |
| `src/database/migrations/065_remove_item_level_insurance_charge.sql` (new) | Deletes the auto calculated (not manual) Insurance rows from bookings that are still DRAFT, so their totals stop including it. Safe to run more than once. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | Each charge in the create and update payload can carry a `chargeKey`. It is saved in the existing `calculation_basis` column as `KEY:<key>` (cut to 40 characters). Nothing else about how charges are saved changed. |

## Why

The business wants one Insurance charge per Proforma Invoice, calculated on the PI's basic amount, not a separate one on every product. The duplicate charge problem came from the saved charge having no link back to the form row it came from, so the reopened form could not tell a master order charge from a custom one.

## How

**Insurance in the Order Booking.** The charge calculation reads the product's charge rules (`master.product_charge`). A rule is now ignored when its master charge is Insurance. The master data and the product rules are not touched, so nothing breaks in the Master screens. The Add Product preview uses the same calculation, so Insurance disappears there too.

**Draft bookings.** Migration 065 removes the old auto Insurance rows for drafts. A draft that is saved again would lose them anyway, the migration just does it for all drafts at once.

**Charge key on the PI.** `charge_code` could not be used for this. It has a foreign key to `master.charge`, and the order charges (Admin, Packing and so on) live in a different table (`master.order_charge`). Putting an order charge code there makes the insert fail, and the insert is wrapped in a try/catch, so every charge would be dropped without any error. `calculation_basis` is a free text column that was not used for charges, so the key goes there. The detail endpoint already returns this column, so no change to the read side was needed.

## Decisions made

**Skip Insurance in the calculation instead of deleting the product rules.** Deleting rules would change Master Data that people may still look at, and would not stop a rule being added again later. Filtering in one place is easier to undo.

**Match Insurance by code `INSUR` or by name.** The seed data uses code `INSUR` with name `INSURANCE`, but other companies may have created their own code. Matching both covers both cases.

**Did not add a new column for the charge key.** It would need a migration on a table other work depends on. The existing text column does the job.

## Notes

- Run `065_remove_item_level_insurance_charge.sql` (with `npm run migrate`) on each environment. Until then, draft bookings keep their old Insurance rows until the booking item is saved again.
- The PI side needs the matching frontend PR. On its own, this PR only removes Insurance from the Order Booking.
- Existing PIs saved before this change have no `KEY:` value. The frontend matches those by charge name.

## Next steps

- Decide whether the old Insurance rows on confirmed bookings should be cleaned up, and how, with the business.

## Test plan

**Automated tests (no test files added or changed in this PR):**
- [x] Type check (`tsc --noEmit`) on the backend: passes.
- [ ] No automated tests cover the charge calculation or the PI charge save. None were added.

**Manual checks (local):**
- [ ] Migration 065 on a database with draft bookings that have Insurance rows - not run, no shared database was changed from this session.
- [ ] Save a PI with a custom charge and a master order charge, then reopen it and check `calculation_basis` - not checked against the real API. Covered only by reading the code and by the type check.
- [ ] Add a product with an Insurance rule in the Add Product modal and check that no Insurance charge shows up - not checked in the running app.
