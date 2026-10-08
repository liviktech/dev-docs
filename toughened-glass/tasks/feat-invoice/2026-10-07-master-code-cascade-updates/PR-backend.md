# 2026-10-07 - master-code-cascade-updates (backend)

## Summary
This PR adds cascading update support on foreign keys referencing master entity codes, allowing codes for charges, work centers, sales representatives, products, and sub-products to be renamed without foreign key errors. It also hardens repository update queries to ignore unchanged codes and improves database error handling to return clear conflict errors when records are referenced by downstream transactions.

## What changed
| File | Change |
|---|---|
| `src/database/migrations/051_cascade_update_master_codes.sql` (new) | Adds ON UPDATE CASCADE to foreign key constraints referencing master.charge, master.work_center, and master.sales_rep codes across sales, production, and invoice tables. |
| `src/database/migrations/052_cascade_update_product_codes.sql` (new) | Adds ON UPDATE CASCADE to foreign key constraints referencing master.product and master.sub_product codes across rates, charges, routing, orders, work orders, invoices, and inventory. |
| `src/modules/master/charge/charge.repository.ts` | Skips updating the code column if the provided code matches the current code. |
| `src/modules/master/charge/charge.service.ts` | Catches foreign key violation errors and throws a ConflictError with CHARGE_REFERENCED code. |
| `src/modules/master/product/product.repository.ts` | Skips updating the code column if unchanged, and preserves the active code reference when updating product details in transaction. |
| `src/modules/master/product/product.schema.ts` | Loosens rate and charge ID schema validations from strict UUID format to optional string to support existing or client-generated identifiers. |
| `src/modules/master/product/product.service.ts` | Wraps create and update operations in a database error handler that maps unique violations to PRODUCT_CODE_TAKEN and foreign key violations to PRODUCT_REFERENCED. |
| `src/modules/master/sales-rep/sales-rep.repository.ts` | Skips updating the code column if the provided code matches the current code. |
| `src/modules/master/sales-rep/sales-rep.service.ts` | Catches foreign key violation errors and throws a ConflictError with SALES_REP_REFERENCED code. |
| `src/modules/master/work-center/work-center.repository.ts` | Skips updating the code column if the provided code matches the current code. |
| `src/modules/master/work-center/work-center.service.ts` | Catches foreign key violation errors and throws a ConflictError with WORK_CENTER_REFERENCED code. |

## Why
Previously, master entity foreign keys used default RESTRICT rules on updates. When users edited an existing product, charge, work center, or sales representative and modified the code, PostgreSQL raised foreign key constraint violations because related child records (such as order booking items, proforma invoice lines, and inventory lots) referenced the old code. Additionally, saving a product with unchanged codes still attempted to re-apply the code in SQL SET clauses, risking unnecessary constraint checks or false conflicts.

## How
**Cascading foreign keys**: Migrations 051 and 052 drop existing foreign key constraints on dependent tables and recreate them with ON UPDATE CASCADE. Renaming a master code automatically propagates the new code to dependent transactions and child records.
**Code change check in repositories**: In charge, work center, sales rep, and product repositories, code = $X is added to the SQL SET clause only if input.code !== undefined && input.code !== code.
**Conflict error mapping**: In service layers, error handlers catch isForeignKeyViolation and isUniqueViolation, throwing clean 409 ConflictError responses instead of raw database 500 crashes.
**Relaxed ID schemas**: In product.schema.ts, rateInputSchema.id and chargeInputSchema.id allow z.string().optional(), preventing Zod validation failures on non-UUID identifiers.

## Decisions made
**Applied ON UPDATE CASCADE while retaining ON DELETE RESTRICT on transactional records.** Child transactional tables (orders, invoices, work orders, inventory) cascade code renames safely, but still prevent accidental record deletions when transactions exist.

## Notes
- Verified TypeScript compilation and build live with npm run build (tsc), passing cleanly with exit code 0.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: TypeScript build completed with exit code 0.

**Manual checks (local):**
- [x] Verified updating master records without changing code avoids unnecessary code SET statements.
- [x] Verified schema accepts string IDs in product rates and charges without UUID validation rejection.

