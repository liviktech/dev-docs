# Plan: question upload in the admin panel

## What we are building

The admin can add questions for **SAT** and **ACT**, one by one or in bulk from a CSV or Excel file. Every question is tagged with its **exam**, **subject** (Math, English, Science), **category** (Algebra, Trigonometry) and **difficulty** (easy, medium, hard). The admin can also manage the subjects and categories from the panel. The mobile app will later ask the backend for "SAT, Math, Algebra, medium" and get the right questions, with no sorting logic in the app.

This is two repos: `learning-app-backend` (new admin APIs and one migration) and `learning-app-admin` (new screens and reusable components). Nothing is built yet. This file is the plan only.

## Big finding: the database already has most of this

Migration `0001` already created a full content model. We reuse it and do not invent new tables.

| Need | What already exists | Notes |
|---|---|---|
| Exams | `exams` table, `exam_code` enum (`sat`, `act`) | **The table is empty.** Nothing inserts SAT and ACT yet. The seed in `seeds/0001` inserts them so they appear by default. |
| Subject and category | `taxonomy_nodes`, a tree per exam. Level 1 `section`, level 2 `domain`, level 3 `skill`. | Subject = `section`, category = `domain`. The `skill` level stays unused for now and is available if the client wants a third level later. |
| Question | `questions` (exam, `external_key`, archived flag) and `question_revisions` (stem, explanation, difficulty, calculator flag, status `draft`, `published`, `retired`) | Published revisions are never edited. An edit makes a new revision. This keeps student history correct. |
| Options | `question_options` (text, `is_correct`, `position`) | There is no answer letter that matters. The API can shuffle `position` on every call and relabel A, B, C, D. |
| Explanation | `question_revisions.explanation_markdown`, nullable | "Schema now, data later" is already true. |
| Passages | `stimuli` and `stimulus_revisions`, linked from a question revision | Answers "yes" to the shared passages question. |
| Images | `media_assets`, `question_revision_assets`, `question_option_assets`, `stimulus_revision_assets` | Answers "schema only for now". **Nothing to build for images in this task.** |
| Tag a question | `question_revision_skills` (question revision to taxonomy node, one primary) | We tag each question with its category node. |
| Publish checks | A database trigger: to publish, a revision needs at least 2 options, exactly 1 correct option and at least 1 mapped node | See the safety points below. |

## Decisions already made

| Question | Answer |
|---|---|
| Exams | Two fixed exams, **SAT and ACT**, always present. Admins cannot add or remove exams. |
| Subjects and categories | **Managed by the admin in the panel** (add, rename, reorder, archive). The starting list is seeded. |
| Bulk upload | **CSV and Excel (.xlsx)**, with a check before import. |
| Publishing | **Admin publishes directly.** No review step. Draft is still available. |
| Images | Schema only (already there). Upload comes in a later phase. |
| Passages | Supported. Several questions can share one passage. |
| Admin role | One role, `admin`, same as the login task. |
| Difficulty | **Three levels** (easy, medium, hard). Confirmed by the client. |
| Starting subjects and categories | The client sent the full list. It is seeded as is (see "The seed file"). Nothing else is seeded. |
| Bulk file format | No existing format from the client, so we design the columns ourselves (see "Bulk file format"). |

## Defaults I picked (change them if you disagree)

