# 2026-11-25 - admin-ui-polish (admin)

## Summary

The admin panel list pages were built one by one, so each looked a little different. Tables had no shared footer or loading state, row actions were loose buttons, filters were plain grids of dropdowns, the summary cards were only on some pages, and text and controls were large and not consistent. This PR gives every list page the same building blocks and tightens the overall look (smaller type, smaller controls, a slightly darker page background so white tables stand out).

The Subjects page also gets a new card layout with collapse, search and drag to reorder. The exam editor's question list now uses the same table, footer and actions menu as the other pages.

Only `learning-app-admin` changed. Backend, web and mobile have no changes for this work, so there are no PR files for them. No API shape changed.

**The "Show archived" checkbox on the Subjects page was removed on purpose** - see "Decisions made" below. This was a choice, not a miss.

Nothing was checked in a browser. The dev server needs a login and I could not sign in from here. Everything below was checked by typecheck, unit tests and a production build only.

## What changed

Files with no real change are marked so you can skip them.

| File | Change |
|---|---|
| `src/components/common/data-table.tsx` | Added `loading`, `loadingLabel`, `footer`, `fill` and `empty` props. Headers stay while loading, the book loader shows over the body, old rows are hidden behind it. `fill` makes the table take the page height with sticky headers and a pinned footer. |
| `src/components/common/data-table.test.tsx` (new) | Tests for the loading and refresh behaviour. |
| `src/components/common/table-footer.tsx` (new) | One footer for every table: "Showing x to y of z", rows per page, page buttons. Replaces the old pagination bar. Has an optional `id` so two footers can sit on one page. |
| `src/components/common/pagination.tsx` (deleted) | Replaced by `TableFooter`. Nothing imports it any more. |
| `src/components/common/actions-menu.tsx` (new) | Three-dot menu used for row actions (items can be links, buttons, or destructive). |
| `src/components/ui/dropdown-menu.tsx` (new) | shadcn dropdown menu that `ActionsMenu` is built on. |
| `src/components/common/filter-bar.tsx` (new) | White bar above every list: filter tabs with counts on the left, then search, dropdown filters and a Reset button on the right. |
| `src/components/common/filter-bar.test.tsx` (new) | Tests for tab counts, search typing and the Reset button. |
| `src/components/common/stat-card.tsx` (new) | `StatCard` and `StatCardGrid` for the summary cards. Shows a dash while the number is loading. |
| `src/components/common/stat-card.test.tsx` (new) | Tests for the card. |
| `src/components/common/empty-state.tsx` | New `bare` option (no border) so it can sit inside a table frame. Title and description slightly smaller. |
| `src/components/common/query-status.tsx` | New `hideLoader` option for pages whose table shows its own loader. Errors still show. |
| `src/components/common/page-header.tsx` | Slightly smaller header. Action buttons in the header are sized the same. |
| `src/components/common/select-input.tsx` | Small addition for the filter bar. |
| `src/features/content/pages/questions-page.tsx` | Summary cards, `FilterBar` (status tabs with counts, search, seven dropdowns, Reset), shared table with footer and loading, "Actions" column with a menu. |
| `src/features/content/pages/subjects-page.tsx` | Rebuilt around cards (see below), SAT/ACT tabs on top, summary cards, a `FilterBar` with search and Reset. "Show archived" removed. |
| `src/features/content/components/subject-card.tsx` (new) | One card per subject with its categories. Can be collapsed. |
| `src/features/content/components/category-row.tsx` (new) | One category row with its question counts and actions menu. |
| `src/features/content/components/drag-handle.tsx` (new) | Grip for drag reorder. Up and down arrow keys also move the row. |
| `src/features/content/hooks/use-drag-reorder.ts` (new) | Drag and drop state for subjects and categories. |
| `src/features/content/api/content.queries.ts` | Added `useQuestionCounts` (published, draft, archived, total) for the cards and tab counts. |
| `src/features/content/api/content.api.ts` | The question list call now lets the caller pass `pageSize` (before it was always 20). |
| `src/features/exams/pages/exams-page.tsx` | Summary cards, `FilterBar` (status tabs, search, Exam dropdown), shared table. Delete flow with a confirm dialog and an "Actions" menu (Edit, Publish or Withdraw, Delete). |
| `src/features/exams/api/exams.queries.ts` | Added `useExamCounts` (published, draft, withdrawn, total). |
| `src/features/exams/api/exams.api.ts` | Optional `pageSize` on the list call. |
| `src/features/exams/components/section-editor.tsx` | The question list in each section is now a table with `#`, `Question` and `Actions` (Move up, Move down, Remove), a footer with paging, and an empty state. Section Remove moved into an actions menu. |
| `src/features/exams/components/question-picker.tsx` | Uses `TableFooter` with a rows per page choice. |
| `src/features/exams/pages/exam-editor-page.tsx` | Removed the narrow `max-w-3xl` so the page matches the others. The save bar matches the table frame. |
| `src/features/exams/components/blueprint-preview-dialog.tsx`, `duration-field.tsx`, `src/features/exams/lib/exam-editor.ts` | Text weight and formatting only. No real change. |
| `src/features/audit/pages/audit-page.tsx` | Summary cards (total, today, failed), `FilterBar` (All, Success, Failed tabs with counts plus a Domain dropdown), shared table, "Actions" menu with View details. |
| `src/features/audit/api/audit.queries.ts` | Added `useAuditCounts`. The list keeps the previous page while loading the next one. |
| `src/features/audit/api/audit.api.ts`, `components/copy-chip.tsx` | Formatting only. No real change. |
| `src/features/audit/lib/audit-formatters.ts` | The label for `exam.config.delete` is now "Delete Exam" (was "Archive Exam"). Rest is formatting. |
| `src/features/cooldown/pages/cooldown-page.tsx` | Summary cards, `FilterBar` (All, Customized, Default tabs with counts, search by exam version name, Reset), shared table and footer, "Actions" menu. Search is done in the browser on the loaded list. |
| `src/features/cooldown/components/cooldown-dialog.tsx`, `schemas/cooldown.schemas.ts`, `api/cooldown.queries.ts` | Formatting and text weight only. No real change. |
| `src/features/analytics/pages/usage-section.tsx` | The local `Highlight` card was replaced by the shared `StatCard`. |
| `src/features/analytics/components/recent-lists.tsx` | Uses `TableFooter` (with rows per page) instead of `Pagination`. |
| `src/features/analytics/pages/*.tsx`, `components/analytics-filter-bar.tsx`, `analytics-report-note.tsx` | Small type size and weight changes. No real change. |
| `src/components/analytics/*.tsx` | Small type size and weight changes (KPI number is a bit smaller). No real change. |
| `src/components/ui/button.tsx`, `input.tsx`, `select.tsx`, `card.tsx`, `table.tsx`, `field.tsx`, `dialog.tsx`, `alert-dialog.tsx`, `alert.tsx`, `calendar.tsx`, `popover.tsx`, `tabs.tsx`, `textarea.tsx`, `chart.tsx` | Smaller controls and type: buttons 32px high with 13px text, inputs and selects 36px high with 14px text, rounder corners reduced from `xl` to `lg`, card titles semibold. |
| `src/components/common/app-sidebar.tsx`, `auth-card.tsx`, `section-tab.tsx`, `section-tabs.tsx`, `markdown-preview.tsx`, `options-editor.tsx` | Smaller type and spacing. No real change in behaviour. |
| `src/layouts/dashboard-layout.tsx` | On desktop the page is fixed to the screen height and the main area scrolls inside, so tables can fill the space. Smaller logo, name and avatar. |
| `src/pages/dashboard-page.tsx` | Smaller card numbers. |
| `src/features/content/pages/import-page.tsx`, `question-detail-page.tsx`, `components/question-form.tsx`, `components/image-uploader.tsx` | Text weight and formatting only. No real change. |
| `src/index.css` | New colours `--table-header`, `--row-hover` and a darker `--page`, `--secondary`, `--muted`, `--border`, `--input`. Base text is now `text-sm`. The quote style and indent of the file were also reformatted, so the diff looks bigger than it is. |
| `src/assets/animations/open-book.json` | Only reformatted (it was one line, now pretty printed). The animation itself is the same. |
| `src/test/setup.ts` | Added the pointer capture and `scrollIntoView` stubs that jsdom is missing, so the new dropdowns can open in tests. |
| `src/features/audit/pages/audit-page.test.tsx`, `src/features/audit/lib/audit-formatters.test.ts`, `src/features/content/pages/content-pages.test.tsx`, `src/features/exams/pages/exams-pages.test.tsx` | Updated for the actions menu, the "Delete Exam" label, the new search box on Subjects, and the extra count calls. |

