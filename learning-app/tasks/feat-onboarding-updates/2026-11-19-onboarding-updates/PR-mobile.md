# 2026-11-19 - onboarding-updates (mobile)

## Summary

The onboarding screens only worked against a dev-only mock, with subjects hardcoded and "Video lessons" and "AP Exams" still offered. This PR connects the app to the new backend onboarding API, loads subjects from the database, removes "Video lessons" from the study style options, and removes AP Exams from the goals step. The "Other" test choice at signup is hidden behind a flag.

The type check shows no errors in the files changed here. The app was not run on a device or emulator, so nothing is checked live.

**The Skill Level screen layout is not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/services/onboarding-api.ts` | Calls go straight to the real `/student/...` routes. The mock wrapper and `ApiError` are removed. `OnboardingExam` is now `'sat' \| 'act'`. `LearningStyle` has no `video_lessons`. Subjects are `{ id, name }` from the API. New `SubjectChoice` type and `selectedSubjects(state)` helper. File line endings normalized to CRLF. |
| `src/services/onboarding-mock.ts` (deleted) | The dev-only mock is gone. |
| `src/data/onboarding-options.ts` | Removed the "Video lessons" entry from `STYLES`. |
| `src/app/(auth)/signup-test.tsx` | New `SHOW_OTHER_TEST = false` flag. The "Other" choice is filtered out of the list when the flag is off. |
| `src/app/(onboarding)/subjects.tsx` | Subjects come from the API and are chosen by id. A small helper picks an icon by subject name. |
| `src/app/(onboarding)/goals.tsx` | AP Exams removed from the goals step. Uses subject ids. |
| `src/app/(onboarding)/skill-levels.tsx` | Uses `selectedSubjects`, subject id and subject label. No layout change. |
| `src/components/profile/onboarding-details.tsx` | Profile summary uses `selectedSubjects` and id based rows. |

## Why

The user asked for onboarding to work end to end, with subjects coming from the database only, "Video lessons" removed from "How do you like to study?", and AP Exams removed completely, including the "What are you aiming for?" goals step. The backend work is in `PR-backend.md`. Most of the work here was moving from subject names to subject ids, because the same name (Mathematics) exists in both SAT and ACT.

## How

**Subjects by id**: the app saves and reads subject ids. When more than one exam is offered, the label shows the exam name first, for example "SAT Mathematics", so two Mathematics subjects are not confused.

**Real API only**: with the mock removed, every step calls the backend and receives the full state back, so no refetch is needed. `currentStep` in the response tells the app which step to show, which is how resume works.

**Hidden "Other"**: `ALL_TESTS` keeps every choice and `TESTS` is the filtered list. Setting `SHOW_OTHER_TEST = true` brings "Other" back with no other change.

**Kept the same on purpose**: the six step routes (`STEP_ROUTES`) and the step screens' flow are unchanged.

## Decisions made

**Skill Level screen unchanged.** The user said no change. Only the data source (subject ids) moved.

**ACT Writing stays in the list.** The user confirmed it. It comes from the database like the other subjects.

**"Other" is hidden, not removed.** The backend still accepts it, so it can be turned on later with the flag when more exams are added.

## Notes

- The Subjects screen is empty until the database has SAT and ACT subjects seeded.
- Backend migrations 0008 and 0009 must be applied first, or every onboarding call fails.
- `tsc` still reports one error in `src/components/app-tabs.web.tsx`. I did not touch that file.

## Next steps

- Run the manual checklist in section 9 of `PLAN.md` on a real device.

## Test plan

**Automated tests (none added):**
- [ ] No mobile test file covers the onboarding screens - not added in this PR.
- [x] Type check (`tsc`): no errors in the files changed here. One error remains in `src/components/app-tabs.web.tsx`, which this PR does not touch.

**Manual checks (local):**
- [ ] New signup, steps 1 to 6, then log out and back in to confirm it resumes - not run, no device or emulator was used.
- [ ] Confirm "Video lessons", AP Exams and "Other" are gone from the screens - not run, only checked in code.
