# 2026-11-20 - work-order-dispatch-dashboard (backend)

## Summary

The Overview dashboard only had Enquiry and Order Booking sections. This PR adds the two API routes the Work Order and Dispatch sections need, `GET /dashboard/work-order` and `GET /dashboard/dispatch`, built the same way as the Order Booking one (routes, controller, service, repository, schema, types). Both routes are read only. Both type-check clean with `tsc --noEmit`. I did not hit the routes against a real database or run any automated tests.

**Overdue and due-soon dispatch figures are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/dashboard/dashboard.routes.ts` | Registers `GET /work-order` (needs `WORK_ORDER_VIEW`) and `GET /dispatch` (needs `DISPATCH_VIEW`), each with its own query validation. |
| `src/modules/dashboard/work-order/dashboard-work-order.schema.ts` (new) | Query schema: `from`, `to`, `granularity`, `compare`, `party`. Same rules as the other sections, including the max period check (reuses `MAX_PERIOD_DAYS` from the enquiry schema). |
| `src/modules/dashboard/work-order/dashboard-work-order.types.ts` (new) | Response shape: `period`, `previousPeriod`, `trend`, `previousTrend`, `pipeline`, `breakdown.status`. |
| `src/modules/dashboard/work-order/dashboard-work-order.repository.ts` (new) | Three queries on `production.work_order`: created and completed per bucket, counts per status for the pipeline, and counts per status for the pie. |
| `src/modules/dashboard/work-order/dashboard-work-order.service.ts` (new) | Works out the period, the grain and the previous period, then runs the queries in parallel. |
| `src/modules/dashboard/work-order/dashboard-work-order.controller.ts` (new) | Thin handler that calls the service. |
| `src/modules/dashboard/dispatch/dashboard-dispatch.schema.ts` (new) | Same query schema as above, for dispatch. |
| `src/modules/dashboard/dispatch/dashboard-dispatch.types.ts` (new) | Response shape: `period`, `previousPeriod`, `trend`, `previousTrend`, `pipeline`. |
| `src/modules/dashboard/dispatch/dashboard-dispatch.repository.ts` (new) | Shipment counts per status, shipments created and approved per bucket, and a party name lookup. |
| `src/modules/dashboard/dispatch/dashboard-dispatch.service.ts` (new) | Works out the period, adds the PIs' pending dispatches to the shipment counts, and builds the pipeline. |
| `src/modules/dashboard/dispatch/dashboard-dispatch.controller.ts` (new) | Thin handler that calls the service. |

## Why

The dashboard should show every main module, and Work Order and Dispatch were missing. The Order Booking section already had the pattern, so most of the work was fitting each module's real data and dates to it, so a click on the dashboard opens the same rows in the list.

## How

**Work order dates**: Counts use the work order's application date (`application_date`), the same field the Work Orders list filters on. Pipeline numbers are exact status counts (Draft, In Progress, Completed, Cancelled) so each one matches a tab in the list.

**Dispatch dates**: Counts use the shipment's created date (`created_at`), the same field the Dispatch list filters on. The shipments are reached through the same joins the list uses (final invoice, PI dispatch, proforma invoice), so the party filter works the same way.

**Dispatch pending count**: The list's Pending tab holds both draft shipments and the pending dispatch of each PI. The service adds `listPendingParentRows` (already used by the list) to the draft shipment count, so Pending, Approved and Postponed add up to Total exactly like the list tabs. Cancelled is returned separately because the list's All tab leaves it out.

**Party filter**: Both routes take a party code. The Dispatch list works by party name, so the service looks the name up once to filter the pending dispatches.

**Trend**: The trend is zero filled per bucket (day, week or month). For Dispatch it counts shipments only, not the PIs' pending dispatches, because a pending dispatch turns into a shipment when it is approved and would be counted twice.

## Decisions made

**Overdue, due-soon, pieces and gate pass figures were built and then removed.** I first returned overdue and due-in-7-days dispatches, pieces dispatched and gate pass counts. They did not fit the card design the other sections use, so I removed the code instead of leaving it unused. They can be added back if a design is agreed.

**Dashboard counts use the same date as each list.** I did not use one common date for all modules, because then a click on the dashboard would open a list with different rows than the number shown.

## Notes

- Restart the backend dev server to pick up the new routes.
- The compiled `dist` folder already had an old work order dashboard with no matching source. I read it for reference only and wrote the new code from the current schema.

## Test plan

**Automated tests (none run):**
- [ ] No unit tests were added or run for the two new modules. The Order Booking dashboard route has the same gap in this diff.
- [x] `tsc --noEmit`: passes with no errors.

**Manual checks (local):**
- [ ] `GET /dashboard/work-order` and `GET /dashboard/dispatch` against a real database - not checked. Only the type check is done, so the SQL is not proven yet.
- [ ] Check that the dispatch Pending count equals the Pending tab count in the Dispatch list for the same dates and party - not checked.
- [ ] Check a user without `WORK_ORDER_VIEW` or `DISPATCH_VIEW` gets 403 - not checked, the guard is the same `requireRight` used by the other sections.
