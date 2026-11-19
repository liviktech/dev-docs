# 2026-11-18 - question-upload (backend)

## Summary

The database already had tables for exams, subjects, categories, questions, options and passages, but the API had no way to fill them. Nothing could add SAT or ACT questions, and the `exams` table was empty. This PR adds the admin APIs for subjects and categories, passages and questions (single and bulk), a migration that cuts difficulty to three levels, and a seed with the client's SAT and ACT list (2 exams, 7 subjects, 73 categories).

## What changed

| File | Change |
|---|---|
| `src/modules/taxonomy/` (new) | Controller, service, repository, schemas, DTOs and interfaces for `/admin/exams` and `/admin/taxonomy`. Add, rename, reorder, archive, restore and delete subjects and categories. |
| `src/modules/passages/` (new) | `GET` and `POST /admin/passages`. Creates a passage with its first published revision. |
| `src/modules/questions/` (new) | `/admin/questions` list, get, create, edit, archive and restore, plus `bulk-questions.service.ts` for `POST /admin/questions/bulk`. |
| `src/common/schemas/pagination.schemas.ts` (new) | Shared page and page size rules with a capped page size. |
| `src/database/migrations/0006_question-difficulty-three-levels.sql` (new) | Changes the difficulty check to allow only 1, 2 and 3. |
| `src/database/seeds/0001_exams-and-taxonomy.sql` (new) | Seeds SAT and ACT, 7 subjects and 73 categories from the client's list. Safe to run twice. The down step does nothing on purpose. |
| `package.json` | Adds `seed:up` and `seed:down`. They use their own folder and their own tracking table, `pgmigrations_seed`. |
| `src/app.module.ts` | Registers `TaxonomyModule`, `PassagesModule` and `QuestionsModule`. |
| `src/shared/http/configure-http.ts` | CORS now allows `PATCH` and `DELETE` as well as `GET` and `POST`. |
| `README.md` | Setup now says to run `npm run seed:up` after `migrate:up`, and lists the two new scripts. |
| `architecture.md` | New endpoints, seeded exams, difficulty values and the publish flow (section 10c). |
| `test/unit/question-upload.spec.ts` (new) | Unit tests for the pure parts (see Test plan). |
| `test/integration/question-upload.int-spec.ts` (new) | Integration tests for access, taxonomy, questions and bulk. |

## Why

The client's Nov 16 notes ask for question upload for SAT and ACT, with exam, subject, category and difficulty on every question, one by one or in bulk. The real work was the publish flow, because the database has a trigger that checks a revision only when it is updated to `published`. The easy part was the schema, because most of it was already in migration `0001`.

## How

**Reuse the existing tables**: Subject is `taxonomy_nodes` level `section` and category is level `domain`. The panel says Subject and Category and the database names stay as they are, so no rename migration is needed. A question is tagged to one category and its subject comes from the parent.

**Publish flow**: The publish trigger runs on `UPDATE` only, so inserting a revision as `published` would skip the checks. The service always inserts a draft, adds the options and the tag, then updates the status to `published`, all in one transaction. Editing a published question creates revision 2 and retires revision 1 in the same transaction, so old student attempts keep the old text.

**Checks the database does not do**: The service checks that the category belongs to the same exam, is a category (not a subject) and is active.

**Bulk import**: Each row is checked on its own, so one bad row does not fail the request. `dryRun` defaults to true. A row whose `external_key` already exists is reported as a duplicate and skipped. A request takes up to 25 rows and calls the same service method as single create, so the two cannot drift apart.

**Seed in its own folder**: `seed:up` and `seed:down` use `src/database/seeds` and their own tracking table, so starting data stays apart from schema changes. Order on a new machine is `migrate:up`, then `seed:up`. Subject and category codes come from the names, and the same category name can sit under different subjects.

**Kept the same on purpose**: Strict Zod objects reject unknown fields. `createdBy` always comes from the logged in admin. All routes are behind `AdminRoleGuard`. Question text is stored as markdown and is never turned into HTML on the server.

## Decisions made

**Three difficulty levels.** The client confirmed three levels. The migration restricts the values to 1, 2 and 3 instead of keeping five numbers with three labels. The column is empty, so this is safe now and hard to change once data exists.

**Editing a published question always publishes the new revision.** There is no unpublish. To take a question out of the app, archive it.

**Edit takes the whole question again and cannot change the exam or the external key.** A different exam would break the category check, and the key is the duplicate guard.

**A subject or category with the same name as a sibling gets 409.** Names are matched when importing a file, so two siblings with the same name would be ambiguous.

**Archive instead of delete.** Delete is allowed only when nothing uses the item, otherwise 409. Archiving a parent is refused while it has active children.

**No stricter throttle on the bulk route.** The existing throttle policies are auth specific, so only the general limit applies. Add one if abuse becomes a concern.

**CORS now allows `PATCH` and `DELETE`.** It allowed only `GET` and `POST`, so the browser blocked the reorder arrows (which call `PATCH`), and also rename and delete in the Subjects page. I found this by reading the code and did not reproduce it in a browser. It needs a backend restart.

## Notes

- Swagger document generation for the new tags (`Admin - Taxonomy`, `Admin - Passages`, `Admin - Questions`) was not checked.
- ACT Science and Writing are marked optional in the subject description (`Optional section`), because the table has no optional flag.
- The admin API may return `is_correct`. The future mobile API must not send it with a question and must check the answer on submit.

## Next steps

- Ask the client who prepares the CSV and Excel files, and whether names that repeat under more than one subject (for example "Data Interpretation") should be shared. Today they are separate categories per subject.
- Mobile APIs that serve questions (filter on the server, shuffle options, never send `is_correct`) are a later task.

## Test plan

**Automated tests (53 unit tests passing, integration not run):**
- [x] `test/unit/question-upload.spec.ts`: `slugify`, `buildSubjectTree`, taxonomy request schemas, question request schemas (three difficulty levels only, two to five options, fields the client must not set) and bulk schemas (25 rows per request, bad rows kept for per-row reporting).
- [x] `npm run typecheck`: no errors.
- [x] `npm run lint`: no errors.
- [x] `npm test`: 7 files, 53 tests passed.
- [ ] `test/integration/question-upload.int-spec.ts`: access, taxonomy, questions (including publish refused without exactly one correct option, and edit makes revision 2 and retires revision 1) and bulk (dry run, duplicates, 26 rows rejected). Not run, because it needs the migration and seed applied by the developer.

**Manual checks (local):**
- [ ] `migrate:up`, `seed:up`, `seed:down`, `seed:up` on a real database - not run, developer runs these.
- [ ] Admin panel against this backend - not checked, because the migration and seed have not been run yet.