| Topic | Default | Why |
|---|---|---|
| Names in the panel vs the database | Panel says **Subject** and **Category**. The database keeps `section` and `domain`. | No migration needed just to rename. |
| Difficulty values | A migration changes the allowed values from 1 to 5 to **1, 2, 3** (1 easy, 2 medium, 3 hard). The column is empty, so this is safe. | The client confirmed three levels. Five numbers with three labels would be confusing. |
| Answers | Exactly **one correct option** for now. | The publish trigger enforces it today. Multiple answers later means relaxing that trigger and nothing else, because `is_correct` is already per option. |
| Option count | 2 to 5 options (A to E). | Matches the table. SAT uses 4, ACT uses 4 or 5. |
| Archive instead of delete | Subjects, categories and questions are archived. A hard delete is allowed only when nothing uses the item. | Past student answers point at them. |
| Archiving a parent | Refused while it still has active children. Archive the children first. | Simple and predictable. |
| Archiving something with published questions | Allowed after a confirm dialog that shows the question count. | The admin should know what disappears from the app. |
| Editing a published question | Creates a new revision. If "publish" is chosen, the old revision becomes `retired` and the new one `published` in one transaction. | Matches the table rules. Old attempts keep the old text. |
| Bulk upload size | Max **500 rows** per file, sent to the API in **chunks of 25**. | The backend JSON body limit is 100 kb (`BODY_SIZE_LIMIT`). Chunks keep each request small without touching the limit. |
| Bulk parse location | The **browser** reads the CSV or Excel file and sends plain JSON. The server never parses spreadsheet files. | Smaller attack surface on the server, and one Zod schema checks both single and bulk input. The server still validates every row. |
| Duplicates in bulk | A row whose `external_key` already exists is reported as a duplicate and skipped. An identical stem in the same exam is a warning, not an error. | `external_key` is already unique in the table. |
| Text format | Stem, options, explanation and passage are stored as **markdown** (the columns are already `*_markdown`). The panel shows a safe preview and never renders raw HTML. | Leaves room for formulas and images later. |
| Spreadsheet libraries | **Chosen:** `papaparse` for CSV and `read-excel-file` (browser build) for .xlsx. Both are maintained, and the old `xlsx` package was avoided. | Checked on 2026-11-18. `npm audit` still reports older issues in the project that are not from these two. |

## The data model in simple words

```text
Exam (SAT or ACT)
  Subject (Math, English, Science)            <- taxonomy_nodes, level 1
    Category (Algebra, Trigonometry)           <- taxonomy_nodes, level 2
      Question  -> has difficulty (easy, medium, hard)
        Options (A to E, one is correct)
        Explanation (can be empty for now)
        Passage (optional, can be shared by many questions)
```

A question is tagged to **one category** (its primary node). Its subject comes from the category's parent. The question also stores its exam, and the service checks that the category belongs to the same exam.

## Part 1: backend (`learning-app-backend`)

### Migrations (written by the agent, **run by the developer only**)

| File | What it does | State |
|---|---|---|
| `migrations/0006_question-difficulty-three-levels.sql` | Replaces the difficulty check with `BETWEEN 1 AND 3`. | **Written.** Not run. |
| `seeds/0001_exams-and-taxonomy.sql` | Seeds the two exams, 7 subjects and 73 categories from the client's list. | **Written.** Not run, not tested against a database yet. |

The difficulty change took the number `0006` because the seed moved out of the migrations folder (see below).

### The seed file

The seed lives in its own folder, `src/database/seeds/`, with its own numbering and its own tracking table (`pgmigrations_seed`). Two new scripts run it: `npm run seed:up` and `npm run seed:down`. Order on a new machine: `migrate:up`, then `seed:up`. This keeps starting data apart from schema changes.

`seeds/0001` contains **only** the client's list and nothing else. It is safe to run twice because existing rows are skipped. The down step does nothing on purpose: deleting seed rows could remove things admins added or questions already use. Re-run safety means `up, down, up` works.

| Exam | Subject | Categories |
|---|---|---|
| ACT | English | 8 |
| ACT | Mathematics | 13 |
| ACT | Reading | 9 |
| ACT | Science (optional) | 8 |
| ACT | Writing (optional) | 5 |
| SAT | Reading and Writing | 13 |
| SAT | Mathematics | 17 |

That is 2 exams, 7 subjects and 73 categories. Notes:

- "Optional" for ACT Science and Writing is stored in the subject's description ("Optional section"). The table has no optional flag, and we do not need one yet.
- Subject and category codes are made from the names (for example `authors-purpose-and-tone`). The same category name can appear under different subjects (for example "Data Interpretation" in ACT Mathematics and ACT Science, or "Author's Purpose and Tone" in ACT English and ACT Reading) because codes only need to be unique inside their parent.
- Order in the panel follows the order in the client's list.
- SAT and ACT Mathematics are separate rows, as planned.
- The panel can still add, rename, reorder and archive anything after seeding.

