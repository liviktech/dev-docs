# 2026-11-25 - date-picker-current-month-default (backend)

## Summary

There are no backend code changes in this PR. All changes in this PR were implemented in the frontend.

## What changed

No backend files were touched in this PR.

## Why

The dashboard already opens on the current month and users expected the same from every screen with the same picker. Opening on all dates also made lists and KPI cards heavier and less useful by default.

The easy part was the helper and the per-screen default. The real work was finding how each screen holds its date state: URL-backed lists, plain local state, a route-level state (Order Booking) and the shared report page hook. The Reset buttons and "active filters" checks had to be checked on each so that the new default did not look like a changed filter.

## How

**Order Booking and Work Orders**: the range is read from the URL when present and falls back to the current month otherwise. Order Booking's list writes the range into the URL, so its URL now carries `from` and `to` even for the default month.

**Reports**: the starting filters get the month dates. The filter-bar Reset on the reports already kept the date range, so it works as before.

## Decisions made

**Department Daily report still opens on today.** It is a daily summary and already started on today, so it is not "all dates". Not picked: moving it to the month.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- Existing bookmarks and shared links that carry their own `from` / `to` keep working; only links without dates change what they show.
- Order Booking's URL now always carries the dates, so the address bar is longer than before. This does not change behaviour.
- The default month runs from the 1st to today, not to the end of the month, the same as the dashboard.

## Next steps

- Decide whether the Department Daily report should also open on the month.

## Test plan

**Automated tests:**
- [x] Backend type check (`npx tsc --noEmit`): passes with no errors.

**Manual checks (local):**
- [x] Backend API endpoints and database operations verified.
