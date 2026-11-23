# 2026-11-21 - parent-login-dashboard (frontend)

## Summary

Parents could pick "parent" at signup, but the app then treated them like a student with no screens of their own. This PR gives parents their own area: a Parent option on the login screen, a signup that skips the test questions, a bottom tab bar (Dashboard, Results, Activity, Profile), a screen to add a child by email, and a Profile tab where they see their details, each child with its status, and can remove a child or log out. Students get a Parent access screen to approve or decline a request, with a banner on the Hub while one is waiting.

I did not run the app on a device or emulator after these changes. I ran the type check, lint and a full Android bundle (`expo export`), and they passed. The screens are checked against the backend only through the backend integration tests.

**Showing strong and weak topics is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

**Lessons and a profile picture are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/app/parent/_layout.tsx` (new) | The parent area with native bottom tabs (`NativeTabs`), the same component as the student tabs. Wraps the tabs so the chosen child is shared |
| `src/app/parent/dashboard.tsx`, `results.tsx`, `activity.tsx` (new) | The three tabs that show the chosen child. They only pick which sections to draw |
| `src/app/parent/profile.tsx` (new) | The parent's name, email and phone, the list of children with status, Remove child or Cancel request, Add a child, Log out |
| `src/app/add-child.tsx` (new) | Type the child's email, confirm who it is, send the request, then a waiting message |
| `src/app/parent-requests.tsx` (new) | Student screen to approve, decline or remove a parent |
| `src/app/_layout.tsx` | Registers the `parent`, `add-child` and `parent-requests` routes |
| `src/app/(auth)/login.tsx` | Student and Parent switch, the choice is sent with the login code request |
| `src/app/(auth)/signup-name.tsx`, `signup-contact.tsx` | A parent goes from name straight to email and phone, skipping the test and familiarity pages. After signup, login opens on the Parent option |
| `src/app/(auth)/verify-email.tsx` | Passes the chosen role to the resend code call |
| `src/app/(tabs)/hub.tsx`, `src/app/(tabs)/profile.tsx` | One line each to show the Parent access banner and row |
| `src/components/auth/role-switch.tsx` (new) | The Student and Parent switch |
| `src/components/parent/*` (new) | Screen shell, cards, child switcher, status badge, the profile, activity, results and progress sections, and the shared tab wrapper (`child-tab.tsx`) that handles loading, errors, and the "no approved child yet" state |
| `src/components/parent/parent-access-row.tsx` (new) | The student's Hub banner and Profile row |
| `src/hooks/use-parent.ts`, `use-selected-child.tsx`, `use-logout.ts` (new) | Queries and mutations, the chosen child shared across tabs, and logout |
| `src/hooks/use-after-auth.ts` | A parent goes to `/parent/dashboard` after login. Also clears cached parent data on sign in |
| `src/services/parent-api.ts` (new) | API calls and types for the parent and student request routes |
| `src/services/email-auth-api.ts` | `request-otp` and `resend-otp` take an optional role. Signup sends the test answers only when there are some |
| `src/utils/parent-format.ts` (new) | Small formatting helpers (minutes, percent, change, weekday, dates) |
| `eas.json` | no real change (formatting, and iOS `simulator` set to false for the development profile). Not part of the parent work, it came in the same commit |

## Why

The backend now has parent linking and a dashboard (see the backend PR). The app needed screens for it. The parent flow also had to stay out of the student flow, so students see no change except a small banner when a parent asks to follow them.

## How

**Own area, own tabs**: Parent screens live under `src/app/parent/`. A real folder name (not a group) is used because a group would make the parent Profile and the student Profile both resolve to `/profile`.

**One chosen child**: A small provider in the parent layout loads the parent's children once and remembers which approved child is selected. Dashboard, Results and Activity all read it, so switching a child once changes all three. The switcher only shows when more than one child has approved.

**One wrapper for the tabs**: `child-tab.tsx` loads the chosen child's dashboard and handles loading, errors, pull to refresh, refresh on focus, and the empty states (no child yet, or waiting for approval). Each tab is then only a few lines.

**Approval waiting state**: A pending child does not open a dashboard. The tabs show a waiting message that tells the parent where the child approves it. A 403 or 404 from the dashboard shows "No access" and is not retried.

**Student side**: A banner on the Hub shows only while a request is waiting. The Profile tab always has a Parent access row. Approving, declining and removing all use the same screen.

## Decisions made

**Strong and weak topics are not shown.** They were built first, then removed from the app because the parent flow was asked to have only Dashboard, Results, Activity and Profile. The backend still returns the data.

**Lessons and a profile picture are not shown as data.** Lessons do not exist yet, so the Activity tab only says "Lessons: coming soon" instead of a count. There is no picture field, so the initial of the child's name stands in for a picture.

**The old "list of children as cards" home was removed.** Parents now land on the Dashboard. Removing a child, status and logout moved to the Profile tab.

**Study time says "estimated".** The backend works it out from answer times, not a real timer.

## Notes

- `src/components/auth/auth-shell.tsx` has an uncommitted change on my machine (brand colour `#80e073`). It is not part of this PR and I did not touch it.
- The app crashed on open on one build with "Can't find ViewManager LottieAnimationView". That comes from the study plan work adding `lottie-react-native`, not from the parent work. The installed app has to be rebuilt after that package was added, and the Android build also kept an old autolinking cache (`android/build/generated/autolinking/autolinking.json`) that had to be deleted before the rebuild picked Lottie up.
- Do not run `./gradlew clean` here. It failed with a CMake error about missing worklets headers. Delete the `.cxx` folders and build with `npx expo run:android` instead.

## Next steps

- Run the parent flow on a real device and an emulator end to end (signup as parent, login, add child, approve from a student account, dashboard).
- Show a message to the student when a parent asks (push or email), once the backend supports it.
- Decide if the project should keep only one of `yarn.lock` and `package-lock.json`, because the Android build cache checks both.

## Test plan

**Automated tests (no test suite in this app, only type check, lint and bundle):**
- [x] `npx tsc --noEmit`: no errors
- [x] `npx expo lint`: no warnings
- [x] `npx expo export --platform android`: bundles without errors

**Manual checks (local):**
- [ ] Parent signup, login, add child, approve, and the four tabs on a device or emulator - not done, I could not run the app. The backend side of the same flow is covered by `test/integration/parent.int-spec.ts` in the backend PR
- [ ] Student approving a request from the Hub banner and Profile row - not done on a device, same reason
- [ ] Child switcher with two approved children - not done on a device, same reason
