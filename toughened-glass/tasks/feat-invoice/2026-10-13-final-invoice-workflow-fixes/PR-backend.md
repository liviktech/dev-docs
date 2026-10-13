# 2026-10-13 - final-invoice-workflow-fixes (backend)

## Summary
This PR updates the Final Invoice backend module to support server-side pagination, KPI aggregation, and a lean display query matching the PI Status pattern. Previously, the final invoice list endpoint loaded all database records into memory and returned heavy internal fields like company IDs, work order IDs, raw timestamps, intermediate advance terms, and full approver records on every page request. The list endpoint now supports `page`, `limit`, and `view=table`, returning only the columns displayed in the table and computing the remaining balance directly in SQL. Heavy approver signatures and detail views continue to load on demand by invoice ID.

## What changed
| File | Change |
|---|---|
| `src/modules/final-invoice/final-invoice.repository.ts` | Added `LIST_TABLE_SELECT` with 16 lean display columns, implemented `REMAINING_SQL` for database-level balance calculations, updated `listFinalInvoices` to support `view=table`, and implemented `listFinalInvoicePage` returning items, total count, status tab counts, and KPI totals in a single request. |
| `src/modules/final-invoice/final-invoice.schema.ts` | Added `page`, `limit`, and `view` (`table`) validation to `listFinalInvoicesQuerySchema`. |
| `src/modules/final-invoice/final-invoice.service.ts` | Updated `listFinalInvoices` service to delegate to `listFinalInvoicePage` whenever pagination parameters (`page` or `limit`) are present. |
| `src/modules/final-invoice/final-invoice.types.ts` | Added `FinalInvoiceKpis` and `FinalInvoicePage` types, and updated `FinalInvoiceListAggregateRow` to extend `Partial<FinalInvoiceRow>` so lean table projections can omit database internal IDs. |

## Why
The Final Invoice table previously fetched all columns for every invoice, including internal database IDs, creation timestamps, and raw advance terms that were never displayed on screen. In addition, `remaining_amount` was calculated in JavaScript by reading joined advance payment terms, requiring extraneous columns to be selected. Aligning Final Invoice with the lean query pattern used in PI Status removes redundant data transfer, speeds up database queries, and avoids sending sensitive or unnecessary database internals over the wire.

## How
**Lean table projection (`LIST_TABLE_SELECT`)**: Selected only the 16 display columns (`id`, `final_invoice_no`, `final_invoice_code`, `invoice_date`, `work_order_code`, `order_booking_id`, `pi_code`, `party_name`, `total_amount`, `remaining_amount`, `status`, `party_approved_at`, `party_approved_by_name`, `admin_approved_at`, `admin_approved_by_name`, `has_payment_proof`). Omitted `company_id`, `work_order_id`, `created_at`, `payment_proof_file_name`, and raw advance terms.

**Database-level remaining balance calculation (`REMAINING_SQL`)**: Replaced JavaScript remaining balance calculation with a SQL expression subtracting agreed advances from total invoice amount with a floor of zero. This runs directly in Postgres and feeds both table rows and KPI summaries.

**Unified pagination and KPI retrieval (`listFinalInvoicePage`)**: Replaced multiple round-trips by running row queries, total counts, tab counts by status, and KPI aggregations concurrently via `Promise.all`. Defaulted paginated list queries to `view=table`.

**Preserved full detail query**: Kept `findFinalInvoiceById` reading from `LIST_SELECT` with full header columns, items, charges, and taxes, so opening Preview and Detail modals continues to load complete invoice details without breaking.

## Decisions made
**Defaulted paginated page queries to lean table view.** When callers request paginated rows via `page` or `limit`, the service defaults `view` to `table` if not explicitly specified. Full rows remain accessible for unpaginated calls like CSV export by omitting the view filter.

## Notes
- Checked type safety live with `npx tsc --noEmit` and production build with `npm run build`, exiting cleanly with code 0.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: Production build via TypeScript compiler completed with exit code 0.

**Manual checks (local):**
- [x] Tested `GET /api/final-invoices?page=1&limit=10&view=table&status=DRAFT` and confirmed response items only contain the 16 display columns with zero database internal keys leaked.
- [x] Tested `GET /api/final-invoices/:id` and verified complete invoice details (items, charges, taxes, approver signatures) load cleanly.