### Endpoints (all under `/api/v1/admin/...`, all behind `AdminRoleGuard`)

**Exams and the subject and category tree**

| Method | Path | What it does |
|---|---|---|
| GET | `/admin/exams` | The two exams. |
| GET | `/admin/taxonomy?exam=sat` | The whole tree for one exam: subjects, their categories, and a question count on each. Archived items included only with `includeArchived=true`. |
| POST | `/admin/taxonomy/nodes` | Add a subject (no parent) or a category (parent is a subject). The level is worked out from the parent, never sent by the client. |
| PATCH | `/admin/taxonomy/nodes/:id` | Rename, change description, change order. |
| POST | `/admin/taxonomy/nodes/:id/archive` and `/restore` | Archive or bring back. |
| DELETE | `/admin/taxonomy/nodes/:id` | Only when it has no children and no questions, else 409. |

**Passages**

| Method | Path | What it does |
|---|---|---|
| GET | `/admin/passages?exam=sat` | List, for the picker in the question form. |
| POST | `/admin/passages` | Create a passage (a `stimulus` plus its first published revision). |

Editing a passage is out of scope. The answer for now is to create a new one.

**Questions**

| Method | Path | What it does |
|---|---|---|
| GET | `/admin/questions` | Paged list. Filters: exam, subject, category, difficulty, status, text search. |
| GET | `/admin/questions/:id` | One question with its current revision, options and tags. |
| POST | `/admin/questions` | Create. Body has `publish: true or false`. |
| PATCH | `/admin/questions/:id` | Edit (makes a new revision, see defaults). |
| POST | `/admin/questions/:id/archive` and `/restore` | Archive or bring back. |
| POST | `/admin/questions/bulk` | Body has `rows` (max 25) and `dryRun`. With `dryRun` it only checks and reports per row. |

List endpoints use a capped page size (the backend rule). The response shape is the usual `{ success, message, data, requestId }`. Swagger tag names: `Admin - Taxonomy`, `Admin - Passages`, `Admin - Questions`. Throttling: only the general limit applies for now. A stricter limit for the bulk route was left out because the existing throttle policies are auth specific; add one if abuse becomes a concern.

**Built (backend).** All of the above exists in `src/modules/taxonomy`, `passages` and `questions`. Decisions made while building: the edit endpoint takes the whole question again and cannot change exam or external key; editing a published question always publishes the new revision (no unpublish, archive instead); a subject or category with the same name as a sibling gets 409; bulk rows are parsed per row so one bad row does not fail the request; `dryRun` defaults to true. Details are in `architecture.md` section 10c.

### Code layout

New modules in `src/modules/`: `taxonomy`, `passages`, `questions`, each with controller, service, repository, schemas, DTOs and interfaces, following the structure `auth` already uses. The bulk logic is in `questions` and calls the same service method as single create, so the two cannot drift apart.

### Safety points to keep

- **The publish trigger runs on `UPDATE`, not on `INSERT`.** If we insert a revision directly as `published`, the checks are skipped. So the service always inserts a `draft`, adds options and the tag, then updates the status to `published`, all in one transaction. A test must prove that a bad question cannot be published.
- Validate in the service that the chosen category belongs to the same exam as the question, that it is a category (not a subject), and that it is active. The database does not check this.
- The question list API for admins may show `is_correct`. The future mobile API **must not** send it with the question. It checks the answer on submit.
- Unknown body fields are rejected (strict Zod objects). `createdBy` always comes from the logged in admin, never from the body.
- Markdown fields are stored as text and never turned into HTML on the server.
- No logging of question text in bulk error output beyond row numbers and field names.

### Backend tests

Integration tests per module:

- Taxonomy: add subject, add category, wrong parent rejected, rename, reorder, archive refused with active children, archive with questions works, delete refused when used, student token gets 403.
- Questions: create as draft, create and publish, publish refused with no correct option or two correct options, category from the other exam rejected, edit published creates revision 2 and retires revision 1, archive and restore, list filters and paging.
- Bulk: dry run reports row errors without writing, a clean file imports, duplicate `external_key` is skipped, a failed row does not stop the report for other rows.
- Migration: `migrate:up`, `migrate:down`, `migrate:up`.

