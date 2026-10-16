# 2026-10-15 - dispatch-server-pagination (backend)

## Summary
`GET /dispatches` always returned every dispatch with every column of the dispatch row (`company_id`, `created_by`, `updated_at`, signature URLs and more), with no paging. The Dispatch page called it three times, so it pulled about 58 KB each load. This PR adds a table view, `GET /dispatches?view=table&page=1&limit=10`, that returns one page with only the columns the table shows, plus the total, the count per status tab, the KPI numbers and the list of parties. It follows the same shape the Order Booking list already uses. The plain `GET /dispatches` (no `view`) still returns the full list as before. Checked by type check only, the new query has not been run against the database yet.

## What changed
| File | Change |
|---|---|
| `src/modules/dispatch/dispatch.repository.ts` | Pulled the joins into one `DISPATCH_FROM` constant and the filters into `buildDispatchWhere` (status, dates, search, new party name filter). `listDispatches` uses them with the same behaviour as before. Added `listDispatchTablePage`, which runs five queries in parallel: the page rows, the matching total, counts per status, the KPI numbers and the distinct party names. |
| `src/modules/dispatch/dispatch.service.ts` | `listDispatches` returns the table page when `view` is `table`, otherwise the full list as before. |
| `src/modules/dispatch/dispatch.schema.ts` | Added optional `view` (`table`) and `partyName` to the list query. `page` and `limit` already existed (default 1 and 50, max 100). |
| `src/modules/dispatch/dispatch.types.ts` | Added `DispatchTableRow` and `DispatchTablePage`. |
| `src/modules/dispatch/dispatch.controller.ts` | no real change (it passes the query straight to the service) |

## Why
The Dispatch page should only get what it shows, with correct pagination, like the Order Booking page. The main work was the status tab counts and KPI numbers, which the page used to build by loading all rows three times. They are now counted in SQL.

## How
**Lean rows**: the table query selects 17 columns: id, dispatch code, final invoice id / code / no, total amount, work order code, order booking id, PI code, party name, status, vehicle type / number, driver name, approved at, approved by name, has gate pass, created at.

**Filters**: search, from date, to date, status and party name all go through one `buildDispatchWhere`. The search rule is unchanged (work order code, party name or final invoice code, case insensitive).

**Counts**: `statusCounts` uses the same filters except the status tab, so tab numbers follow search, dates and party. `summary` counts every dispatch for the company with no filters, for the KPI cards. Anything not `CONFIRMED` counts as pending.

**Paging**: `LIMIT` and `OFFSET` from `page` and `limit`. Order is `created_at DESC, id`, so rows do not jump between pages when two share a timestamp. The response has `items`, `totalCount`, `page`, `pageSize`, `totalPages`, `statusCounts`, `summary` and `partyNames`.

**Kept the same on purpose**: the full list, the detail endpoint, approve and gate pass endpoints and their responses are untouched.

## Decisions made
**Party list comes in the same response.** The Party dropdown used to be built from an unfiltered list of all dispatches. Returning `partyNames` with the page avoids a fourth call. Dispatches with no party name are skipped.

**Filter by party name, not id.** The dispatch query only joins the party name, so `partyName` is an exact match on that.

**Kept the old shape when `view` is missing.** Other callers of `/dispatches` keep working with no change.

## Notes
- Each table request runs five small queries. They share the same joins and run in parallel.
- Line endings in the touched files are unchanged (CRLF).

## Next steps
- Run the new endpoint against a database with real data and compare the counts with the old three-call result.

## Test plan
**Automated tests (no test cases for this module exist):**
- [x] Type check (`npx tsc --noEmit`): no errors.
- [ ] No dispatch test file exists, so no tests were added or run.

**Manual checks (local):**
- [ ] Call `GET /dispatches?view=table&page=1&limit=10&status=DRAFT` and check the item fields, `totalCount`, `statusCounts`, `summary` and `partyNames` - not run yet, backend not restarted.
- [ ] Call `GET /dispatches` with no `view` and check it still returns the full list - not run yet.
