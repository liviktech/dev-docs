# 2026-10-15 - dispatch-server-pagination (frontend)

## Summary
The Dispatch page was slow and noisy on load. It fired three separate `/dispatches` calls that each returned every row with every database column (about 58 KB), it had no pagination, and it called `/master/companies/my-company` on load even though the Gate Pass modal was closed. This PR makes the page ask for one page of rows at a time with only the columns the table shows, the same way the Order Booking page already does. Tab counts, the KPI cards and the party dropdown now come from that one response. The page looks and behaves the same as before. Pagination controls now work against the server. Checked by type check only, not yet run against the new backend in the browser.

## What changed
| File | Change |
|---|---|
| `src/features/dispatch/dispatch-page.tsx` | Replaced the three list queries with one `dispatchTableQueryOptions` call. Added `page`, `pageSize` and a 350 ms debounced search. Any filter or tab change goes back to page 1. Tab counts, KPI cards and party options are read from the response. The party filter is now sent to the server as `partyName`. Steps back a page if a refetch leaves the current page empty. |
| `src/features/dispatch/api.ts` | Removed `fetchDispatchList`. Added `fetchDispatchTablePage`, which calls `/dispatches?view=table&page=..&limit=..` plus the filters and maps the items with the existing `mapApiToDispatchRecord`. |
| `src/features/dispatch/queries.ts` | Replaced `dispatchListQueryOptions` with `dispatchTableQueryOptions` (keeps the previous page on screen while the next loads). Approve and gate pass mutations still invalidate the same `['dispatches']` key. |
| `src/features/dispatch/types.ts` | Added `DispatchTablePage` (the API page shape) and an optional `partyName` on `DispatchFiltersState`. |
| `src/features/dispatch/components/dispatch-table.tsx` | Added an optional `pagination` prop that is passed to the existing `manualPagination` option of `DataTable`. Columns and actions are unchanged. |
| `src/features/dispatch/components/gate-pass-modal.tsx` | `useMyCompany()` is now `useMyCompany({ enabled: isOpen })`, so the company is fetched only when the modal opens. |

## Why
The Dispatch page was the odd one out. Order Booking and Final Invoices already use server-driven pagination with a lean `view=table` response. The user wanted the Dispatch network calls to carry only the values that are displayed, with correct pagination, without changing the current flow. The `my-company` call was the easy part (one flag). The real work was replacing the three client-side list calls and moving the counts to the server.

## How
**One query for the page**: the table rows, `totalCount`, `statusCounts`, `summary` (KPI cards) and `partyNames` all come from one `/dispatches?view=table` response. Before, the two extra calls existed only to count tabs and build the party list.

**Server-side filters**: status, search, from date, to date and party are query params. Changing any of them resets to page 1. Search waits 350 ms after typing before it calls the server.

**Same record shape**: the table and all modals still receive a `DispatchRecord`. Only the source changed, so no modal needed edits. The fields that are no longer sent (`company_id`, `created_by`, signature URL and so on) were not used by the page.

**Kept the same on purpose**: the Pending tab still maps to `DRAFT`, the KPI cards still count every dispatch whatever the filters are, and the date pickers, reset button and approve / gate pass / invoice flows are untouched.

## Decisions made
**Party filter sends the party name.** The dispatch rows only carry the party name, not a party id, so the filter and the dropdown value are the name. Using an id would have needed more fields on every row.

**Reused the existing table pagination.** `DataTable` already supports `manualPagination` (used by Order Booking), so no new pagination UI was added.

**Removed the old list function.** `fetchDispatchList` and `dispatchListQueryOptions` had no other users, so they were replaced instead of kept as dead code.

## Notes
- `FinalInvoicePreviewModal` already gated its company fetch with `enabled: isOpen`, so only the Gate Pass modal needed the fix.
- A dispatch with no party name no longer shows up as a "Customer" option in the party dropdown (the server skips empty names).
- Line endings in the touched files are unchanged (CRLF).

## Next steps
- Open the Dispatch page against the updated backend and confirm the network tab shows only the one `/dispatches?view=table...` call and no `my-company` call.

## Test plan
**Automated tests (no test cases for this module exist):**
- [x] Type check (`npx tsc --noEmit -p tsconfig.app.json`): no errors.
- [ ] No Dispatch test file exists, so no unit tests were added or run.

**Manual checks (local):**
- [x] Read the two HAR files and compared the Dispatch calls with the Order Booking call (`view=table&page=1&limit=10&status=DRAFT`).
- [ ] Load the Dispatch page in the browser and check the network calls, tab counts, KPI cards, pagination and filters - not run yet, needs the backend restarted with the matching backend PR.
