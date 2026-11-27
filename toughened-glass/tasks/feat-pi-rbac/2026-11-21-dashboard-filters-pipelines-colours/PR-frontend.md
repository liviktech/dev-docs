# 2026-11-21 - dashboard-filters-pipelines-colours (frontend)

## Summary

The Overview dashboard had a party picker (and sometimes a sales rep picker) inside almost every section, the Work Order and Dispatch sections had a status pie that repeated their pipeline bars, the sections did not share one colour pattern, and the Final Invoice cards failed with an error as soon as a party was picked. This PR moves the party filter to the header so one party applies to every section, removes the repeated pies, gives every pipeline the Enquiry colours, adds the missing Enquiry "Open" bar (and Work Order "Rejected" and PI "Declined" bars), orders the sections like the sidebar, and cleans the extra text out of the PI and Final Invoice cards. It depends on the backend PR of the same name for the new `open` and `declined` figures and for the Final Invoice party fix. The type check passes (`npx tsc --noEmit`). Nothing was checked in the browser by me.

**A party breakdown pie for Work Order and Dispatch is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/dashboard/components/dashboard-filter-bar.tsx` | Added an "All Parties" picker on the left of the date range. It writes the `party` value in the URL, which every section already reads. |
| `src/features/dashboard/use-dashboard-filters.ts` | Removed `obParty`, `woParty` and `dpParty` (the per-section parties). Reset and the "is default" check no longer look at them. |
| `src/features/dashboard/components/sales-rep-filter.tsx` (new) | The sales rep picker only, for the Enquiry section. |
| `src/features/dashboard/components/rep-party-filters.tsx` (deleted) | Replaced by the file above. Party is now in the header, and PI and Final Invoice no longer have a rep picker. |
| `src/features/dashboard/components/order-booking-filters.tsx` | Order type picker only. The party picker was removed. |
| `src/features/dashboard/sections/enquiry-section.tsx` | Uses the rep picker. Added the Open bar. The Approved bar now opens the list on the Approved tab only (it used to ask for Approved and Converted together). |
| `src/features/dashboard/sections/order-booking-section.tsx` | Takes the party from the header. Pipeline colours changed (see "How"). The file also shows about 260 changed lines that are only the editor reformatting it (quotes, semicolons, line breaks). No real change in those lines. |
| `src/features/dashboard/sections/work-order-section.tsx` | Removed the status pie, its code and the section's party picker. Two column grid. Added a Rejected bar when there are any. Pipeline colours changed. |
| `src/features/dashboard/sections/dispatch-section.tsx` | Removed the status pie, its code and the section's party picker. Two column grid. Pipeline colours changed. |
| `src/features/dashboard/sections/pi-section.tsx` | Removed the section filters, the card titles and the "Total" badges. Strips the sales rep from its request. Added a Declined bar. Pipeline colours changed. |
| `src/features/dashboard/sections/final-invoice-section.tsx` | Same clean up as PI: no section filters, card titles or "Total" badges, and the sales rep is stripped from its request. |
| `src/features/dashboard/registry.ts` | Section order is now Enquiry, Order Booking, Work Order, PI, Final Invoice, Dispatch (the sidebar order). Dispatch used to sit before PI. |
| `src/features/dashboard/api.ts` | Added `open` to the enquiry funnel type and `declined` to the PI pipeline type. |

## Why

The dashboard should be filtered once, at the top, the same way the date range is. Per-section party pickers made it easy to look at different parties in different sections by accident. The pies in Work Order and Dispatch showed the same numbers as the bars next to them. The bars also needed to match what the list tabs show, because a click opens that list.

## How

**One party for the whole dashboard**: the header picker sets `party`. Enquiry, PI, Final Invoice and Dispatch already used that value. Order Booking and Work Order now use it too instead of their own. Each section still strips the sales rep from its request, because the rep picker belongs to Enquiry only.

**Colours**: every pipeline now goes cyan, green, purple, grey from top to bottom, like the Enquiry pipeline. The trend line is purple in every section, like Enquiry's "Converted". Amber and rose are no longer used in the section pipelines, except the new Enquiry "Open" bar which is amber.

**Counts match the list tabs**: each bar counts one status, so a click opens the matching tab. Enquiry now reads Received = Open + Approved + Converted + Closed. PI has a Declined bar because the PI Status list has a Declined tab. Work Order shows a Rejected bar only when there are rejected orders.

**Kept the same on purpose**: the amounts and percentages beside the PI and Final Invoice pie slices are still shown. They are the data on those cards, unlike the name only pies in the other sections.

## Decisions made

**The Work Order and Dispatch pies were removed, not replaced.** The first plan was to turn them into a pie by party. It needs new backend queries, and it was started and then taken back out when the request changed to "remove the pie and balance the two cards". The sections now have the pipeline and the trend only.

**Colours follow the position of the bar, not the meaning.** So "Completed" in Work Order is grey and "Draft" is green. This matches how Enquiry looks, which is what was asked for. If the meaning based colours are wanted back, only the `COLORS` constants in each section change.

**Section order follows the sidebar.** Dispatch is last because it is under Billing in the sidebar, after Final Invoice.

## Bugs found while testing (already there before, not fixed here)

**1. Order Booking bars do not cover every status.** The orders table allows `CANCELLED` and `COMPLETED`, but "Total Received" counts them while there is no bar for them. With today's data (6 draft + 19 confirmed = 25) it adds up, so nobody sees it yet.

**2. Work Order has two statuses with no bar.** `PENDING_APPROVAL` and `APPROVED` are counted in the total but not in any bar. Approving a work order moves it straight to In Progress, so they should be rare.

## Notes

- The Prev. period line showing 0 is expected here. It compares with an equal length range just before the selected one (about 11 to 31 Oct), and there is no data in that range yet. It plots from the same data as the bars, so it will show up once older records exist.
- The 2026-11-20 task (`work-order-dispatch-dashboard`) describes the status pie and the per-section party picker. This PR replaces both, so read that doc as the history.

## Next steps

- Decide whether Order Booking (Cancelled, Completed) and Work Order (Pending Approval, Approved) need their own bars.
- If a party breakdown is wanted later, add it in the backend first and then add one card back.

## Test plan

**Automated tests (no new tests added):**
- [x] Frontend type check (`npx tsc --noEmit`): passes after every change.
- [ ] No test file covers the dashboard sections, so none was run or added.

**Manual checks (local):**
- [x] The Final Invoice party filter was run against the local database with party `PTY-20261111-110` and returned 3 invoices (see the backend PR).
- [x] The 500 error on the Final Invoice request was seen in the browser DevTools Network tab before the backend fix.
- [ ] Header party picker, the new bars and the colours in the browser after the changes - not checked by me, needs a look on the running app.
- [ ] Clicking each bar and checking the list tab shows the same count - not checked.
