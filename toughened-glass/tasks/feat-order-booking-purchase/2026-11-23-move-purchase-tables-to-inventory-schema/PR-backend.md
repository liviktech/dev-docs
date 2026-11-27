# 2026-11-23 - move-purchase-tables-to-inventory-schema (backend)


## What changed

### Migration

`src/database/migrations/172_move_purchase_tables_to_inventory_schema.sql`

- Runs `ALTER TABLE public.<t> SET SCHEMA inventory` for the 15 tables, inside one `DO` block.
- This changes the namespace only. Rows are not copied. Indexes, constraints, defaults and foreign keys move with the tables or keep pointing at the same objects, including `inventory_lot_group`'s foreign key to `inventory_purchase`.
- Safe to re-run: a table is moved only if it is still in `public`.
- Stops with a clear error if `inventory.<t>` already exists, instead of overwriting.
- Rollback: `ALTER TABLE inventory.<t> SET SCHEMA public;` for the same 15 names.

Older migrations (077-167) are left as they are, because applied migrations are never edited. On a fresh database they still create the tables in `public`, and 172 then moves them.

### Backend code

Every `public.<table>` for the 15 tables is now `inventory.<table>`. Files:

- `modules/inventory-purchase/inventory-purchase.repository.ts` (largest)
- `modules/inventory-purchase/purchase-edit-request.repository.ts`
- `modules/inventory-purchase/inventory-purchase.types.ts` (comments only)
- `modules/procurement/procurement.repository.ts`
- `modules/procurement/purchase-request-modification.ts`
- `modules/inventory/inventory.repository.ts`
- `modules/inventory-purchase-return/purchase-return.repository.ts`
- `modules/inventory-purchase-return/purchase-return.guard.ts`
- `modules/tally/tally.sources.ts`, `modules/tally/tally.repository.ts`
- `modules/archive/archive.specs.ts`, `modules/archive/archive.snapshot.ts`
- `modules/report/sales-income/sales-income-report.repository.ts`

Two places build the table name in code and needed a hand edit, not a find-and-replace:

- `archive.specs.ts`: the `byPurchase` helper used `` `public.${table}` `` and now uses `inventory`.
- `procurement.repository.ts`: `nextNumber` checked `public.${table}` for `indent_request`, `purchase_request` and `grn`. It now picks `public` for `grn` and `inventory` for the other two.

### Docs

`docs/database_schema.sql` has a new "Purchase & Procurement" section with the 15 table definitions. They were generated from the live database after the move, so they match what is deployed.


## Why

- `SET SCHEMA` is a metadata change. Postgres tracks foreign keys by object ID, so they do not depend on the schema name.
- Checked on the live database before the move: no views, functions or triggers reference these tables, and the `inventory` schema had no name clashes.
- Nothing in the application sets `search_path`, so every reference had to be schema-qualified. After the change no `public.<table>` reference to these tables is left in `src`, `tests` or `api`.
- Calculations live in application code and stored columns, so they do not change.


## Decisions made

- **Target schema is `inventory`**, the existing schema, instead of a new `purchase` or `procurement` schema. This was the owner's choice.
- **The two purchase-return tables were added late** (originally the list had 13) and are in the same migration.
- **No compatibility views were added in `public`.** A hard cutover keeps the schema clean, and the cost is the deploy ordering described above.
- **`search_path` was not changed.** Qualified names keep it obvious which schema a table is in, and avoid picking a wrong same-named table.


## Notes

**Migration 172 has already been applied to the shared Neon database.** From that moment, any backend still running the old code fails with `relation "public.inventory_purchase" does not exist` on purchase, procurement and return screens.

- Pull this branch and restart your local backend.
- In any new SQL or migration, write `inventory.<table>` for the 15 tables above. `public.<table>` will fail.
- If you have an open branch that adds queries on these tables, change the schema prefix before merging.
- Pick migration numbers above 172.


## Next steps

- **Old code breaks while the new code is not deployed.** This is the one real risk, and it is why the deploy and the migration have to go together.
- **Other branches.** Any branch with new `public.inventory_purchase*`, `public.purchase_request*` or `public.indent_request*` SQL will fail until the prefix is changed.
- **Rollback.** Run `SET SCHEMA public` for the 15 tables and revert the code changes. No data is touched in either direction.


## Test plan

Done by the author:

1. **Throwaway local Postgres.** A copy of the Neon schema plus the rows of the 13 tables that had data was loaded, then migration 172 was run.
   - Row counts for all tables were identical before and after.
   - Running 172 a second time changed nothing and raised no error.
   - Foreign keys stayed valid and were still enforced (a bad parent was rejected).
2. **Neon, after applying 172.**
   - All 15 tables are in `inventory` and none are left in `public`.
   - Row counts match the counts taken before the move.
   - All 23 foreign keys pointing at these tables are valid.
3. **Query smoke test against the moved schema, local and Neon (read-only).** 21 repository functions ran without error:
   - purchase list, detail, approvals, payments, charges, stock posting and payment transactions
   - indent list and detail and their items
   - purchase request list and detail and their items
   - procurement info
   - Tally ready-to-send lists for purchase invoices and purchase payments
   - purchase return list, detail and items
4. `npx tsc --noEmit` passes.

**Not tested** (please check in the app after deploying):

- Write flows were not run against Neon, to avoid adding test rows to the shared database. These are create or approve a purchase, Add to Stock, record a payment, create an indent or purchase request, create or approve a return, and archive or delete a purchase. They use the same table names that the read tests covered.
- No screen was clicked through in the browser.

The existing backend tests (`npm test`) do not run any SQL, so they say nothing about this change.

Suggested manual check after deploy: open Purchases, Indents, Purchase Requests and Purchase Returns, then create or approve one record in each.
