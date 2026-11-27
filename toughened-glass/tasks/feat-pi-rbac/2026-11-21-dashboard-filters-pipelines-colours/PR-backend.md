# 2026-11-21 - dashboard-filters-pipelines-colours (backend)

## Summary

The dashboard's Enquiry pipeline had no "Open" figure and its "Approved" figure counted Approved and Converted together, so it showed 8 while the Enquiries list's Approved tab showed 3. The PI pipeline left out Declined PIs, so its bars did not add up to the total. The Final Invoice section failed with a 500 error as soon as a party was picked. This PR fixes all three. The frontend PR of the same name uses the new figures. The type check passes (`npx tsc --noEmit`). The Final Invoice party condition was run against the local database and returned rows. The Enquiry and PI queries were not run, only type checked.

## What changed

| File | Change |
|---|---|
| `src/modules/dashboard/enquiry/dashboard-enquiry.repository.ts` | The funnel query now returns `open` (status OPEN). `approved` now counts status APPROVED only (it used to be APPROVED and CONVERTED). |
| `src/modules/dashboard/enquiry/dashboard-enquiry.types.ts` | Added `open` to `EnquiryFunnel` and fixed the comment on `approved`. |
| `src/modules/dashboard/pi/dashboard-pi.repository.ts` | The pipeline query now returns `declined` (status DECLINED). |
| `src/modules/dashboard/pi/dashboard-pi.types.ts` | Added `declined` to `PiPipeline`. |
| `src/modules/dashboard/final-invoice/dashboard-final-invoice.repository.ts` | The party condition now compares only `pi.party_code`. It used to also check `p.id`, a column that does not exist. |

## Why

A bar on the dashboard has to match the tab of the list it opens. Enquiry "Approved" was the only bar that mixed two statuses, which is why 8 did not match 3. PI had a Declined tab in its list but no figure for it. The Final Invoice error was a plain bug: Postgres rejected the whole query, so all three cards failed together.

## How

**Enquiry funnel**: one status per figure. `open + approved + converted + closed` now equals `received`.

**PI pipeline**: `declined` is counted next to `cancelled`, so `pending + approved + closed + cancelled + declined` equals `total`.

**Final Invoice party filter**: `master.party` has no `id` column (its key is `party_code`, checked in the database). The condition is now `pi.party_code = $n`, the same rule every other dashboard section uses. It is shared by all the Final Invoice queries (trend, pipeline, payment, breakdown), so one change fixes all three cards.

**Kept the same on purpose**: the response keys that already existed, so the frontend only had to add the two new fields.

## Decisions made

**The first Final Invoice fix was a wrong guess.** The error looked like Postgres being unable to decide the type of the party parameter (it was used bare in one place and cast in another), so every use was cast to text. The browser showed the same 500, because the real cause was the missing `p.id` column. Reading the table's columns showed it, and the condition was reduced to the one that is needed.

**A party breakdown for Work Order and Dispatch was started and removed.** Queries for a "by party" pie were added and then taken out when the pie was dropped from the frontend. No backend change for it is left.

## Notes

- The local backend must reload (or be restarted) to return `open` and `declined`.
- The Final Invoice check was a `SELECT COUNT(*)` with the same joins and the new condition for party `PTY-20261111-110`, which returned 3. The response of the real route was not read.

## Next steps

- Add a test for the dashboard repositories. There is none today, so the Enquiry and PI counts are only type checked.

## Test plan

**Automated tests (no new tests added):**
- [x] Backend type check (`npx tsc --noEmit`): passes.
- [ ] No test file covers the dashboard queries, so none was run.

**Manual checks (local):**
- [x] Read the columns of `master.party` with `npm run db:query` and confirmed there is no `id` column.
- [x] Ran the Final Invoice joins with `pi.party_code = 'PTY-20261111-110'` on the local database and got 3 rows.
- [ ] Enquiry funnel and PI pipeline queries against the database - not run, type checked only.
- [ ] The Final Invoice cards loading in the browser after the fix - not checked by me.
