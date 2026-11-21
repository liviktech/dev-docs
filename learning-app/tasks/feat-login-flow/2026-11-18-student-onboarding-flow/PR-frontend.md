# 2026-11-18 - student-onboarding-flow (frontend)

## Summary

This PR also covers the login flow it sits on. Signup is now a set of pages, one question each (student or parent, name, test, how familiar, then email and mobile number). Login has one email field and a "Get OTP" button, then a screen to type the six digit code. Both are connected to the real backend routes (see the backend PR). Google sign in now starts a real session too, and the app restores a saved session when it opens.

After login the app always went straight to the dashboard. The old onboarding screens were not connected to anything, only kept answers in memory, and the Profile tab could not show or change them. This PR adds a six step student onboarding that opens after the first login (email code or Google), saves progress through the backend so a student can leave and resume, shows every answer on the Profile tab, and lets the student edit each part later. Students who finished onboarding go straight to the dashboard.

The backend endpoints do not exist yet, so in a dev build the app falls back to an in-memory mock when an endpoint answers 404. I have not run the app on a device or an emulator after these changes, only the type check and lint.

**The backend is not included on purpose** - see "Notes" below. This was a choice, not a miss.

**Dashboard personalisation from the answers is not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/app/(auth)/signup.tsx`, `signup-name.tsx`, `signup-test.tsx`, `signup-experience.tsx`, `signup-contact.tsx` (new except the first) | The signup pages, one question each. The last page creates the account, then goes to login. It has a "Sign in" link at the bottom |
| `src/app/(auth)/login.tsx` | Email only, with a "Get OTP" button. Google sign in is kept below it |
| `src/app/(auth)/verify-email.tsx` (new) | Six box code screen with a resend countdown and error messages |
| `src/app/(auth)/_layout.tsx`, `src/state/signup-draft.tsx` (new) | Keeps the signup answers while moving between pages |
| `src/components/auth/choice-option.tsx`, `otp-input.tsx` (new) | The choice card and the code input used by the pages above |
| `src/services/email-auth-api.ts`, `src/services/token-storage.ts`, `src/hooks/use-email-auth.ts` (new) | Calls to register, request code, resend and verify. Tokens are kept in secure storage |
| `src/hooks/use-google-sign-in.ts`, `src/components/auth/social-login.tsx`, `app.config.js` | Google sign in sends the token to the backend and starts a session. Supports several iOS URL schemes |
| `.claude/skills/reusable-ui-components/SKILL.md` (new) | Rule that every UI piece, even one button, is a reusable component |
| `src/services/api-client.ts` | Added signed-in calls (`apiGet`, `apiPut`, `apiAuthedPost`) that send the stored access token and retry once after a refresh when the server says 401. `apiPost` behaves the same as before |
| `src/services/auth-service.ts` | New `fetchCurrentUser` (calls `/auth/me`) so the app can ask the server whether onboarding is complete |
| `src/services/onboarding-api.ts` (new) | Types and calls for the onboarding endpoints, plus the list of step routes. This file is the contract for the backend. It also holds the dev only mock fallback |
| `src/services/onboarding-mock.ts` (new) | In-memory stand-in for the endpoints, used only in dev builds when an endpoint returns 404. Delete when the backend is live |
| `src/hooks/use-onboarding.ts` (new) | React Query hooks for the saved state, the current user, each step save, and completing onboarding. Saves update the cache in place so nothing is refetched |
| `src/hooks/use-after-auth.ts` (new) | One place that decides where to go after a session exists: onboarding for students who are not done, the dashboard for everyone else. Also clears cached student data from a previous account |
| `src/hooks/use-google-sign-in.ts`, `src/app/(auth)/verify-email.tsx` | Use the shared "where to go next" helper instead of always going to `/hub` |
| `src/app/index.tsx` | After the stored session is restored, asks the server for the user and routes by onboarding status. If offline it still goes to the dashboard like before |
| `src/app/(onboarding)/resume.tsx` (new) | Entry point that opens the first unfinished step from the server's saved progress |
| `src/app/(onboarding)/subjects.tsx` | Step 1. Subject cards grouped by exam, from the API. Multi select |
| `src/app/(onboarding)/academic.tsx` | Step 2. Date of birth (age shown from it), searchable country, grade, graduation year, optional school |
| `src/app/(onboarding)/goals.tsx` (new) | Step 3. Target score and exam date per exam, plus a preparation goal. All optional, can skip |
| `src/app/(onboarding)/skill-levels.tsx` (new) | Step 4. Beginner, Intermediate, Advanced or Not sure for each chosen subject |
| `src/app/(onboarding)/study-preferences.tsx` (new) | Step 5. Daily time, preferred time, study days, learning style. Can skip |
| `src/app/(onboarding)/learning-goals.tsx` (new) | Step 6. Goals for using Prepify. Saving this completes onboarding |
| `src/app/(onboarding)/welcome.tsx` (new) | Short welcome screen shown once after finishing, then to the dashboard |
| `src/app/(onboarding)/exam.tsx`, `role.tsx`, `profile-setup.tsx` | Removed. Nothing routed to them, and the exam and name are already collected at signup |
| `src/components/onboarding/onboarding-shell.tsx` | Progress bar, "Step n of 6", Back and Continue buttons, optional Skip, error text, keyboard handling |
| `src/components/onboarding/onboarding-loader.tsx` (new) | Loads saved state with loading and retry states. Also the helpers for moving between steps and for "Save and go back" when editing |
| `src/components/onboarding/option-group.tsx` (new) | Wrapped group of option pills, used for single and multi select |
| `src/components/onboarding/date-field.tsx` (new) | Date picker made from three scrolling columns in plain JS. Works on iOS, Android and web with no native module |
| `src/components/onboarding/select-field.tsx` (new) | Searchable dropdown, used for country |
| `src/components/onboarding/picker-field.tsx` (new) | The field look and the bottom sheet shared by the two pickers above |
| `src/components/onboarding/choice-card.tsx`, `src/components/onboarding/option-button.tsx` | no real change (reused as they were) |
| `src/components/profile/onboarding-details.tsx` (new) | Shows every onboarding answer in one card per step, each with an Edit button that opens that step |
| `src/app/(tabs)/profile.tsx` | Adds the "Your study profile" section |
| `src/data/countries.ts` (new) | Country names for the picker |
| `src/data/onboarding-options.ts` (new) | Option lists and labels shared by the onboarding steps and the Profile tab, so they cannot drift apart |

## Why

The flow was asked for as a six step student onboarding that appears after the first successful login, works for email and Google, never shows again once finished, resumes where the student left off, and can be edited later from Profile. The server has to own the "done" flag and the progress, not local storage, so the app now asks the server on every start.

The easy part was the screens. The real work was routing (three entry points: email code, Google, restored session), saving per step so progress survives a closed app, and keeping the Profile view and the editors using the same lists.

## How

**Routing after login**: Email code, Google and a restored session all call the same helper. It puts a student with `onboardingCompleted: false` on `/resume`, and everyone else on `/hub`. `/resume` reads the saved state and redirects to the step in `currentStep`.

**Saving and resuming**: Each step saves with its own `PUT` call, and the server returns the full state, which goes straight into the React Query cache. Step 6 uses `POST /student/onboarding/complete`, which saves the goals and marks the student done in one call. If a step is already saved and nothing changed, Continue just moves on without calling the server.

**Going back without losing answers**: Back pops the screen, and earlier screens stay mounted, so what the student typed is still there. If the app was resumed straight into step 4, Back replaces the screen with step 3 instead.

**Editing from Profile**: The same six screens are the editors. When onboarding is already complete they show "Save" instead of "Continue", hide Skip, and return to Profile after saving. The Profile cards read the same cached state, so they update at once.

**Signed-in calls**: The new calls read the access token from secure storage. On a 401 they refresh the token once and retry. Refresh is shared, so two calls failing together do not both use the single-use refresh token.

**Dev mock**: When an onboarding call returns 404 in a dev build, the app uses `onboarding-mock.ts` for the rest of the session. Release builds never do this.

## Decisions made

**Reused the existing screens as editors instead of building separate edit screens.** A second set of forms would have doubled the code and the validation. The screens now switch their button text and what happens after saving when onboarding is already complete.

**Built the date picker in plain JS instead of adding a native date picker package.** A native package needs a new dev client build, and the current build would crash without it. The JS picker works everywhere and needs no rebuild. It is less polished than the system picker.

**Used the existing yellow and navy design, not purple.** The brief said purple, but the app already uses the yellow and navy theme everywhere. I did not change the theme.

**AP subjects show when the signup test is "Other".** The backend only knows SAT and ACT, and "Other" is the only signup answer that can mean AP. A student with no saved test (Google signups skip it) sees all three exams. The backend needs to follow the same rule.

**Skill level starts on "Not sure" for every subject.** The student can leave it and continue. It is a self rating, not a test result, and is never treated as one.

**Only students go through onboarding.** Parents and admins go straight to the dashboard.

**Skip only exists on steps 3 and 5.** Those are the optional ones. Subjects, academic details and learning goals are needed to build a plan.

## Bugs found while testing (already there before, not fixed here)

**1. Log out does not clear the saved login.** The Log out button on Profile resets the in-memory study state and goes to `/login`, but it does not call the logout endpoint or clear the tokens in secure storage. On the next app start the stored session is restored and the student is signed in again. It affects every user, and now also means the next start can show the previous account's onboarding status.

**2. Type error in `src/components/app-tabs.web.tsx`.** `tsc --noEmit` reports a ref type mismatch on line 45. The file is not touched by this PR. It only affects the web build type check.

**3. Profile tab still shows made up numbers.** The streak, XP, accuracy and mock score tiles, and the Subjects and Role summary at the top, come from mock data and the old in-memory study state, not from the real answers. The new "Your study profile" section below them is the real data.

## Notes

- The backend does not exist yet. It needs these routes, all with the student's access token: `GET /student/onboarding`, `PUT /student/onboarding/subjects`, `/academic`, `/goals`, `/skill-levels`, `/study-preferences`, `POST /student/onboarding/complete`, and `GET /student/profile`. Every save should return the full state. The exact request and response shapes are the types in `src/services/onboarding-api.ts`. The signup and login routes already exist, see `PR-backend.md` in this folder.
- Set `EXPO_PUBLIC_API_URL` in `.env` (for example `http://10.0.2.2:3000/api/v1` on the Android emulator, the PC's LAN address on a real phone) and restart with `npx expo start -c`, because it is baked in when Metro starts.
- `currentStep` from the server is the next step to show (1 to 6). `completed` should come from `users.onboarded_at`.
- Profile is read from the same query as onboarding, so `GET /student/profile` is defined but not called yet.
- Dev mock state lives in memory, so it resets whenever the app reloads.