Then: `npm run typecheck`, `npm run lint`, `npm test`, `npm run test:integration`. Update `architecture.md` in the same change (new endpoints, seeded exams, difficulty values, the publish flow).

## Part 2: admin panel (`learning-app-admin`)

### Screens

| Route | Screen |
|---|---|
| `/questions` | Question list with filters (exam, subject, category, difficulty, status), search, paging, and buttons to add, import, edit and archive. |
| `/questions/new` and `/questions/:id/edit` | The question form. Exam, then subject, then category as linked dropdowns. Difficulty. Calculator allowed. Optional passage (pick or create). Stem, options A to E with one marked correct, explanation (optional). Buttons: **Save as draft** and **Publish**. A preview shows how the markdown looks. |
| `/questions/import` | Step 1 pick the exam and download the template. Step 2 upload CSV or Excel. Step 3 see a table of rows with errors and warnings. Step 4 import the clean rows. Shows a result summary at the end. |
| `/subjects` | "Subjects and categories". Exam tabs (SAT, ACT). A tree with add, rename, reorder, archive and restore, and a question count on each line. |

The dashboard layout gets a **sidebar** (Questions, Subjects and categories) next to the existing top bar.

### Bulk file format

One row is one question. Columns:

`external_key` (optional but recommended, makes re-uploads safe), `subject`, `category`, `difficulty` (easy, medium or hard), `calculator_allowed` (yes or no), `passage_key` and `passage_text` (optional, rows with the same key share one passage; the panel creates each passage once through `POST /admin/passages` before the import and sends its id with the rows, because rows of one passage can fall in different chunks), `stem`, `option_a` to `option_e`, `correct` (A to E), `explanation` (optional).

The exam is chosen once on the screen, because one file is for one exam. Because there is no existing client format, this layout is ours and can change before the first real upload. Subject and category are matched by name against the managed list. An unknown name is a row error that says which name was not found. The template file is generated from the real list so the names are right.

### New reusable components (`components/common`)

The rule stays: pages never use raw `<table>`, `<select>`, `<textarea>` and so on. These are added first, then used.

| Component | Used for |
|---|---|
| `DataTable` (a plain shadcn table, no TanStack Table needed yet) with `Pagination` | Question list, import preview, passages. |
| `SelectField` | Exam, subject, category, difficulty. |
| `TextareaField` | Stem, explanation, passage. |
| `CheckboxField`, `RadioGroupField` | Calculator flag, choosing the correct option. |
| `OptionsEditor` | The A to E options with a correct marker. |
| `MarkdownPreview` | Safe preview, no raw HTML. |
| `StatusBadge` | Draft, published, archived. |
| `ConfirmDialog` | Archive and delete confirms. |
| `FormDialog` | Add and rename in the subjects screen. |
| `EmptyState` | Empty lists. |
| `Tabs` | Exam switch. |
| `FileInput` | CSV and Excel picking. |
| `AppSidebar` | Navigation. |

### API layer

Feature folders `features/questions`, `features/taxonomy`, `features/passages`, each with `api/` (request functions and TanStack Query hooks), `components/`, `pages/`, `schemas/`. Every call goes through `apiRequest`. Query keys are defined in one place per feature. Mutations invalidate the right lists. The subject and category lists are cached because many screens use them.

The form schema and the import row schema share the same rules as the backend Zod schemas (same limits, same messages where possible).

### Admin panel checks

`npm run typecheck`, `npm run lint`, `npm run build`, `npm test`. Component tests for the table, the options editor and the form. Page tests for the form (draft and publish, server errors) and the import screen (a file with good and bad rows). A manual run in the browser against the real backend: add subjects and categories, add a question as draft, publish it, edit it, import a CSV and an Excel file with errors and then a clean one, archive a category.

### Built (admin panel)

All screens exist: `/subjects`, `/questions`, `/questions/new`, `/questions/:id/edit`, `/questions/import`, with a side menu. Differences from the plan:

