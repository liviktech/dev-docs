# 2026-11-21 - parent-login-dashboard (backend)

## Summary

The backend already let someone sign up as a parent, but a parent could not do anything after that. This PR adds a way for a parent to link a child by the child's email, for the child to approve or reject the request, and for an approved parent to read a read-only dashboard of the child's learning (profile, activity, results, progress). Parents also no longer need to send the test questions when they sign up. Login can now say which kind of account it expects (student or parent).

Everything was checked against a real PostgreSQL test database through the HTTP routes. I did not run the new migration on the shared hosted database.

**Email or push notification to the student is not included on purpose** - see "Next steps" below. This was a choice, not a miss.

**Lessons and a profile picture are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0015_parent-child-links.sql` (new) | New table `parent_child_links` (parent, child, status pending/approved/rejected). One row per pair, so a duplicate request is stopped by the database |
| `src/modules/parent/` (new module) | Controllers, services, two repositories, a role guard, Zod schemas, DTOs and constants for linking and the dashboard. See "How" |
| `src/modules/parent/parent.controller.ts` (new) | Parent routes: look up a child, send a request, list children, remove or cancel, read the dashboard |
| `src/modules/parent/parent-requests.controller.ts` (new) | Student routes: list parent requests, approve, reject |
| `src/modules/parent/parent-dashboard.service.ts`, `parent-dashboard.logic.ts`, `repositories/parent-dashboard.repository.ts` (new) | Builds the dashboard from existing data. The logic file holds the pure maths (accuracy, weekly compare, strong and weak topics, summary text) |
| `src/modules/parent/parent-links.service.ts`, `repositories/parent-links.repository.ts` (new) | The request, approve, reject and remove rules, with the limits and the 24 hour wait after a rejection |
| `src/modules/parent/guards/parent-role.guard.ts` (new) | Same idea as the student guard, but needs the `parent` role |
| `src/modules/auth/schemas/auth-request.schemas.ts` | `register`: `targetTest` and `testExperience` are optional, but still required for students. `request-otp` and `resend-otp`: new optional `role` |
| `src/modules/auth/auth.service.ts` | Skips saving exam preferences when there is no `targetTest`. When `role` is sent with a login code request and the account has the other role, answers "account not found" |
| `src/modules/stats/stats.module.ts`, `src/modules/mock-exams/mock-exams.module.ts` | Now export `StatsService` and `MockExamsService` so the dashboard can reuse them. No other change |
| `src/app.module.ts` | Registers `ParentModule` |
| `src/shared/constants/http.constants.ts` | New error codes (`CHILD_NOT_FOUND`, `PARENT_LINK_EXISTS`, `PARENT_LINK_NOT_APPROVED`, `PARENT_LINK_LIMIT`, `PARENT_LINK_COOLDOWN`, `PARENT_LINK_RESPONDED`) |
| `src/common/decorators/auth-throttle.decorator.ts`, `src/common/guards/throttler.options.ts`, `src/config/env.schema.ts`, `src/config/security.config.ts`, `.env.example` | New `link` rate limit (`THROTTLE_LINK_LIMIT`, default 20 per minute per IP) for the look up and request routes |
| `test/integration/parent.int-spec.ts` (new) | 24 integration tests for login, linking, approval, access rules and the dashboard |
| `test/unit/parent-dashboard.spec.ts` (new) | 8 unit tests for the dashboard maths |
| `test/integration/setup-env.ts` | Sets a high `THROTTLE_LINK_LIMIT` for tests |
| `architecture.md` | New section 10i, the `link` throttler row, and the updated signup and `request-otp` docs |

## Why

A parent account was useless on its own. Parents want to see how their child is doing, but a child's data is private, so the child has to agree first. The easy part was the dashboard, because most numbers already exist (stats, mock exam history, onboarding answers). The real work was making sure a parent can never see a child who has not approved, and can never reach another family's child by changing an id or an email.

## How

**Linking**: A parent types the child's email. A look up route confirms the student exists and shows only the name and a masked email. A second route creates a `pending` link. The child sees the request in the app and approves or rejects it. Only an `approved` link opens the dashboard.

**Access rule**: The dashboard route takes the link id, never a student id or email. The service finds the student through the parent's own approved link, so nothing sent by the client can point at another student. A link that is not the caller's, or does not exist, answers 404 (the same answer, so it does not leak). A pending or rejected link answers 403 `PARENT_LINK_NOT_APPROVED`. There are no write routes under the dashboard.

**Limits**: A parent can follow up to 10 children. A child can have up to 4 approved parents, checked under a row lock so two approvals at once cannot go over. After a rejection the same parent must wait 24 hours to ask again. A child removes a parent by rejecting an approved link, which ends access straight away.

**Dashboard data**: Profile comes from the user and onboarding rows (age is worked out from the date of birth and the date of birth itself is not sent). Overall accuracy and subjects reuse `StatsService`. Mock exam results and the change from the last attempt reuse `MockExamsService`. Activity, the 14 day chart, strong and weak topics and the streak are new read-only queries. The streak reuses the existing `computeStreak`.

**What is never sent**: passwords, tokens, tutor chats, phone numbers, the student's full email or id. Tests scan the responses for these.

**Login role**: The app sends `role` with the login code request. If the email belongs to the other role it answers "account not found". If `role` is left out, everything works as before.

## Decisions made

**Study time is an estimate.** Nothing in the app records time spent (the `time_spent_seconds` column is never written). So each session counts the time between its first and last answer, at least 20 seconds per answered question, and never more than 3 hours. I did not add a new tracking field because that is a bigger change. The app labels the number as estimated.

**Lessons and a profile picture are returned as nothing.** Lessons do not exist in the product yet, so `completedLessons` is always `null`. There is no profile picture field anywhere, so none is returned.

**A student removes a parent by rejecting.** I did not delete the row. A deleted row would let the parent ask again straight away. A rejected row keeps the 24 hour wait. For the same reason a parent cannot delete a rejected link.

**Looking up a child tells you if an email is registered.** The spec asks to check that the child exists and show their details, so this is on purpose. It is limited to logged in parents, returns only a name and a masked email, and has its own rate limit.

**Strong and weak topics are still returned by the API.** The mobile app no longer shows them (asked for after this was built). I left the data in the API.

**Reused instead of copied.** The dashboard calls the stats and mock exam services instead of repeating their SQL, which is why those two modules now export a service.

## Bugs found while testing (already there before, not fixed here)

**1. Three unit test files fail on the original code too.** `test/unit/mock-exams.spec.ts` and `test/unit/assessment-shuffle-clock.spec.ts` fail with "Cannot read properties of undefined (reading 'config')", and `test/unit/otp.service.spec.ts` fails one case (the email text does not contain `4 8 2 9 1 3`). I checked by stashing my changes and running them again, and they failed the same way. This affects anyone running `npm test`.

**2. `test/integration/student-onboarding.int-spec.ts` fails the Prettier check.** It was not touched by this PR.

**3. Time spent on a question is never saved.** The column exists but nothing writes it, which is why study time had to be estimated.

## Notes

- The `.env` `DATABASE_URL` points at a shared hosted database. I never ran any migration there. Run `npm run migrate:up` on it on purpose when this is merged.
- The migration was first numbered 0014, then renumbered 0015 because `0014_study-plans` landed first. A local database that already ran the old name needs that row removed from `pgmigrations` first.
- Local PostgreSQL 17 on my machine could not run migrations (Windows blocked `plpgsql.dll`), so I ran all tests on a temporary PostgreSQL 18 instance.
- `npm run typecheck`, `npm run lint` and Prettier on the new files were clean.

## Next steps

- Send the student an email or push message when a parent asks to follow them. Today the request only shows inside the app.
- Decide whether the API should stop returning strong and weak topics now the app does not show them.
- File tickets for the three old bugs above.

## Test plan

**Automated tests (235 integration tests and the new unit tests passing, 3 old unit files failing as listed above):**
- [x] `test/integration/parent.int-spec.ts`: parent signup without test answers and student still needs them, login role match and mismatch, role separation, child look up (masked email only, 404 for unknown, parent and disabled accounts), request then approve, no data before approval, duplicate request, reject then 24 hour wait, removing an approved parent ends access, parent cancel and unlink, several children and several parents, the 4 parent limit
- [x] `test/integration/parent.int-spec.ts`: another parent's link id, an invented id, a student id used as a link id, another student answering a request, no write routes, disabled student, empty dashboard for a new student, real practice results (accuracy, strong and weak topics, weekly numbers, streak), and no email, student id, token or phone in any response
- [x] `test/unit/parent-dashboard.spec.ts`: accuracy rounding, 14 day window, this week against last week, topics with too few answers, strong, weak and repeated mistake rules, daily goal days, summary text
- [x] `npm run test:integration` (235 passed on my last run, before the analytics changes were merged in)
- [x] `npm run typecheck`, `npm run lint`: clean
- [x] Migration `0015` up, down, up on the test database: clean

**Manual checks (local):**
- [x] Ran the whole flow through the real HTTP routes in the integration tests: register as parent, log in with a code, look up a child, request, approve, read the dashboard
- [ ] Ran against the shared hosted database - not done on purpose, the migration has to be run there by you
- [ ] Email to the student - not built, see "Next steps"