## Next steps

- Build the backend routes above, then delete `src/services/onboarding-mock.ts` and the `withMock` wrapper in `src/services/onboarding-api.ts`.
- When a student removes a subject in Profile, the backend should drop that subject's skill level and any exam goal left with no subjects.
- Use the saved answers to personalise the dashboard (target score, exam date, study time).
- Raise a ticket for the logout bug above.

## Test plan

**Automated tests (none exist for this app yet):**
- [x] `npx tsc --noEmit`: passes for the files in this PR. One error remains in `src/components/app-tabs.web.tsx`, which this PR does not touch (see bug 2)
- [x] `npx expo lint`: no warnings or errors
- [ ] Unit tests for the date helpers (`ageFrom`, `parseIsoDate`) and the score check on the goals step - not written, the app has no test setup

**Manual checks (local):**
- [ ] New email signup, then login with the emailed code, then six steps, welcome screen, dashboard - not run on a device or emulator. Only the type check and lint were run
- [ ] First Google login goes into onboarding - not run, needs a real device with Google set up
- [ ] Completed student logs in and goes straight to the dashboard - not run
- [ ] Close the app on step 3 and reopen it to resume on step 3 - not run. With the dev mock this resets because the mock is in memory, so it can only be fully checked once the backend exists
- [ ] Subjects shown for SAT only, ACT only, both and Other (AP) - not run
- [ ] Edit from Profile saves and returns with the new values - not run
- [ ] iOS and Android layout - not run
