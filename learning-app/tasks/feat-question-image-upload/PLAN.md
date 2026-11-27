# Plan: question image upload (AWS S3)

Branch: `feat-question-image-upload`. Repos: backend (`learning-app-backend`), admin (`learning-app-admin`). The web (`learning-web-app`) and mobile display come later.

## Goal

Let admins attach images to questions in the admin panel, store them in S3, and later show them in the web and mobile apps. Three ways to add images:

1. Single question: upload images directly in the question form.
2. Bulk: a CSV plus multiple selected image files.
3. Bulk: a CSV plus one ZIP of images (an optional alternative to option 2).

After any of these, the admin can click the question and see it in full (stem, all options, correct answer, explanation and every image) on a detail page.

A question can have several images in the stem, in each option, and in the explanation. Passage images use the existing `stimulus` usage.

## Status

Phases 1 to 5 and 3b are built, not committed, and checked with unit and integration tests only. Nothing has run against a real S3 bucket or in a browser yet. Phase 6 is not started.

What differs from the plan above:

- **Alt text:** the single upload asks for a short description before the file is chosen, because the backend saves alt text when the upload is presigned. It cannot be edited later. In bulk, alt text is the `image_alt` column, else the file name without its extension.
- **Option images in the migration:** `0017` only adds `upload_status`, `file_hash` and `confirmed_at`. Ordering and alt text already existed on the link tables and `media_assets`.
- **Image limits:** 5 images in the stem, 2 per option, 5 in the explanation, 5 MB each, JPEG, PNG or WebP.
- **Edit:** an edit sends the whole question again, so the image lists replace the old ones. The new revision links what it lists and the old revision keeps its own.
- **Bulk uploads happen at import time:** the dry run checks rows and image references first, and images are uploaded only for rows that passed. This avoids storage use by rejected rows.
- **Passage images:** not wired. The plan only listed the stem, option and explanation.
- **Lambda resizing, bucket CORS, IAM and the lifecycle rule:** still to be set up outside these repos.

## What the code already gives us

- `media_assets` already exists from migration `0001` (`storage_key`, `content_type`, `byte_size`, `width`, `height`, `alt_text`). The link tables `question_revision_assets` (usage `stimulus`, `question_stem`, `explanation`) and `question_option_assets` exist too. No code uses any of them yet.
- Editing a published question makes revision n+1 and retires the old one. Image links hang off a revision, so an edit must copy or replace the links, or the images get dropped.
- Bulk import is parsed in the browser and sent in chunks (up to 50 rows, body limit 100 kb). Image bytes cannot go through the API, so images upload straight to S3.
- Backend rules from `skills.md` and `architecture.md` apply: NestJS, raw `pg`, Zod, SQL migrations, controller then service then repository. `architecture.md` must be updated in the same change.

## Approach and decisions

- Reuse the existing tables. One new migration adds an upload status (`pending` or `attached`) to `media_assets`, an optional `file_hash`, and ordering and alt text handling on the link tables.
- Asset first flow. The admin uploads the image and gets an `assetId`. The question payload then lists `assetIds` per slot (stem, each option, explanation).
- Images are separate fields, not inline markdown. Question detail returns `stemImages[]`, `explanationImages[]` and `options[].images[]`, each as `{ id, url, width, height, altText }`. This keeps web and mobile rendering simple.
- Private bucket served through CloudFront. The database stores the S3 key only, never the full URL. The API builds the URL from `CDN_BASE_URL`.
- An S3 triggered Lambda converts to WebP and makes thumbnail and medium sizes. The backend has no queue or Redis, so resizing stays out of the NestJS app.
- Both bulk paths feed one pipeline (CSV filename to file map). The ZIP is unzipped in the browser, so there is no server side ZIP job.

## Phases

### Phase 1: storage foundation (backend)

- Add `@aws-sdk/client-s3`, `@aws-sdk/s3-presigned-post` and `@aws-sdk/s3-request-presigner`.
- Add env vars `S3_BUCKET`, `AWS_REGION`, `CDN_BASE_URL`, `UPLOAD_MAX_BYTES` in `src/config/env.schema.ts` and `.env.example`, with a new `storage.config.ts`.
- New `media` module behind `AdminRoleGuard`, with endpoints:
  - `POST /admin/media/presign` takes filename, content type, size and alt text. It returns a presigned POST that enforces content type and `content-length-range`, and creates a `pending` asset with key `questions/<yyyy>/<mm>/<uuid>.<ext>`.
  - `POST /admin/media/:id/confirm` does a `HEAD` on the object, checks the magic bytes, records size and dimensions, and marks it valid.
  - `DELETE /admin/media/:id`.
- Allowed types are jpg, png and webp. SVG is rejected.
- Migration with up and down, then run `migrate:up`, `migrate:down`, `migrate:up`.
- Infra notes: bucket CORS for the admin origin, IAM scoped to the `questions/` prefix, presigned URL expiry 5 minutes or less, lifecycle rule that deletes `pending` uploads after 24 hours.

### Phase 2: question API wiring (backend)

