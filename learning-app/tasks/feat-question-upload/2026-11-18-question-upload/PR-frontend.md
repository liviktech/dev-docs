# 2026-11-18 - question-upload (frontend)

## Summary

The admin panel could only sign up, log in and show an empty dashboard. This PR adds the question upload screens for SAT and ACT: subjects and categories, the question list, the question form (draft and publish), and CSV and Excel import with a check before import. It also gives the whole panel a new look that follows the client's reference design (navy sidebar, blue buttons), renames the app to Prepify Admin, and adds one shared book loader that every page uses while it loads.

## What changed

| File | Change |
|---|---|
| `src/features/content/` (new) | The whole feature: `api/` (requests, TanStack Query hooks, error messages), `components/` (question form, passage dialog, subject and category dialog), `lib/` (CSV and Excel reading, row mapping), `pages/` (subjects, questions, question form, import) and `schemas/`. |
| `src/features/content/lib/bulk-file.test.ts`, `src/features/content/pages/content-pages.test.tsx` (new) | Tests for the file reading and for the subjects and questions pages. |
| `src/components/common/` (new files) | Shared pieces: `app-sidebar`, `checkbox-field`, `checkbox-input`, `confirm-dialog`, `data-table`, `empty-state`, `exam-tabs`, `file-input`, `form-dialog`, `link-button`, `markdown-preview`, `options-editor`, `page-loader`, `pagination`, `query-status`, `select-field`, `select-input`, `status-badge`, `textarea-field`. |
| `src/components/ui/` (new files) | shadcn parts: `alert-dialog`, `badge`, `checkbox`, `dialog`, `radio-group`, `select`, `skeleton`, `table`, `tabs`, `textarea`. `skeleton` is now unused. |
| `src/components/ui/button.tsx`, `src/components/ui/input.tsx` | Restyled in one place. Default buttons are blue with a darker blue hover, outline buttons have blue text and border, and buttons and inputs are 40px high with larger corners. |
| `src/components/common/page-header.tsx` | Larger, bolder title. No change in behaviour. |
| `src/components/common/loading-screen.tsx` (deleted) | The old full page spinner. It is replaced by `page-loader.tsx`. |
| `src/components/common/page-loader.tsx` (new) | The book loader. It plays the supplied Lottie file in a loop, with a line of text under it, and centres itself in the space it is given. |
| `src/components/common/query-status.tsx` (new) | The loading and error state for data a page waits on. |
| `src/assets/animations/open-book.json` (new) | The supplied Lottie animation. Only its colours were changed to the blue palette. The shapes and the page turn are the same. |
| `src/index.css` | Palette as theme variables (page background, primary hover, teal, navy sidebar colours). |
| `src/layouts/dashboard-layout.tsx` | Full height fixed sidebar with the branding, the menu and a book shape pinned to the bottom. New top bar with Log out and an avatar. The page area is wrapped in `Suspense` with the book loader, as a safety net for lazy pages. The content fills the width beside the sidebar. |
| `src/routes/routes.tsx` | The pages for the new screens are imported directly, so the heading and filters show on the first paint. Routes and paths are the same. |
| `src/routes/protected-route.tsx` | No longer replaces the page with a spinner while the session is checked, so the frame and the page stay on screen. The redirects and the error handling are the same. |
| `src/pages/dashboard-page.tsx` | Keeps its heading and uses `QueryStatus` for the admin data. |
| `src/hooks/use-debounced-value.ts` (new), `src/lib/exams.ts` (new) | A debounce hook for the search box and the exam list (SAT, ACT). |
| `src/test/setup.ts` | Adds stand-ins for canvas and `matchMedia`, which the Lottie player needs and jsdom does not have. |
| `src/components/ui/select.tsx` (new) | The dropdown opens below its field at the same width, with a 4px gap. |
| `index.html`, `README.md`, `src/features/auth/pages/login-page.tsx` | Renamed to Prepify Admin. Text only. |
| `package.json`, `package-lock.json` | Adds `papaparse` and `read-excel-file` (CSV and Excel), `lottie-react` (the loader) and `@types/papaparse`. |

## Why

The client's Nov 16 notes ask for question upload for SAT and ACT, one by one or in bulk. The panel needed screens for it. The new look comes from the client's reference image. The loader came from a request to show a proper loading state in the content area only, and to make sure every page has one, now and later. The real work was the import flow, which checks every row first and only then imports. The easy part was the screens themselves, because the backend rules are strict.

## How

**Pages use shared components**: Pages never use raw `<table>`, `<select>`, `<textarea>` or `<label>`. The shared components above are used instead. Two repeated pieces of code were replaced: `CheckboxInput` (a checkbox with a label, used for "Show archived" and "Publish the questions now") and the shared button for the "Import another file" link.

**Subjects and categories**: Exam tabs (SAT, ACT), a list of subjects with their categories, and a question count on each. Add, rename, archive, restore and delete are available. Reorder uses up and down arrows that swap the `position` of two neighbours with two `PATCH` calls. Archive and delete ask for confirmation, and the archive dialog shows how many published questions use the item.

