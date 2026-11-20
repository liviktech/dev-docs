# 2026-11-20 - feat-mock-exams (admin)

## Summary

The admin panel had no way to build mock exams. This PR adds an Exams area: a list page and a create/edit page where an admin picks published questions, chooses one timer for the whole exam or a timer per section, and saves a draft or publishes. It also adds a "Shuffle answer options" checkbox to the question form, and an optional `shuffle_options` column to the import file.

It needs the backend from `PR-learning-app-backend.md` (routes under `/admin/mock-exams`). It was tested with mocked responses only; I did not run it against the real backend in a browser.

## What changed

| File | Change |
|---|---|
| `src/features/exams/schemas/exams.schemas.ts` (new) | Response shapes, the request body type, the time presets (30, 45, 60, 90, 120) |
| `src/features/exams/api/exams.api.ts`, `exams.queries.ts` (new) | Requests and react-query hooks: list, get, create, save, publish or withdraw, archive |
| `src/features/exams/lib/exam-editor.ts` (new) | The editor state, what is missing before save, turning the state into the request body, moving questions |
| `src/features/exams/components/duration-field.tsx` (new) | Preset buttons plus a custom minutes box |
| `src/features/exams/components/question-picker.tsx` (new) | Pop-up to search published questions of the exam and add several at once |
| `src/features/exams/components/section-editor.tsx` (new) | One section: subject, time (section timers only), ordered question list with move and remove |
| `src/features/exams/pages/exams-page.tsx` (new) | Exam list with filters, paging, publish or withdraw, archive |
| `src/features/exams/pages/exam-editor-page.tsx` (new) | New and edit exam page |
| `src/routes/routes.tsx`, `src/layouts/dashboard-layout.tsx` | `/exams`, `/exams/new`, `/exams/:id/edit` and an "Exams" menu item |
| `src/components/common/status-badge.tsx` | A "Withdrawn" status |
| `src/features/content/**` | `shuffleOptions` in the question form, mapping, schemas and API body; import file reads an optional `shuffle_options` column |
| `src/features/exams/**/*.test.*` (new) | Editor logic and page tests |

## Why

Admins need to prepare exam papers and set the time; students can only take published exams. Options that read like "All of the above" must keep their order, so admins need a switch for that.

## How

**Timing.** Two cards switch between one timer and a timer per section. Only the time that matches the chosen mode is sent to the backend, so a leftover value cannot be saved.

**Adding questions.** The picker lists published questions of the chosen exam, starts on the section's subject, and greys out questions already used in any section.

**Saving a live exam.** The page says that saving makes a new draft and students keep the current version until it is published. The list shows "Has a draft that is not published" and its button publishes the draft.

**Import file.** A blank or missing `shuffle_options` value is not sent, so the backend decides from the option texts.

## Decisions made

**Changing the exam (SAT or ACT) is not allowed after saving.** The backend refuses it, and questions belong to one exam. On a new exam, changing it clears the sections.

**The import template download was not changed.** `shuffle_options` is read if the column exists, but the template file still has the old columns.

## Notes

- Section names cannot be typed; a section is its subject. The backend has no name column for sections.
- Lint shows 3 existing warnings in `components/ui` files. None are in new code.

## Next steps

- Add a `shuffle_options` column to the import template and the import help text.
- Try the screens against the running backend in a browser.

## Test plan

**Automated tests (all passing):**
- [x] `src/features/exams/lib/exam-editor.test.ts`: what is missing before save, request body for each timing mode, time totals, moving questions
- [x] `src/features/exams/pages/exams-pages.test.tsx`: list, empty state, withdraw sends the request, a failed publish leaves it a draft, an incomplete exam is not saved, timing switch shows the right time fields
- [x] `npx vitest run`: 50 passing
- [x] `npx tsc -b`: clean. `npx oxlint`: only the 3 existing warnings

**Manual checks (local):**
- [ ] Building and publishing an exam in a browser against the real backend - not done
- [ ] The question picker dialog and the editor layout on screen - not looked at
