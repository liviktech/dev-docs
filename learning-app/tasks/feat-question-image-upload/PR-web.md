# 2026-11-23 - question-image-upload (web)

## Summary

The student web app showed only the text of a question. Images that an admin attached (diagrams, graphs) were not shown anywhere. This PR shows them in practice, mock exams and both review pages: on the question, on the passage, on each answer option and on the explanation.

Checked live: unit tests pass (51 of 51 across 10 files) and typecheck shows no errors. The pages were **not** opened in a browser, so no image was seen rendering on screen yet.

## What changed

| File | Change |
|---|---|
| `src/components/practice/question-images.tsx` (new) | Shared component. Shows a list of pictures with their description (alt text), the stored width and height so the page does not jump while loading, lazy loading, and nothing when the list is empty. |
| `src/components/practice/question-images.test.tsx` (new) | Tests: pictures appear with alt text, an empty list renders nothing, and an option with only a picture shows it and is named by its alt text. |
| `src/lib/practice/api.ts` | New `PublicImage` type. Questions have `stemImages` and `passageImages`, options have `images`, and answer feedback has `explanationImages`. |
| `src/lib/mock-exam/api.ts` | Same fields on exam questions, options and review questions (`explanationImages`). |
| `src/components/practice/answer-option.tsx` | New `images` prop. The picture shows under the option text. An option with no text is allowed. The accessible label uses the text, or the picture descriptions when there is no text. |
| `src/components/practice/passage-card.tsx` | New `images` prop, shown under the passage text. |
| `src/components/practice/explanation-card.tsx` | New `images` prop, shown under the explanation. |
| `src/components/practice/question-runner.tsx` | Shows the question images under the question, and passes passage, option and explanation images on. |
| `src/components/mock-exam/exam-runner.tsx` | Same for the exam. There is no explanation during an exam. |
| `src/components/practice/review-item.tsx`, `src/components/mock-exam/exam-review-item.tsx` | Same for the two review pages, including explanation images. |

## Why

Admins can now attach images to questions, and the backend returns them to the student endpoints. Many SAT and ACT math questions cannot be answered without the figure, so the web app has to show it.

## How

**One shared component**: every place that shows a picture uses `QuestionImages`, so the look and the loading behaviour are the same everywhere.

**Plain `<img>`, not `next/image`**: the files already sit on the CDN and the server sends the full URL, so the Next.js image optimizer adds nothing, and no `remotePatterns` setting is needed. Width and height from the server reserve the space.

**Explanation images follow the explanation**: they are shown only where the explanation is shown, which is after the student has answered. The backend does not send them before that.

**Kept the same on purpose**: the text rendering (plain text with line breaks), the keyboard shortcuts, and every existing prop. The new props are optional, so other callers did not change.

## Decisions made

**No markdown or math rendering.** This PR is only about pictures. Question text is still shown as plain text.

**Images are not shown in the stats and bookmarked lists.** Those lists show a short part of the stem. A "has image" icon could be added later.

## Notes

- This needs the backend change in `PR-backend.md`. Without it the new fields are missing and no pictures show (the page still works, because every new prop is optional where it is used).
- The new fields are typed as required in `lib/practice/api.ts` and `lib/mock-exam/api.ts`. The real backend always sends them. Older mock data would need them added.
- Images load only if `CDN_BASE_URL` serves the files publicly. During admin testing the bucket was private and returned 403, so this must be fixed (CloudFront or public read on `questions/*`) before pictures show.
- The mobile app has no repository in this workspace, so there is no `PR-mobile.md`.
- Lint was not run in this session.

## Next steps

- Open a practice session and an exam with an image question and look at it on a phone width and a wide screen.
- Add a click to enlarge for large figures.
- Add a "has image" mark in the stats and bookmarked lists.

## Test plan

**Automated tests (51 passing):**
- [x] `src/components/practice/question-images.test.tsx`: pictures show with alt text and the right `src`
- [x] `src/components/practice/question-images.test.tsx`: an empty list renders nothing
- [x] `src/components/practice/question-images.test.tsx`: an option with only a picture shows it and is named by its alt text
- [x] `npm run typecheck`: no errors
- [ ] `npm run lint`: not run in this session

**Manual checks (local):**
- [ ] Practice session with an image question: question, option and explanation images - not run in a browser
- [ ] Mock exam and both review pages with images - not run in a browser
- [ ] Images loading from `CDN_BASE_URL` - not possible yet, the bucket was private and returned 403