- The side menu is a small custom `AppSidebar` (links), not the full shadcn sidebar.
- `MarkdownPreview` shows safe plain text with line breaks. Real markdown and formulas come later.
- Reorder is a move up and down button that swaps two positions.
- Import: the browser checks every row with `dryRun`, then creates each passage once and sends the clean rows. "Publish now" is a checkbox (default on).
- New common components: SelectInput, SelectField, TextareaField, CheckboxField, StatusBadge, ConfirmDialog, FormDialog, EmptyState, Pagination, DataTable, FileInput, MarkdownPreview, OptionsEditor, AppSidebar, ExamTabs, LinkButton.
- Checks: `npm run typecheck`, `npm test` (35 tests) and `npm run build` pass. **Not checked in a browser against the real backend**, because the migration and seed have not been run yet.

## Order of work

1. Backend: difficulty migration `0006` and the seed in `seeds/` (both written), taxonomy, passages and questions modules, bulk import (**all built**, typecheck, lint and unit tests pass). The integration tests are written but not run yet.
2. Admin panel: reusable components, subjects, question form and list, bulk import (**all built**, see above).
3. End to end check in the browser. Fix and note anything that differs from this plan.

Both repos use the branch `feat-question-upload`. The agent does not commit or push, and does not run migrations. The developer runs `migrate:up`, `seed:up` and the integration tests.

## Out of scope on purpose

- **Image upload.** The tables exist. Storage and upload come in the next phase.
- **Math formula rendering.** Text is stored as markdown, so formulas can be added with the images phase.
- **Editing a passage** and deleting a passage.
- **Multiple correct answers.** Schema supports it, but the publish rule still says exactly one.
- **Category weightage for exams** (percent of questions per category). Separate config task.
- **Mobile APIs** that serve questions. Planned later, see below.
- **Approval flow for questions.** Admins publish directly.
- **Importing images or explanations in bulk** beyond the plain text columns.

## What this prepares for the mobile app (not built here)

- The mobile API will filter by exam, subject or categories and difficulty on the server, so the app has no sorting logic.
- It will shuffle option order on every call and relabel A, B, C, D, and will never send `is_correct`.
- "Do not repeat the same questions for a while" can use `assessment_attempt_items`, which already records every delivered question per student, so no new table is needed for it.
- Exam mode will use category weightage from admin config. Practice mode will let students pick several categories.

## Risks and unknowns

- **Empty `exams` table.** Until `npm run seed:up` is run, nothing works. The developer must run it after `migrate:up`, and the README setup says so.
- **The seed migration has not been run.** It was written to match the real table and index definitions, but it is unchecked against a database. The developer should run `seed:up`, then `seed:down` and `seed:up` again, and count the rows (2 exams, 7 subjects, 73 categories).
- **Difficulty constraint** (migration `0006`) is easy to change now and hard once data exists. Run it before any question is saved.
- **Publish trigger bypass** on `INSERT`, described above. This is the most important test.
- **Chunked bulk import is not all or nothing.** The dry run checks every chunk first and the import only starts if it is clean. If a chunk still fails in the middle (for example the connection drops), earlier chunks are saved. Re-uploading is safe because `external_key` skips what exists, so we say "use external_key" in the template.
- **Spreadsheet parsing in the browser** can be slow for big files and is a source of odd data (dates, numbers in text cells). The row schema must coerce and report clearly.
- **Large passages** can push one bulk chunk over the 100 kb body limit. The panel must split by size as well as by count.
- **Library versions** (TanStack Table, shadcn components, the spreadsheet library) move fast. Check real docs at build time and note any difference.
- **Docs drift.** `architecture.md` must be updated in the same change.

## Answered by the client

- **Subjects and categories:** the full ACT and SAT list was sent and is seeded as is.
- **Difficulty:** three levels are enough.
- **File format:** no existing format. We use the columns in this plan, and the panel gives a template to download.

## Still open

- **Who prepares the CSV and Excel files.** If someone already has questions in a spreadsheet, a sample would let us check the columns against real data.
- **Duplicate wording.** The client list has some names under more than one subject (for example "Data Interpretation" and "Author's Purpose and Tone"). They are kept as separate categories per subject. Tell us if the client wants them shared instead.
