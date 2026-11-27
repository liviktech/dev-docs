# 2026-11-27 - user-analytics-theme (admin)

## Summary

User Analytics was seven tabbed sections with a shared filter bar. This PR replaces them with one page that shows parent and child registrations per month, filtered by year and month. It also changes the admin panel from a blue theme to an orange theme with all colours in one file, adds "Back to list" buttons, and removes some filters from the Questions page.

**The build is currently failing.** `npm run build` and `npm run typecheck` stop on one unused variable, `categoryOptions` in `src/features/content/pages/questions-page.tsx` (line 102), left behind when the category filter was removed. Fix this before merging.

**This PR needs the matching backend change.** The new page calls `GET /admin/analytics/registrations`, and the backend change also removes the endpoints behind the old sections. The two should be released together.

## What changed

| File | Change |
|---|---|
| `src/features/analytics/pages/user-analytics-page.tsx` (new) | The new single User Analytics page: year and month filters, four summary cards, a monthly parent and child bar chart, and a month comparison table. |
| `src/features/analytics/api/analytics.api.ts`, `api/analytics.queries.ts` | Added the registrations request and hook. Removed the growth, engagement, usage, features, academic and parents ones. |
| `src/features/analytics/schemas/analytics.schemas.ts` | Added the registrations response schema. Removed the schemas for the deleted sections. |
| `src/features/analytics/lib/filters.ts` | Removed the per section list of unsupported filters. |
| `src/features/analytics/pages/` (7 section files and `user-analytics-layout.tsx`, deleted) | The old tabbed sections and their layout. |
| `src/features/analytics/components/analytics-filter-bar.tsx`, `hooks/use-analytics-filters.ts` (deleted) | The old shared filter bar and its hook. |
| `src/components/analytics/` (composed chart, date filter, heatmap, stacked bar chart, deleted) | Chart parts used only by the old sections. |
| `src/routes/routes.tsx` | `/user-analytics/*` now loads the new page (still on demand). Child routes removed. |
| `src/theme.css` (new) | All theme colours for light and dark, now orange, plus colours for the student and parent user types. |
| `src/index.css` | Colour values moved out to `theme.css`, which it now imports. Added the two user type colour tokens. |
| `src/features/audit/pages/audit-page.tsx` | The content action badge uses the theme colour instead of a fixed blue. |
| `src/features/content/pages/question-form-page.tsx` | "Back to list" button on the Add a question page. |
| `src/features/exams/pages/exam-editor-page.tsx` | "Back to list" button on the exam editor, next to the status badge. |
| `src/features/content/pages/questions-page.tsx` | Removed the category, explanation and exposure filter dropdowns. |

## Why

The User Analytics module needed to answer one question simply: how many parents and children signed up, and how that compares with the month before. The theme change moves the panel to the orange brand colour and puts every theme colour in one place.

## How

**One request for the page**: the server returns counts already grouped by month, so no individual user records are loaded.

**Filters in the URL**: year and month are kept as query parameters, so a shared link shows the same view. Invalid values fall back to the current year and all months.

**Growth badge**: shows a percentage change. When the previous month had no sign ups there is no percentage, so it shows "New" or "No change".

**Old links**: the route ends in `/*`, so old addresses such as `/user-analytics/growth` open the new page.

**Theme tokens**: components keep reading the same token names, so only the values changed.

## Decisions made

**One page instead of trimming the tabs.** The engagement, usage, feature, academic performance and parent analytics views are removed from the panel, not hidden. Reviewers should confirm that nobody still relies on them.

## Notes

- Only the dropdowns were removed from the Questions page. The related state and query code for category, explanation and exposure is still in the file and could be cleaned up with the build fix.
- No test was added for the new User Analytics page.
- `npm run format:check` reports style issues in 142 files. This was not compared against `dev`.
- `npm run lint` shows warnings only. The `categoryOptions` one comes from this PR.

## Next steps

- Remove the unused `categoryOptions` (and the leftover filter code), then run the build again.
- Add a test for the User Analytics page.

## Test plan

**Automated tests (125 passing in 20 files, build failing):**
- [x] `npm run test`: 20 test files, 125 tests passed
- [ ] `npm run typecheck`: fails with TS6133, `categoryOptions` is declared but never read
- [ ] `npm run build`: fails on the same error
- [x] `npm run lint`: warnings only
- [ ] `npm run format:check`: reports 142 files

**Manual checks (local):**
- [ ] User Analytics page against the backend (year and month filters, cards, chart, comparison table) - not run
- [ ] Orange theme in light and dark mode, including text contrast - not run
- [ ] "Back to list" buttons and the Questions page filters in the browser - not run
