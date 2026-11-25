# 2026-11-23 - sheet-requirements (admin)

## Summary

This PR updates the Prepify Admin Panel to support multi-select questions, question explanations, and dynamic exam blueprints (requirements P1-010, P1-011, P1-012, and P1-013). Content authors can now select between single-choice and multiple-choice question formats, mark multiple options as correct using checkboxes, author rich explanations, and define dynamic exam section blueprints with an interactive Blueprint Preview dialog.

Checked live: all 90 Vitest unit tests pass and the production build completes cleanly. Auto-saving before blueprint preview was validated to ensure live preview calculations always reflect current unsaved edits.

## What changed

| File | Change |
|---|---|
| `src/features/exams/components/blueprint-preview-dialog.tsx` (new) | Interactive modal dialog that queries the backend blueprint preview endpoint, shows rule coverage percentages, lists sample questions, and flags insufficient question pool warnings. |
| `src/features/cooldown/` (new) | UI components and helper types for displaying question cooldown rules. |
| `src/components/common/options-editor.tsx` | Enhanced options editor to render checkboxes when `questionType === 'multiple_choice'` and radio buttons when `single_choice`. Added validation ensuring at least one option is marked correct. |
| `src/features/content/components/question-form.tsx` | Added question type dropdown, explanation textarea with character counts, and connected multi-select options state to the form lifecycle. |
| `src/features/content/pages/question-detail-page.tsx` | Updated question detail view to display the question type, highlight all correct options, and render the explanation box. |
| `src/features/content/pages/questions-page.tsx` | Added filter controls for question format (all, single choice, multiple choice) and explanation presence. |
| `src/features/content/schemas/content.schemas.ts`, `lib/question-mapping.ts` | Updated Zod schemas and mapping utilities to parse and serialize `questionType` (`single_choice` \| `multiple_choice`) and `explanation`. |
| `src/features/exams/components/section-editor.tsx` | Section blueprint editor allowing admins to define rules specifying target question counts per subject, topic, and difficulty tier. |
| `src/features/exams/pages/exam-editor-page.tsx` | Added Blueprint Preview action button. Ensured `await save()` automatically runs before opening the preview dialog so that edits in flight are included in the preview payload. |
| `src/features/exams/lib/exam-editor.ts`, `lib/exam-editor.test.ts` | Helper logic for blueprint rule validation, total question summation, and unit test coverage. |
| `src/features/exams/schemas/exams.schemas.ts`, `api/exams.api.ts`, `api/exams.queries.ts` | Added API query hooks and mutation contracts for blueprint preview and section configuration. |

## Why

1. **Question formats (P1-010, P1-011)**: Standardized exams such as SAT and ACT contain questions where students must select one or more correct answers. The content authoring tool previously only permitted a single radio button selection.
2. **Explanations (P1-012)**: Explanations provide crucial learning feedback when students review mistakes. Content creators needed a dedicated field to write clear explanations.
3. **Exam blueprints (P1-013)**: Large exams are generated from dynamic blueprints rather than static question lists. Admins need to define selection rules and verify whether existing published questions satisfy the blueprint criteria before publishing an exam.

## How

**Conditional options UI**: The `OptionsEditor` component inspects the current `questionType`. When set to `multiple_choice`, each option card renders a square checkbox, allowing several options to be designated as correct. In `single_choice` mode, selecting an option automatically unchecks any previously selected option.

**Auto-save before preview**: Generating a blueprint preview requires sending the exam's current section structure and blueprint rules to `/admin/exams/:id/blueprint-preview`. In `ExamEditorPage`, clicking "Preview Blueprint" first executes `await save()` if form state is dirty, guaranteeing the preview accurately reflects the current configuration.

**Validation and guard rails**: Forms prevent publishing questions if multiple-choice is selected without at least one correct option, or if single-choice is selected with zero correct options.

## Decisions made

**Single-choice switch clears extra correct options.** When switching a question form from `multiple_choice` back to `single_choice`, any secondary correct options are safely truncated to keep data models strictly consistent.

**Auto-saving before preview instead of sending local state.** We chose to persist draft state to the backend rather than constructing an ad-hoc local preview payload, ensuring the preview engine runs the exact same SQL logic used during live student mock exams.

## Test plan

**Automated tests (all 90 passing):**
- [x] `src/features/exams/lib/exam-editor.test.ts`: tests blueprint rule aggregation, section question summation, and blueprint validation errors.
- [x] `src/features/content/pages/question-detail-page.test.tsx`: tests question stem, options, and explanation rendering.
- [x] `src/features/content/components/image-uploader.test.tsx`: tests media upload interactions.
- [x] `npm run test`: all 12 test suites passing (90 tests total).

**Manual checks (local):**
- [x] Verified creating a multiple-choice question and marking 2 options correct.
- [x] Verified saving question with explanation and viewing it in question detail.
- [x] Verified configuring section blueprint rules and clicking Preview Blueprint opens the dialog with valid question counts.
