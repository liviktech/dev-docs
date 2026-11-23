# 2026-11-23 - question-image-upload (backend)

## Summary

Questions could not have images. The tables for images existed from migration `0001`, but no code used them. This PR adds an admin only `media` module that lets the browser upload an image straight to S3, checks the stored file, and links confirmed images to a question's stem, each option and the explanation. The admin question detail now returns those images.

The student endpoints (practice and mock exams) are **not changed**. They still return no images. That is Phase 6 and is described in `IMPLEMENTATION-AND-PHASE-6.md`.

Checked live: the upload reached a real S3 bucket from the admin page. The presign call returned 201 and the browser upload and the confirm call were seen working after the bucket CORS and IAM policy were fixed. The question save with images and the detail response were not checked against real S3. Everything else is covered by unit and integration tests only.

## What changed

| File | Change |
|---|---|
| `src/modules/media/` (new) | New module. `media.controller.ts` has `POST /admin/media/presign`, `POST /admin/media/:id/confirm` and `DELETE /admin/media/:id`, all admin only. `media.service.ts` builds the presigned POST, confirms the stored object and deletes unused images. Also a repository, DTOs, Zod request and response schemas, interfaces and `lib/image-info.ts`. |
| `src/modules/media/lib/image-info.ts` (new) | Reads type, width and height from the first bytes of a JPEG, PNG or WebP file. Returns null for anything else, so SVG and renamed files are rejected. |
| `src/database/migrations/0017_media-upload-status.sql` (new) | Adds `upload_status`, `file_hash` and `confirmed_at` to `media_assets`. |
| `src/config/env.schema.ts` | New optional variables `S3_BUCKET`, `AWS_REGION`, `CDN_BASE_URL`, `UPLOAD_MAX_BYTES`, `UPLOAD_PRESIGN_TTL_SECONDS`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`. Bucket, region and CDN URL go together. The two AWS keys go together. A blank value counts as not set. |
| `src/config/storage.config.ts` (new) | Returns the S3 settings, the optional credentials, the size limit and the presign expiry. |
| `src/app.module.ts` | Registers the media module and the storage config. |
| `src/modules/questions/schemas/question-request.schemas.ts` | `stemAssetIds` (max 5), `explanationAssetIds` (max 5) and `options[].assetIds` (max 2). |
| `src/modules/questions/schemas/question-response.schemas.ts`, `interfaces/question.interfaces.ts` | The detail returns `stemImages`, `explanationImages` and `options[].images`. List rows return an image count. |
| `src/modules/questions/repositories/questions.repository.ts` | Links assets when a revision is written. Reads images for the detail. Checks the assets exist, are confirmed and are not used by another question. |
| `src/modules/questions/questions.service.ts`, `bulk-questions.service.ts`, `questions.module.ts` | Pass image ids through create, update and bulk import. The module imports `MediaModule`. |
| `package.json`, `package-lock.json` | Adds `@aws-sdk/client-s3` and `@aws-sdk/s3-presigned-post`. |
| `.env.example` | Documents the new variables. |
| `architecture.md` | Section 10c and the media configuration are updated. |
| `test/unit/media.spec.ts` (new), `test/unit/question-upload.spec.ts`, `test/integration/question-upload.int-spec.ts` | New and extended tests. |

## Why

Admins need to attach diagrams and graphs to SAT and ACT questions, and the web and mobile apps must show them later. Image bytes cannot go through the API because the body limit is 100 kb, so the browser sends the file to S3 directly and the backend only signs and checks it.

## How

**Upload in three steps**: the admin calls presign with filename, type, size and alt text. The backend creates a `pending` asset with the key `questions/<year>/<month>/<uuid>.<ext>` and returns a presigned POST. The POST limits the size (`content-length-range`) and fixes the content type. The browser uploads to S3. The admin then calls confirm.

**Confirm reads the real file**: confirm reads the first 256 KB of the object, checks the magic bytes against the type the admin declared, records the true size, width and height, and marks the asset `confirmed`. A wrong or empty file is deleted from S3 and the row is removed. Calling confirm twice is safe.

**Keys only in the database**: the database stores the S3 key. `MediaService.publicUrl(key)` builds `CDN_BASE_URL/key` when a response is made. The key is built from a UUID and never from the uploaded filename.

**Question revisions**: an edit makes a new revision. The new revision links the images it lists, and the old revision keeps its own, because old attempts point to old revisions.

**AWS credentials**: if `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are set, they are passed to the S3 client. If not, the client uses the AWS default lookup (an IAM role in production). They have to be declared in the env schema because `ConfigModule` only copies declared keys into `process.env`. Before this was fixed, the keys in `.env` never reached the SDK and the presign call failed with `CredentialsProviderError`.

