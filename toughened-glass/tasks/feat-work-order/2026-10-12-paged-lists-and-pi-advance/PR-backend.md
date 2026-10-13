# 2026-10-12 - paged-lists-and-pi-advance (backend)

## Summary

The Work Order, Order Booking and PI Status list endpoints returned every field of every record, and the PI list was about 1.85 MB for 10 rows. This PR adds server-side paging and an optional lean `view=table` response to those lists, and makes the PI list return the real stored advance and balance so the frontend no longer guesses them.

I checked the repository functions against the live database with read-only scripts. I did not call the HTTP endpoints. See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/modules/proforma-invoice/proforma-invoice.schema.ts` | The list query accepts `page`, `limit` (max 100) and `view` (`table`). |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | New `listProformaInvoicePage`: the filtered, paged rows, a `COUNT`, and per-status counts. `listProformaInvoices` has a lean query used for `view=table`. Both queries now join the stored Advance term and return `advance_type`, `advance_value`, `advance_amount` and `balance_amount`. |
| `src/modules/proforma-invoice/proforma-invoice.service.ts` | Returns the page object when `page` or `limit` is sent, otherwise the old array. |
| `src/modules/proforma-invoice/proforma-invoice.types.ts` | New `ProformaInvoicePage` type and the four advance fields on the list row type. |
| `src/modules/work-order/*` | Work order list supports paging, the lean table view and status counts. |
| `src/modules/order-booking/*` | Order booking list supports paging, the lean table view and status counts. Search by booking number casts it to text. |
| `src/modules/production-scan/*` | Small changes for the scan screens (see the frontend document). |

## Why

The tables only show a handful of columns but received photos, charges, signatories and item lookups for every record, and loaded all records at once. Paging plus a lean view keeps the response small as data grows. The advance and balance columns were worked out in the browser from a saved percentage or a 50% default, which could disagree with what was approved.

## How

**Paging**: an inner query applies the filters, order and `LIMIT/OFFSET` on the PI rows only. The heavy lookups then run for that page's rows. The response is `{ items, totalCount, page, pageSize, totalPages, statusCounts }`.

**Lean view**: `view=table` selects only the table columns, the fields the status badge reads and the ids the row actions use. Without `view` the full row is returned.

**Old callers keep working**: with no `page` or `limit` the endpoints return the same array as before, so other screens are not affected.

**Advance**: a `LEFT JOIN LATERAL` reads the `Advance` row of `invoice.proforma_invoice_payment_term` (the row `saveAdvancePaymentTerm` writes). A `PERCENTAGE` advance is `total * value / 100` rounded to 2 places. A `VALUE` advance is the value, capped at the total. Balance is the total minus the advance and never below 0. When no advance is stored all four fields are null.

## Decisions made

**The 50% default is not applied on the server.** A PI with no stored advance returns null, and the screen shows "-". The old 50% was a frontend guess for approved PIs and could show money that was never agreed.

**The detail query is unchanged.** It does not return the advance fields. The PI view reads its advance from the approve flow and the saved terms as before.

## Notes

- Live data: 109 PIs, 76 have a stored advance (74 percentage, 2 fixed value). The other 33 now show no advance.
- Measured on 10 rows: PI list 1.85 MB and 2.0 s before, about 5.5 KB and 0.25 s with `view=table`.
- The backend needs a restart for any of this to take effect.

## Next steps

- Add the same advance fields to the PI detail query so the PI view does not depend on the approve flow.
- Add tests for the page counts and for percentage and fixed advances.

## Test plan

**Automated tests (none for these modules):**
- [x] `npx tsc --noEmit`: passed after the last change.

**Manual checks (read-only scripts against the live database, repository level):**
- [x] PI page with `view=table` and without: same ids, lean has 20 fields, full has 97.
- [x] A 50% PI returned advance 2295.50 on a total of 4591. A fixed 6800.50 PI on a total of 13601 returned balance 6800.50. PIs without a stored advance returned null.
- [x] Counts from the earlier check: PI 68 total (47 confirmed, 19 draft, 2 declined) and Order Booking 111 total matched the status counts. The PI table has grown since (109 now).
- [ ] Call the HTTP endpoints with `page`, `limit`, `view` and each status. Not checked.
- [ ] Order Booking and Work Order search and status counts through the API. Not checked.