## Why

The pages grew separately and the same things were copied around: a footer, a menu for row actions, a filter row, a stat card. Fixing the look of one page meant fixing it five times. The goal is one component for each of these, then using it everywhere, so the next page gets the same look for free.

The easy part was swapping each page over once the components existed. The real work was the table: it has to keep its headers while loading, fill the page height, keep the footer pinned, and show an empty state inside the frame, all without the page itself scrolling.

## How

**Table**: `DataTable` now owns the frame. With `fill` it grows to the bottom of the page, only the rows scroll, headers are sticky and the footer stays under it. While `loading` is true the headers stay and the book loader covers the body. If there are no rows and an `empty` node is given, it is shown centred under the headers.

**Footer**: `TableFooter` shows the range ("Showing 1 to 20 of 134 questions"), a rows per page dropdown and page buttons. It replaces `Pagination` everywhere, including the analytics recent lists and the question picker.

**Row actions**: every row action menu is an `ActionsMenu` (three dots) and the column header is "Actions" on every table. The menu button has a name like "Actions for SAT Practice Test 1" so it is clear for screen readers and tests.

**Filter bar**: `FilterBar` takes `tabs`, `search`, children (the dropdowns), and `onReset`. The Reset button is greyed out until something is filtered (`canReset`). The tabs show the counts, for example "Published (6)".