**Questions**: A list with five filters and a search box, paging, and archive and restore. The form has linked exam, subject and category dropdowns, difficulty, calculator flag, an optional passage, the stem, two to five options with one correct, and an explanation. It has Save as draft and Publish buttons and a safe text preview.

**Import**: The browser reads the CSV or Excel file and sends plain JSON, so the server never parses spreadsheets. Max 500 rows, sent in chunks of 25. Step by step: pick the exam, download a template made from the real subject names, upload the file, see a table of rows with errors, then import the clean rows. Each passage is created once before the import and its id is sent with the rows. "Publish now" is a checkbox that is on by default.

**Loader**: A page puts `QueryStatus` where its data will appear. It shows the book loader while loading, a red box on error and nothing once the data is there. The heading and filters stay on screen, so only the table area shows the loader. A new page that uses `QueryStatus` gets the same loading state as the rest. `DashboardLayout` also keeps a `Suspense` with the book loader, so a lazy page added later gets the loader too.

**Look**: The palette is set once as theme variables in `src/index.css` and the shared button, input and dropdown components read from it. Pages do not set colours themselves. Geist stayed as the font, to avoid a new dependency.

**Kept the same on purpose**: The login and signup flow, the session handling, the API client, the routes and their paths, the filters and the search behaviour.

## Decisions made

**The panel no longer waits for the session check before showing the page.** The spinner used to hide the whole page. Now the page and its data request start together with the session check. Without a valid session the server returns an error and the user still ends up on the login page.

**Pages are not lazy loaded.** They were lazy loaded at first, but after a reload the whole content area (heading and filters too) was replaced by the loader until the page file arrived, which looked like a blank flash. Importing the pages directly shows the heading at once. The app is small, so the bundle cost is low.

**The loader recolour is a change to the supplied file.** The animation was near black. It is now royal blue with light blue page lines, to match the palette. Nothing was redrawn.

**The name moved from the top bar to the sidebar.** The sidebar shows "Prepify Admin" with the admin's name under it. The top bar now shows only Log out and the avatar.

**`MarkdownPreview` shows safe plain text with line breaks.** Real markdown and formulas come later, with images.

**Import is not all or nothing.** The check runs on every chunk first, and the import only starts if it is clean. If the connection drops in the middle, earlier chunks stay saved. Uploading the same file again is safe because rows with an existing `external_key` are skipped.

**`skeleton.tsx` stays in the repo.** Nothing uses it now. It was left so the shadcn set stays complete.

## Notes

- `npm run build` passes. It prints a warning from `lottie-web` about a direct `eval` call inside the library. The build still succeeds.
- Lint shows 3 warnings (`only-export-components`) in the shadcn files `button.tsx`, `badge.tsx` and `tabs.tsx`. They are from the shadcn template and were not changed here.
- While preparing this file the production build failed on a missing closing brace in `src/index.css`, left by an earlier edit of mine. The dev server did not notice. It is fixed and the build passes.
- Sizes in the new look (sidebar 240px, 40px controls) were estimated from the reference screenshot, so small spacing differences are likely.
- The reorder arrow error was caused by the backend CORS settings, not by this repo. The fix is in the backend PR and needs a backend restart.

## Next steps

- Run the migration and seed in the backend, then check every screen in the browser: subjects (arrows, rename, delete), the question form, the list, and a CSV and an Excel import with errors and then a clean one.
- Delete the test admin `grace@example.com` if it is no longer needed. It is still pending approval.

## Test plan

**Automated tests (35 passing):**
- [x] `src/features/content/lib/bulk-file.test.ts`: reads headers in any case, stops when a column is missing, refuses an empty file or too many rows, shares passage text by key, splits rows into chunks and builds the template file.
- [x] `src/features/content/pages/content-pages.test.tsx`: subjects with categories and counts, the archive warning for published questions, the archive request after confirm, the questions list and its first page request, and the empty state.
- [x] `npm run typecheck`: no errors.
- [x] `npm run lint`: 0 errors and 3 warnings in shadcn files.
- [x] `npx prettier --check`: all files match.
- [x] `npm run build`: passes, with the `lottie-web` eval warning.

**Manual checks (local):**
- [x] Looked at screenshots of the Questions and Subjects pages after the restyle, the dropdown fix, the content width fix and the loader. The loader appeared on Questions and the grey placeholder on Subjects, which was then replaced.
- [ ] Subjects arrows, rename and delete after the CORS fix - not checked, because the backend needs a restart and the test admin is pending approval.
- [ ] Question form, list and CSV and Excel import against the real backend - not checked, because the migration and seed have not been run.
- [ ] The loader on every page after the last change (pages imported directly and `QueryStatus`, which also removes the short blank flash after a reload) - not checked in a browser. Typecheck and the unit tests cover it.
