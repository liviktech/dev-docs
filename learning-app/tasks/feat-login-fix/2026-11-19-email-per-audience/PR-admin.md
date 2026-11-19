# 2026-11-19 - email-per-audience (admin)

## Summary

After a reload, the admin panel showed a short blank screen. Every page was loaded on demand, so until the page file arrived the whole content area (heading and filters too) was replaced by the book loader. This PR imports the signed in pages directly. The heading and filters now show on the first paint and only the table area shows the loader. It also updates the README for the backend change that lets one email have an admin account and a mobile account.

The reload flash is fixed in code. I did not check it in a browser.

## What changed

| File | Change |
|---|---|
| `src/routes/routes.tsx` | Removes the `page(...)` lazy helper and imports `DashboardPage`, `QuestionsPage`, `NewQuestionPage`, `EditQuestionPage`, `ImportPage` and `SubjectsPage` directly. Routes and paths are the same. |
| `README.md` | Setup step 3 now says migration `0007` lets one email have an admin and a mobile account. |

## Why

The lazy loading was added to give every page a loader. In practice it made the page area empty for a moment after a reload, because the `Suspense` fallback in `DashboardLayout` replaces everything inside the content area. The data loader (`QueryStatus`) already shows the loader where the data will appear, so lazy loading was not needed for that. The app is small, so loading everything at once costs little.

## How

**Direct imports**: The pages are imported at the top of `routes.tsx`. The heading and filters of a page render at once. While the data loads, `QueryStatus` shows the book loader in the table area only.

**Safety net stays**: `DashboardLayout` still wraps the page in `Suspense` with the book loader. If a lazy page is added later, it still gets a loader.

**Kept the same on purpose**: The routes, their paths, `ProtectedRoute`, the layout and all page files. Only the way the pages are loaded changed.

## Decisions made

**Pages are not lazy loaded.** I considered keeping lazy loading and hiding the loader for the first 200 ms, or preloading the page files. Both reduce the flash but do not remove it. Direct imports remove it and are the simplest.

## Notes

- The user name under "Prepify Admin" in the sidebar still fills in only after the session check finishes. This is a separate, smaller thing and is not changed here.
- `.claude/` and `skills-lock.json` show as untracked in `git status`. They are local tool files and are not part of this PR.

## Next steps

- Reload `/questions` and `/subjects` in a browser to confirm the blank flash is gone.
- Show a placeholder for the user name in the sidebar while the session loads, if the late name looks odd.

## Test plan

**Automated tests (35 passing):**
- [x] `src/features/content/lib/bulk-file.test.ts` and `src/features/content/pages/content-pages.test.tsx`: still pass with the direct imports. There is no test for `routes.tsx` itself.
- [x] `npx tsc -b`: no errors.
- [x] `npm run lint`: 0 errors and 3 warnings (`only-export-components`) in the shadcn files `tabs.tsx`, `badge.tsx` and `button.tsx`. They were there before.
- [x] `npx prettier --check src/routes/routes.tsx`: matches.

**Manual checks (local):**
- [ ] Reload `/questions`, `/subjects` and `/` and check that the heading and filters stay on screen while the table loads - not checked in a browser. Typecheck and the unit tests cover that the routes build.
