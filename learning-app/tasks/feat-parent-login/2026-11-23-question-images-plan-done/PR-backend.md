# 2026-11-23 - question-images-plan-done (backend)

## Summary

Admins can already upload images for questions (stored in S3 and linked in the database), but no student endpoint sent them, so students never saw them. Also, a student who finished today's planned practice with every answer right still had to tick it off by hand in the study plan.

This PR does two things. Practice sessions and mock exams now return the images for the question, the passage, each option and the explanation. And when a practice session is finished with every answer right, the matching planned session for that student's day is marked done automatically.

Both were checked through the real HTTP routes on a real PostgreSQL test database. I did not check the images against the real S3 bucket, only with a fake CDN address in the tests.

**Auto-complete for mock exams is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/media/question-images.service.ts` (new) | `QuestionImagesService`. Loads the images of a set of questions in one query and turns the stored keys into public URLs. `QuestionImageSet` reads them by owner (stem, explanation, option, passage) |
| `src/modules/media/repositories/media.repository.ts` | New `findQuestionImages`: one statement over the three link tables, only `confirmed` uploads, in the admin's `position` order |
| `src/modules/media/interfaces/media.interfaces.ts` | New types `PublicImage`, `QuestionImageRefs`, `OwnedImageRow` |
| `src/modules/media/schemas/media-response.schemas.ts` | New `publicImageSchema` (id, url, width, height, altText) used by the student responses |
| `src/modules/media/media.module.ts` | Provides and exports `QuestionImagesService` |
| `src/modules/practice/practice.service.ts` | Adds image lists to each question and option, and `explanationImages` to answered questions and to the answer reply. On finish, calls the plan when every answer is right |
| `src/modules/practice/repositories/practice.repository.ts`, `interfaces/practice.interfaces.ts` | Item rows now carry `stimulusRevisionId`. Saving an answer also returns the revision id (new `AnswerFeedbackRow`) |
| `src/modules/practice/schemas/practice-response.schemas.ts` | New fields: `stemImages`, `passageImages`, option `images`, `explanationImages` (in `answer` and in the answer reply) |
| `src/modules/practice/practice.module.ts` | Imports `MediaModule` and `StudyPlanModule` |
| `src/modules/mock-exams/mock-exams.service.ts` | Adds `stemImages`, `passageImages` and option `images` during the exam, and `explanationImages` in the review only |
| `src/modules/mock-exams/repositories/mock-exams.repository.ts`, `interfaces/mock-exam.interfaces.ts`, `schemas/mock-exam-response.schemas.ts`, `mock-exams.module.ts` | Same wiring for mock exams (revision id of the passage, new response fields, `MediaModule` import) |
| `src/modules/study-plan/study-plan.service.ts` | New `completeForSession`, called by practice inside its transaction |
| `src/modules/study-plan/repositories/study-plan.repository.ts` | New `completeTodaysMatch`: one UPDATE that marks the best matching pending item of the student's own today as done |
| `src/modules/study-plan/study-plan.module.ts` | Exports `StudyPlanService` |
| `test/integration/image-fixtures.ts` (new) | Test helpers to add images, link them to a stem, explanation or option, and point the app at a fake CDN |
| `test/integration/practice.int-spec.ts` | 3 image tests, 4 plan tests, and the existing "right answer" test now also expects `explanationImages: []` |
| `test/integration/mock-exams.int-spec.ts` | 1 image test (exam and review) |
| `test/unit/practice.spec.ts` | Test row helper gets `stimulusRevisionId: null` |
| `architecture.md` | Notes in the question images section and the study plan section |

## Why

Students could not see the pictures their questions needed (for example a triangle in a geometry question). The upload side was done in the last PR, so this PR is the read side. The plan change comes from a request that a perfect practice should complete the day's planned session by itself, while "Tests" still lets the student practise again.

## How

**One query for all images**: Instead of one query per question, the service asks once for every question on the screen. It reads the stem and explanation images, the option images and the passage images together and groups them by owner. The database only keeps the S3 key, so `MediaService.publicUrl` builds the address (CDN if set, otherwise the bucket address).

**Same rule as explanations**: Explanation images follow the explanation text. Practice sends them only for an answered question and in the answer reply. Mock exams send them only in the review after submit. The exam payload has a test that scans for the explanation image and fails if it shows up early.

**Only finished uploads**: Images still `pending` (upload not confirmed) are never returned.

**Plan auto-complete**: When a practice session is finished and `score = total`, the same transaction marks one pending plan item of the student's own calendar day as done. A subject session completes the item for that subject, a planned topic before a whole-subject item (any topic when the session had no topic filter). A "retry incorrect" session completes the "Retry incorrect" item. The UPDATE uses `FOR UPDATE SKIP LOCKED` so two finishes at once cannot take the same item.

**Kept the same on purpose**: The response shapes only grew new fields, nothing was renamed, so older fields keep working. Finishing a practice with a wrong answer works exactly as before.

## Decisions made

**Mock exams do not auto-complete plan items.** The request was about practice tests, and a mock exam is a different kind of plan item with its own scoring. Left out until it is asked for.

**Only a perfect session completes a planned item.** A session with a wrong or skipped answer leaves the plan alone. The student can still tick it done by hand.

**Bookmark sessions never complete anything.** They are not tied to a subject or a planned topic.

**One item per finished session.** A session that fits two items completes only the more specific one, so one practice cannot clear a whole day.

**Images are part of the question response, not a new endpoint.** The app already loads these screens, so adding fields avoids a second request per question.

## Bugs found while testing (already there before, not fixed here)

**1. Analytics integration tests fail.** Three cases in `test/integration/analytics.int-spec.ts` fail: "buckets activity into reporting-timezone hours and weekdays", "says not available instead of showing zeros" and "never records parent sessions or parent events". I did not touch the analytics module and I did not run them against the code before my changes, so I cannot say for certain they were failing before.

**2. Some unit test files fail to load.** `test/unit/analytics-lib.spec.ts`, `assessment-shuffle-clock.spec.ts` and `mock-exams.spec.ts` fail with "Cannot read properties of undefined (reading 'config')", and `otp.service.spec.ts` fails one case. The last three already failed on the original code before the parent PR. `analytics-lib.spec.ts` shows the same error.

**3. The type-aware linter cannot start on my machine.** `npm run lint` fails because `tsgolint.exe` will not launch (it worked earlier the same day). It looks like a Windows policy block, so I did not work around it.

## Notes

- The tests run against a temporary PostgreSQL 18 on another port, because the local PostgreSQL 17 cannot load `plpgsql.dll` here (Windows blocks it). I did not run anything against the shared hosted database. No migration is needed for this PR.
- For real images, `S3_BUCKET`, `AWS_REGION` and optionally `CDN_BASE_URL` must be set, and the address must allow public reads of `questions/*` (see section 10k of `architecture.md`).
- `npm run typecheck` and Prettier on the changed files were clean.

## Next steps

- Check one real question with a real uploaded image on a device, against the real bucket or CDN.
- Decide if a finished mock exam should also complete a planned "Full-length mock exam" item.
- File tickets for the old bugs above.

## Test plan

**Automated tests (280 of 283 integration passing, 3 analytics cases failing and not related to this change):**
- [x] `test/integration/practice.int-spec.ts`: "sends stem and option images as public URLs, and hides the explanation images until answered" (order by position, answer reply carries the explanation image)
- [x] `test/integration/practice.int-spec.ts`: "leaves out images whose upload was never confirmed" and "gives questions without images empty lists"
- [x] `test/integration/practice.int-spec.ts`: "ticks off the matching planned session when every answer is right" (other topic and tomorrow stay pending, shows done in `GET /student/plan`)
- [x] `test/integration/practice.int-spec.ts`: "leaves the plan alone when an answer is wrong", "completes only one item, the planned topic before the whole subject", "does nothing when the same practice is done again later"
- [x] `test/integration/mock-exams.int-spec.ts`: "shows question and option images during the exam, and explanation images only in the review"
- [x] `npm run test:integration`: 280 passed, 3 failed (analytics, see above)
- [ ] `npm test` (unit): 4 spec files fail, none of them in code this PR changed (see above)
- [x] `npm run typecheck`: clean
- [ ] `npm run lint` - could not run, the type-aware linter does not start on this machine. The plain `oxlint` run only showed warnings in `analytics.int-spec.ts`

**Manual checks (local):**
- [ ] A real image from S3 or the CDN in the app - not done, the tests use a fake CDN address, so a real upload still has to be checked
- [ ] Retry-incorrect auto-complete - covered only by reading the code and the shared query, there is no integration test for that mode
