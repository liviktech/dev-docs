# 2026-10-13 - enquiry-pwa (frontend)

## Summary

The app could not be installed on a phone and had no mobile-friendly screens. This PR makes the whole app installable as a PWA (manifest, icons, service worker, install and update handling). It adds a separate mobile UI for the Enquiry module only, under `/m/enquiries`. Every other module keeps its existing desktop UI and is reached from a new top menu bar on small screens. I checked that the production build and TypeScript pass. I did not test on a real phone or in browser device emulation.

**The company-level PWA on/off setting is not included on purpose** - see "Decisions made" below. The backend has no such setting yet, so this was a choice, not a miss.

**No backend files changed in this task, so there is no PR-backend.md for it.**

## What changed

| File | Change |
|---|---|
| `public/manifest.webmanifest` (new) | App name, icons, standalone display, start URL `/m/enquiries` |
| `public/icons/icon-192.png`, `icon-512.png`, `apple-touch-icon.png` (new) | App icons (the Gem logo on the app's cyan colour) |
| `src/pwa/sw.js` (new) | Service worker. Caches only the HTML shell, hashed assets and icons. Never touches `/api/*` |
| `vite.config.ts` | Small plugin that emits `sw.js` with a build id so each deploy gets a fresh cache |
| `vercel.json` | No-cache headers for `sw.js` and the manifest |
| `index.html` | Added `viewport-fit=cover` for phone safe areas |
| `src/features/pwa/pwa-access.ts` (new) | PWA allowed check, installed-app check, safe redirect check |
| `src/features/pwa/pwa-store.ts` (new) | Install prompt, service worker registration, update and cleanup |
| `src/features/pwa/pwa-runtime.tsx` (new) | Adds manifest tags, registers the worker, shows offline and "new version" banners |
| `src/features/pwa/use-online-status.ts` (new) | Online or offline hook |
| `src/features/pwa/mobile-shell.tsx` (new) | Layout and top bar for mobile screens. Handles signed out, offline and PWA-disabled cases. Menu with other modules, install and sign out |
| `src/features/pwa/mobile-ui.tsx`, `mobile-styles.ts` (new) | Shared mobile pieces: bottom sheet, searchable picker, confirm sheet, skeleton, status badge |
| `src/features/pwa/mobile-enquiry-list.tsx` (new) | Compact enquiry list with search, status chips, party and date filters, infinite loading |
| `src/features/pwa/mobile-enquiry-form.tsx` (new) | Create and edit form with inline validation and photos |
| `src/features/pwa/mobile-enquiry-detail.tsx` (new) | Details with Approve, Close and Reopen |
| `src/features/pwa/mobile-no-access.tsx` (new) | Blocks the mobile screens without Enquiry VIEW access |
| `src/routes/m.tsx`, `m.index.tsx`, `m.enquiries.tsx`, `m.enquiries_.new.tsx`, `m.enquiries_.$enquiryId.tsx`, `m.enquiries_.$enquiryId_.edit.tsx` (new) | Routes for the mobile Enquiry screens |
| `src/routes/__root.tsx` | `/m/*` skips the desktop layout. Mounts the PWA runtime and offline banner (banner only in the installed app) |
| `src/routes/enquiries.tsx` | In the installed app on a phone, redirects to `/m/enquiries`. Browsers are unchanged |
| `src/routes/login.tsx`, `src/features/auth/login-form.tsx` | Support `?redirect=` after login (same-origin paths only) |
| `src/features/auth/session-expired-dialog.tsx` | Returns to the mobile screen after logging in again |
| `src/features/auth/api.ts` | Added optional `pwaEnabled` on the user |
| `src/features/auth/accessible-routes.ts` | Added menu labels and a helper that lists the modules a user can open |
| `src/components/layout/mobile-menubar.tsx` (new) | Top bar with company name and hamburger menu for screens under 768px. Lists modules, Master Data, profile and Sign out |
| `src/components/layout/nav-items.ts` (new) | Menu items moved out of the sidebar so the sidebar and the mobile menu share one list |
| `src/components/layout/app-sidebar.tsx` | Menu items moved to `nav-items.ts`. No change to how the sidebar looks or works |
| `src/components/layout/app-layout.tsx` | Hides the sidebar and shows the top menu bar under 768px |
| `PWA.md` (new) | Short summary of the PWA work and its limits |
| `src/features/enquiries/enquiries-page.tsx` | **Not part of this task's work.** Contains a mobile card list added to the desktop page, see "Notes" |
| `src/features/inventory/components/receive-stock-dialog.tsx` | **Not part of this task's work.** Removes the unused `handleApplyCalculatedSqm` function, no real change in behaviour, see "Notes" |

## Why

Users need to create and follow up enquiries from a phone. The ask was an installable app with a clean mobile UI for Enquiry only, while all other modules stay reachable and unchanged. The real work was the mobile Enquiry screens and keeping the service worker from ever caching private data.

## How

**Install and updates**: The manifest and service worker are added when a user is signed in (not on the Super Admin console). The service worker is only registered in production builds. A new deploy is picked up when the app comes back to the foreground, and the user taps Refresh to switch, so a half-filled form is never reloaded.

**Caching**: Only the app shell, hashed JS and CSS files, fonts and icons are cached. API calls are never cached, so enquiry and customer data cannot leak between companies or users. Old caches are deleted when a new version activates.

**Mobile Enquiry**: The screens reuse the existing enquiry, party, sales rep and photo APIs and the existing permission checks. Create and edit need CONTROL, edit is only allowed while the enquiry is OPEN, and Approve is for managers and above, the same as desktop. Company data separation is unchanged because the backend takes the company from the login session.

**Menu on phones**: Under 768px the sidebar is replaced by a top bar with the company name and a hamburger menu. Sign out is inside the menu. From 768px up the sidebar is the same as before.

**Kept the same on purpose**: Work Orders and all other modules have no mobile-specific changes. Earlier Work Orders mobile changes were reverted to the committed version.

## Decisions made

**Company PWA setting not enforced.** The backend has no PWA setting (no database column, `/auth/me` does not return it, no toggle in the Super Admin company dialog). I added an optional `pwaEnabled` field on the user and a check for it. If it is `false` the mobile screens show a "not enabled" page and the manifest and service worker are removed. If it is missing it counts as enabled. I did not add the backend part because this task is frontend only.

**Native date input and a searchable picker.** Dates use the phone's own date input, and long lists (party, project, sales rep, source) use a bottom sheet with search. I did not reuse the desktop dropdown and date picker because they are built for a mouse.

**Not included in the mobile form on purpose:** adding a new party and adding a custom enquiry source. These stay on desktop. A source created on desktop still shows up when editing.

**No offline mode.** Nothing is queued or saved offline. When offline the app shows a message and a retry button.

## Bugs found while testing (already there before, not fixed here)

**1. Offline cold start looks like a signed-out user.** The existing session check treats any network failure as "no user". In the mobile shell I show an offline screen with a Retry button instead of sending the user to login, but the same check in the desktop app is unchanged.

## Notes

- The commit that holds this work also contains a mobile card list inside the desktop `src/features/enquiries/enquiries-page.tsx` (about 150 changed lines). I did not write it for this task, and it goes against the rule that the regular Enquiry page stays unchanged. Please review it, and remove it if the desktop page should not change.
- The same commit removes `handleApplyCalculatedSqm` in `receive-stock-dialog.tsx`. It is unrelated to the PWA.
- `eslint` still reports `react-refresh/only-export-components` on route files and `auth-context.tsx`. The same pattern already exists in other route files, so I left it.
- No tests were run in a real browser or on a device. Checked with `vite build` and `tsc -b` only.

## Next steps

- Add a `pwa_enabled` column on the company, return it from `/auth/me`, add a toggle in the Super Admin company dialog, and enforce it in the backend.
- Test at 320px, 375px and 430px widths and on a real Android phone, including install and standalone launch.
- Decide whether to keep or remove the mobile card list in `enquiries-page.tsx`.

## Test plan

**Automated tests (none exist for this area):**
- [x] `vite build`: passes
- [x] `tsc -b`: passes, no errors
- [x] `eslint` on the changed files: no new errors beyond the existing `react-refresh/only-export-components` pattern
- [ ] There is no `test` script in `package.json`, so no unit tests were run

**Manual checks (local):**
- [ ] Mobile Enquiry list, search, filters, create, edit, details at 320px, 375px and 430px - not checked, no browser or device run yet
- [ ] Install and standalone launch on Android - not checked, needs a deployed HTTPS build
- [ ] Offline and retry messages - not checked live
- [ ] Desktop Enquiry page unchanged - not confirmed, see the note about `enquiries-page.tsx`
- [x] Work Orders files match the committed version (checked with `git status`, no changes)