- Add `assetIds` to the create and update schemas (`stemAssetIds`, `explanationAssetIds`, `options[].assetIds`).
- `writeRevision` links the assets. `update` copies or replaces the links on the new revision.
- `findDetail` returns the image objects. List rows get an image count.
- Validate that assets exist, are confirmed, and are not attached to an unrelated question. Add image count limits per slot.
- Delete S3 objects only when no revision references the asset, because old revisions keep their images.
- Tests: extend `test/unit/question-upload.spec.ts` and `test/integration/question-upload.int-spec.ts`.
- Update `architecture.md` section 10c.

### Phase 3: single question upload (admin)

- A reusable `ImageUploader` component with preview, upload progress, reorder, remove and required alt text.
- Use it in `src/features/content/components/question-form.tsx` for the stem, each option and the explanation.
- Add `uploadImage()` to `src/features/content/api/content.api.ts` (presign, send to S3, confirm).
- Show existing images on the edit page and update `question-mapping.ts`.

### Phase 3b: question detail view (admin)

Today the admin panel only has a list and an edit form (`/questions/:id/edit`). There is no read only view. After a single or bulk upload the admin must be able to click a question and see it in full, so we add one.

- New read only page at `/questions/:id` (route added in `src/routes/routes.tsx`, new `question-detail-page.tsx` under `src/features/content/pages/`). It uses the existing `fetchQuestion(id)` call, which already returns the full detail after Phase 2 adds the image objects.
- It shows: subject and category, difficulty, calculator flag, status (draft, published, archived), the passage (with its images), the stem with its images, every option (key A to E, text and images) with the correct one clearly marked, and the explanation with its images.
- Images are shown at a readable size (medium), with a click to open the full size, and alt text as the image `alt`. A broken image shows a clear placeholder instead of an empty gap.
- Buttons on the page: Edit (goes to `/questions/:id/edit`), Archive or Restore, and Back to list.
- In the list (`questions-page.tsx`), make the stem text a link to the detail page, and show a small image count badge on rows that have images. Keep the existing Edit button.
- After a single create or edit is saved, go to the detail page instead of the list, so the admin sees the result with the images straight away.
- After a bulk import, each created row in the result table links to its detail page.
- The detail response is the admin one, so it includes `isCorrect`. Student endpoints are not touched.
- Tests: a page test in `content-pages.test.tsx` that renders a question with options and images and checks the correct option is marked, the images appear, and a question with no images shows no image area.

### Phase 4: bulk, CSV plus multiple images (admin)

- New CSV columns: `stem_images`, `option_a_image` to `option_e_image`, `explanation_images`, `image_alt`. Values are filenames separated by `|`. Update `lib/bulk-file.ts` and the downloadable template.
- Multi file picker in `pages/import-page.tsx`.
- Check before upload: missing images, unused images, duplicate names, case mismatches, wrong types, oversize files.
- Upload about 5 at a time with a retry per file, build the asset map, then send rows in the existing chunks with `assetIds` filled in.
- Show a result per row so a retry only redoes the failures.

### Phase 5: bulk, CSV plus ZIP (admin)

- ZIP picker. Unzip in the browser (`fflate` or JSZip) and feed the Phase 4 pipeline.
- Safety limits: max entry count, max total uncompressed size, skip `__MACOSX`, reject paths with `../`.

### Phase 6: display in web and mobile (later, out of scope for this branch)

- Show images in the practice runner, review and explanation views, using thumbnail and medium sizes, lazy loading, alt text and CDN cache headers.
- Student endpoints need no schema change beyond Phase 2.

## Validation and security

- Check magic bytes, not only the extension. Cap size per image, files per batch and total batch size.
- Object keys are UUIDs, never the original filename.
- Strip EXIF in the Lambda step.
- Admin only endpoints. The bucket is never public.

## Risks and unknowns

- Bucket, CloudFront, IAM and Lambda live outside these repos. Still to decide: does the bucket and CloudFront already exist, and should Terraform or CloudFormation be added to a repo.
- Deleting an image must check every revision, not only the current one, because attempts reference older revisions.
- Existing questions have no images, so there is no data migration.
- Confirmed images that end up unused (removed in the form, or uploaded and then the import is abandoned) stay in S3. The 24 hour lifecycle rule only covers unconfirmed uploads. A cleanup job for orphans is a follow-up.
- Verification gaps: lint cannot run on the backend on this Windows machine (Application Control blocks `oxlint`), and 4 older tests fail (3 in `analytics.int-spec.ts`, 1 email template test). They are not caused by this work.
- The `migrate:down` then `migrate:up` check on the shared dev database was blocked by a stuck advisory lock. The SQL was checked inside a rolled back transaction instead.
- Still to decide: which bulk path is the default in the admin panel. Suggested: ZIP for larger batches.

## Out of scope on purpose

- **Web and mobile display (Phase 6)**, because the user wants the admin side first.
- **Server side ZIP processing**, because the browser can unzip and reuse the same upload path.

## Test plan

- Backend: `npm run typecheck`, `npm run lint`, `npm test`, `npm run test:integration`, plus the migrate up, down, up check.
- Admin: `npm run typecheck`, `npm run lint`, `npm test`, with new cases for the CSV image columns and the ZIP limits.
- Live check against a real S3 bucket for single upload, bulk with multiple files, and bulk with ZIP. After each one, open the question from the list and confirm the detail page shows the stem, all options, the correct answer and every image.
