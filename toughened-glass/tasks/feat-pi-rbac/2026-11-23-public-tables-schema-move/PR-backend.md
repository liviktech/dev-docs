# 2026-11-23 - public-tables-schema-move (backend)

## Summary

Eight tables were created in the `public` schema by mistake, even though the project keeps one schema per module. This PR moves them to the schema of their module, with their data, and changes every query in the code to the new names. Five of them go to `inventory` (`inventory_lot`, `inventory_movement`, `inventory_lot_group`, `inventory_lot_edit_request`, `inventory_lot_offcut`), two go to `production` (`work_order_material_choice`, `work_order_stock_selection`) and one goes to `master` (`archived_record`).

I read the live data and the dependencies before writing the migrations. I did not run the migrations, so the move itself is not checked on a real database. See "Test plan".

**The other `public` tables are not moved on purpose** - see "Next steps" below. This PR only covers the eight tables listed above. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/172_move_inventory_lot_movement_archive_schema.sql` (new) | Drops the empty `inventory.inventory_lot` and `inventory.inventory_movement` (after checking they are empty), then moves `public.inventory_lot`, `public.inventory_movement` and `public.archived_record` to `inventory`, `inventory` and `master`. Also rewrites the table names stored inside old archive snapshots. |
| `src/database/migrations/173_move_lot_group_offcut_edit_request_wo_stock_schema.sql` (new) | Moves `inventory_lot_group`, `inventory_lot_edit_request`, `inventory_lot_offcut` to `inventory` and `work_order_material_choice`, `work_order_stock_selection` to `production`. Same archive snapshot rewrite. |
| `src/modules/archive/archive.snapshot.ts` | The list of tables a snapshot may restore into now uses the new schema names. |
| `src/modules/archive/archive.specs.ts`, `archive.service.ts` | Queries use the new names. `archived_record` is now `master.archived_record`. |
| `src/modules/inventory/inventory.repository.ts`, `customer-lot.repository.ts`, `customer-lot-dispatch.repository.ts`, `inventory-edit-request.repository.ts`, `inventory.service.ts`, `inventory.types.ts` | Table names in the SQL changed from `public.` to `inventory.`. |
| `src/modules/work-order/work-order-inventory.service.ts`, `work-order-stock.service.ts`, `work-order.repository.ts`, `work-order-sync.service.ts`, `replace-order-detail.repository.ts` | Table names changed to `inventory.` and `production.`. |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts`, `purchase-edit-request.repository.ts`, `src/modules/inventory-purchase-return/purchase-return.repository.ts` | Table names changed for the lot, group and movement tables they join. |
| `src/modules/fir/fir-party-lot.ts`, `fir-ro.repository.ts`, `src/modules/dispatch/dispatch.repository.ts`, `src/modules/production-scan/production-scan.service.ts`, `src/modules/report/inventory/inventory-report.repository.ts` | Table names changed in the SQL. |
| `src/database/opening-balance-correction.ts` | Table names changed in the SQL. |
| `src/database/seeds/012_inventory_seed.sql` | Inserts into `inventory.inventory_lot` and `inventory.inventory_movement`. A fresh database runs the seeds after all migrations, so the old names would not exist by then. |
| `docs/DATABASE_SCHEMA.md` | Table names updated. |
| Old migrations (001 to 171) | no real change. They still say `public.` on purpose, see "Decisions made". |

## Why

The team keeps one schema per module (inventory, purchase and so on). These tables were written as `public.<name>` in their migrations instead, so they ended up outside their module. A reviewer asked why. The first inventory migration (`030_inventory_schema.sql`) used `public.` for `inventory_lot` and `inventory_movement`, and the later tables that point at them copied the same prefix.

The real work was the two tables that existed twice. `001_multi_tenant_schema.sql` had already created empty `inventory.inventory_lot` and `inventory.inventory_movement`, so the same names were in two schemas and only the `public` copies held data (34 lots, 158 movements when I checked).

## How

**Move, not copy**: each table is moved with `ALTER TABLE ... SET SCHEMA`. The rows, indexes, constraints and foreign keys move with it, so no data is copied and no row can be lost or doubled. Foreign keys from other tables (offcut, edit request, purchase item, work order stock selection, `fir_item`, `inventory_movement.offcut_id` and so on) follow the table on their own.