**Kept the same on purpose**: student endpoints, their response schemas and the existing `media_assets` and link tables. No new table was added.

## Decisions made

**Alt text is saved at presign time.** The single upload asks for the description before the file is chosen. It cannot be edited afterwards. Bulk uses the `image_alt` column or the file name.

**Image limits.** 5 images in the stem, 2 per option, 5 in the explanation, 5 MB each, JPEG, PNG or WebP only. SVG is rejected because it can carry scripts.

**No S3 resizing yet.** The plan has a Lambda that makes WebP and smaller sizes. It is not built, so the original file is served.

**Passage images are not wired.** The plan only listed the stem, option and explanation.

## Bugs found while testing (already there before, not fixed here)

**1. Four older tests fail.** Three in `test/integration/analytics.int-spec.ts` and one in `test/unit/otp.service.spec.ts` (the email text check). They are in analytics and the OTP email, not in code this PR touches. In this run the OTP test still fails.

## Notes

- Without the S3 settings the media routes answer 503, and the rest of the API starts as before.
- Backend lint could not run on this Windows machine because Application Control blocks `oxlint`.
- The `migrate:down` then `migrate:up` check was blocked by a stuck advisory lock on the shared dev database. The SQL was checked inside a rolled back transaction instead.
- Setup outside this repo is still needed: bucket CORS for the admin origin, an IAM policy with `s3:PutObject`, `s3:GetObject` and `s3:DeleteObject` on `questions/*`, a lifecycle rule for unconfirmed uploads, and a `CDN_BASE_URL` that actually serves the files. During local testing the first two were missing and showed up as a 403 on the browser preflight and an `AccessDenied` on the upload. The image URL also returned 403 because the bucket is private, so `CDN_BASE_URL` still needs CloudFront or public read on `questions/*`.
- Run `npm run migrate:up` on every environment before release.
- If an old `.env` has the AWS keys and `ConfigModule` is not given them, the presign call fails. Restart the backend after changing `.env`.

## Next steps

- Add images to the student practice and mock exam endpoints (Phase 6).
- A cleanup job for confirmed images that nothing uses. Unused images stay in S3 today.
- Allow alt text to be edited after upload.
- Wire passage images.
- Optional Lambda for WebP and thumbnail sizes.

## Test plan

**Automated tests (169 passing, 1 failing, the older OTP email test):**
- [x] `test/unit/media.spec.ts`: image type and size reading for PNG, JPEG and WebP, rejection of SVG, text and truncated files
- [x] `test/unit/media.spec.ts`: presign request schema, storage env rules, blank values, AWS keys kept and required together (15 passing)
- [x] `test/unit/question-upload.spec.ts`: image id limits per slot
- [x] `test/integration/question-upload.int-spec.ts`: create, edit and detail with images, and the asset rules. Last run before the credentials fix. Not re-run this session.
- [x] `npm run typecheck`: passes
- [ ] `npm run lint`: not run, blocked by Application Control on this machine
- [ ] `test/unit/otp.service.spec.ts`: still failing, not part of this work

**Manual checks (local):**
- [x] Add image in the admin question form against a real S3 bucket: presign returned 201, the upload and confirm succeeded once CORS and IAM were set, and the image appeared in the form
- [ ] The saved image displaying through `CDN_BASE_URL` - not working yet, the bucket is private and returned 403
- [ ] Bulk import with multiple images and with a ZIP against real S3 - not run
- [ ] `migrate:up`, `migrate:down`, `migrate:up` on the dev database - blocked by a stuck advisory lock