**Counts**: the cards and the tab counts come from one hook per page (`useQuestionCounts`, `useExamCounts`, `useAuditCounts`). Each asks the existing list endpoint for one row and reads the `total`, so no new backend route is needed and no list is downloaded just to count it. These counts do not change when the page filters change, except the Exam dropdown on Questions and Exams, which narrows them to SAT or ACT.

**Subjects page**: SAT and ACT stay as tabs at the top. Each subject is a card that can be collapsed. Subjects and categories can be reordered by dragging the grip or with the arrow keys. Reorder only sends the positions that actually changed. The search matches subject names and category names. A matching subject shows all its categories, otherwise only the matching ones are shown. While searching, reorder is switched off (the list is not complete, so a drop would give wrong positions).

**Left the same on purpose**: no API paths, request bodies or response shapes changed. The question list `pageSize` default is still 20.

## Decisions made

**Removed "Show archived" on Subjects.** The request was to remove the checkbox completely. Because the page no longer asks for archived items, an archived subject or category now disappears from the page and cannot be restored from the UI. Archive and Restore still exist in the menus, but Restore can never be reached. I did not add another way to see archived items because that was not asked for. See "Next steps".

**Counts use the list endpoint.** I did not add a backend count route. It keeps this PR frontend only. The cost is a few small extra requests per page.

**Cooldown search is in the browser.** The cooldown list is loaded in full, so filtering by name on the client is simple and needs no API change.

**Smaller type everywhere.** Base text is 14px and buttons 13px. I did not make this a setting. It is a global change in the shared `ui` components, so it touches every page.

## Notes

- `open-book.json` and `index.css` show many changed lines only because they were reformatted. Use "ignore whitespace" when reviewing them.
- Git warns that LF will become CRLF on many files. This is the repo's `autocrlf` setting and not part of this change.
- The search box on every filter bar is `type="search"`. Some browsers show a small clear (x) button inside it.

## Next steps

- Decide how archived subjects and categories should be shown and restored on the Subjects page (a filter tab such as "Archived" would fit the new filter bar).
- Look at every changed page in a browser, in particular the Subjects page, the exam editor and small screens.
- Fix the existing lint warnings (see "Test plan") in a separate change.

## Test plan

**Automated tests (all 125 passing, 20 test files):**
- [x] `src/components/common/data-table.test.tsx`: keeps headers and shows the loader when there are no rows yet; keeps previous rows visible while refreshing; shows no loader once loaded.
- [x] `src/components/common/stat-card.test.tsx`: shows the heading, value and description; shows a dash while the value is missing; is not a link or button.
- [x] `src/components/common/filter-bar.test.tsx`: shows counts, marks the chosen tab and reports a change; reports what is typed in the search box; disables Reset until a filter is active.
- [x] `src/features/content/pages/content-pages.test.tsx`: archive from the actions menu for a subject and a category; searches categories by name and folds a subject away.
- [x] `src/features/exams/pages/exams-pages.test.tsx`: Withdraw and Publish from the actions menu; status shown inside the table.
- [x] `src/features/audit/pages/audit-page.test.tsx` and `src/features/audit/lib/audit-formatters.test.ts`: View details from the actions menu; "Delete Exam" label; list answered by URL because the cards also call the API.
- [x] Type check (`npx tsc -b`): no errors.
- [x] Production build (`npx vite build`): builds. It prints a warning about direct `eval` from a dependency.
- [ ] Lint (`npx oxlint`): runs with warnings and was not cleaned. Only one is in a file added here (`table-footer.tsx`, fast refresh export rule). The others are in older files such as `analytics-composed-chart.tsx` and `cooldown-dialog.tsx`.

**Manual checks (local):**
- [ ] Every changed page in a browser (Questions, Exams, exam editor, Subjects, Cooldown, Audit, Analytics Usage, Dashboard) - not possible, the dev server at `localhost:5174` needs a login I could not enter. The unit tests cover the logic, not the look.
- [ ] Drag to reorder subjects and categories - not checked by hand. The tests cover the keyboard and search paths only.
- [ ] Loading and empty states inside the table frame - not seen on screen, only covered by the `DataTable` tests.
