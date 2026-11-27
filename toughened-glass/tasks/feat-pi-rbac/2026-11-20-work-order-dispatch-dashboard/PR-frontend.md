# 2026-11-20 - work-order-dispatch-dashboard (frontend)

## Summary

The Overview dashboard only had Enquiry and Order Booking sections. This PR adds a Work Order section and a Dispatch section, each with three cards laid out exactly like the Order Booking section: a status pie, a pipeline bar chart and a trend chart. Both sections use the existing `DashboardSection`, `DashboardCard`, `BreakdownChart` and `TrendChart` components, so no new component or design was added. The project type-checks clean with `tsc --noEmit`. I did not open the dashboard in a browser, so the screens are not checked live.

**Overdue and due-soon dispatch figures are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

**`src/features/roles/rights-picker.tsx` is not part of this PR on purpose.** It is staged in this repo with formatting and colour changes that are unrelated to the dashboard, so it should go in its own commit.

## What changed

| File | Change |
|---|---|
| `src/features/dashboard/sections/work-order-section.tsx` (new) | Work Order section: status pie, pipeline bars (Total Created, Draft, In Progress, Completed) and a Created vs Completed trend. Own party filter. |
| `src/features/dashboard/sections/dispatch-section.tsx` (new) | Dispatch section: status pie (Pending, Approved, Postponed, Cancelled), pipeline bars (Total, Pending, Approved, Postponed) and a Created vs Approved trend. Own party filter. |
| `src/features/dashboard/api.ts` | Adds the `WorkOrderDashboard` and `DispatchDashboard` types and the two API calls. |
| `src/features/dashboard/queries.ts` | Adds a query option for each new section, with the same caching as the others (one minute, keeps the previous data while loading). |
| `src/features/dashboard/registry.ts` | Adds the two sections, shown only for `WORK_ORDER_VIEW` and `DISPATCH_VIEW`. |
| `src/features/dashboard/use-dashboard-filters.ts` | Adds `woParty` and `dpParty` to the dashboard URL filters, so each section has its own party and does not change the others. The Reset button also clears them. |

## Why

The dashboard should show every main module, and Work Order and Dispatch were missing. The user asked for the same look as the existing sections, so the work was reusing the existing components, not designing new ones.

## How

**Click through**: Clicking a pie slice, a bar or a trend point opens the module's list with the same period, party and status tab, so the list shows the rows that were counted. The Work Orders list gets `from`, `to`, `party` and `status`. The Dispatch list gets `from`, `to`, `party` (by name) and `status`.

**Dispatch party**: The Dispatch list filters party by name, not code. The section looks up the name of the picked party from the party list before it opens the list.

**Work order pie**: Only Draft, In Progress, Completed and Rejected have a tab in the Work Orders list. Pending Approval and Approved slices are shown but not clickable.

**Party filter**: Each section draws its own party picker inline with the existing `OptionSelect`, the same way the Order Booking section does. Nothing else in the section header was added.

## Decisions made

**The Dispatch section went through two designs and ended on the plain one.** First version had a tile card (overdue, due in 7 days, pieces, gate passes) and then a bar chart of the same figures. The user asked for the previous design only, so both were removed and the pie, pipeline and trend layout of Order Booking was used. Those figures are not shown anywhere now.

**A separate party filter component was not kept.** I first made a small shared component for it, then deleted it and put the picker inline in each section so no new component file is added.

## Notes

- The backend routes in the backend PR must be running, and the backend dev server must be restarted to pick them up.
- Dates on the dashboard follow the module lists: application date for Work Orders and created date for Dispatch.

## Test plan

**Automated tests (none run):**
- [ ] No unit tests were added or run for the two new sections.
- [x] `tsc --noEmit -p tsconfig.app.json`: passes with no errors.

**Manual checks (local):**
- [ ] Open the dashboard with a user who has both Work Order and Dispatch view and check the two sections show - not checked in a browser.
- [ ] Click each pie slice, bar and trend point and check the list opens with the same rows - not checked in a browser.
- [ ] Pick a party in each section and check the other sections do not change - not checked in a browser.
- [ ] Open the dashboard with a user who has neither right and check the sections are hidden - not checked, the registry uses the same rights check as the other sections.