**Duplicates are checked before they are dropped**: migration 172 counts the rows in the old `inventory.inventory_lot` and `inventory.inventory_movement`. If either has even one row, it stops with an error. The drop is a plain `DROP TABLE` with no CASCADE, so if anything else depends on them it stops instead of deleting it.

**One transaction, safe to run again**: each step checks what exists first. Migration 173 refuses to move a table if the same name already exists in the target schema.

**Archive snapshots keep working**: a snapshot stores the table name of every row it saved (for example `public.inventory_lot`), and restore only accepts names on its allow list. Both migrations rewrite those names inside the stored snapshots to the new ones, so old archived records can still be restored.

**Kept the same on purpose**: column names, types, the `lot_code` default (`public.next_business_code`) and every API response. Only the table names in the SQL changed.

## Decisions made

**Old migrations were not edited.** 030, 079, 081 and the others still create the tables in `public`. The project rule is that migrations are append-only, and a database that already ran them must still match. A fresh database runs them in order and ends up in the right place after 172 and 173.

**`archived_record` goes to `master`.** It does not belong to one module (it archives Order Booking, Purchase and Lot records). I first put it in `core`, then moved it to `master` as the team asked.

**The two `work_order_*` tables go to `production`, not `inventory`.** They are keyed to a work order and sit next to `production.work_order`. If the team sees them as inventory tables, only the target in 173 and the names in the code need to change.

**Moved with SET SCHEMA instead of create, copy, drop.** It is one statement per table, keeps every constraint, and cannot lose rows.

## Bugs found while testing (already there before, not fixed here)

**1. The code and one database disagree about the purchase tables.** In the database my `.env` points at (`neondb`), the purchase and procurement tables are already in the `inventory` schema, but the code still reads `public.inventory_purchase` (42 places). No migration in the repo moves them, so someone moved them by hand, or that database is not on the same migrations as `rubine_dev`. The app would fail against that database. This was already the case before this PR and I did not touch those tables.

## Notes

- The data checks were read only queries against `neondb`: row counts, foreign keys in and out, views, triggers and functions that mention these tables. There were no views, triggers or functions on any of the eight tables.
- Row counts when I checked: `inventory_lot` 34, `inventory_movement` 158, `archived_record` 5, `inventory_lot_group` 21, `inventory_lot_offcut` 109, `work_order_stock_selection` 33, `inventory_lot_edit_request` 0, `work_order_material_choice` 0.
- 173 needs `master.archived_record` from 172, so run them together and in order.
- The frontend was not touched. No API route or response changed.

## Next steps

- Move the remaining `public` tables to their schemas: the purchase and procurement tables (`inventory_purchase*`, `indent_request*`, `purchase_request*`), and `attachment`, `daily_business_sequence`, `order_photo`, `order_photo_reading`. They belong to other authors and were not in the request. `schema_migrations` and `schema_seeds` are tracking tables and stay in `public`.
- Sort out the `neondb` mismatch from the bug above before anyone runs the app against it.
- Add a rule to the team prompt or review checklist that new tables must name their module schema.

## Test plan

**Automated tests (all passing, but they do not cover the migrations):**
- [x] `npx tsc --noEmit`: passes with no errors after the renames.
- [x] `npm test` (`src/**/*.test.ts`): 8 passing, 0 failing.
- [ ] `tests/` folder tests: not run in this session.

**Manual checks (local):**
- [x] Read only queries on `neondb` for row counts, foreign keys, views, triggers, functions and name clashes in the target schemas: results are in "Notes".
- [x] Searched `src`, `tests` and `docs` for leftover `public.<table>` names for all eight tables: none left outside the old migrations.
- [ ] Run migrations 172 and 173 on `rubine_dev` - not done. I did not know which database `neondb` is and did not want to change a shared one. Take a backup first, then run `npm run migrate`, then check the row counts match the numbers in "Notes".
- [ ] Open Stock, a work order with stock selection, and Settings > Archive (including a restore of an old record) after the migration - not done, because the migration was not run.
