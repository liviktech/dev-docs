# 2026-11-23 - question-image-upload (admin)

## Summary

The admin panel could not attach images to questions, and had no read only page to look at a question in full. This PR adds image upload to the question form, two bulk image paths (CSV plus many image files, and CSV plus a ZIP), and a question detail page at `/questions/:id`.

Checked live: unit tests pass (89 of 89 across 12 files) and typecheck shows no errors. The single image upload was tried in the browser against a real S3 bucket and reached the upload and confirm steps. The bulk paths, the ZIP path and the detail page were not tried in a browser or against real S3.

Student web and mobile display is not part of this admin PR. It is in the backend and web PRs.

## What changed

| File | Change |
|---|---|
| `src/features/content/components/image-uploader.tsx` (new) | Reusable uploader. Needs a short description (alt text) first, then a file. Shows progress, thumbnails, move up and down and remove. Used for the stem, each option and the explanation. |
| `src/features/content/components/image-uploader.test.tsx` (new) | Tests for the uploader. |
| `src/features/content/lib/image-file.ts` (new) | Checks type and size of a chosen image in the browser. |
| `src/features/content/api/content.api.ts` | New `uploadImage()` that calls presign, posts the file to S3, then calls confirm. `postToStorage` sends the form fields in the order S3 needs. Question types carry image ids and image objects. |
| `src/features/content/components/question-form.tsx` | Image uploaders for the stem, each option and the explanation. |
| `src/components/common/options-editor.tsx` | Shows an uploader under each option. |
| `src/features/content/lib/question-mapping.ts`, `schemas/content.schemas.ts` | Map images between the form, the API and the edit page. |
| `src/features/content/pages/question-form-page.tsx` | Loads existing images on edit. After a save it goes to the detail page. |
| `src/features/content/pages/question-detail-page.tsx` (new) | Read only page with subject, difficulty, status, the stem, every option with the correct one marked, the explanation and all images. Click an image to open it larger. A broken image shows a placeholder. Buttons: Edit, Archive or Restore, Back. |
| `src/features/content/pages/question-detail-page.test.tsx` (new) | Detail page tests. |
| `src/features/content/pages/questions-page.tsx` | The stem is a link to the detail page and rows with images show an image count. |
| `src/routes/routes.tsx` | Adds `/questions/:id`. |
| `src/features/content/lib/bulk-file.ts`, `bulk-file.test.ts` | New CSV columns `stem_images`, `option_a_image` to `option_e_image`, `explanation_images` and `image_alt`. Several names are separated by `|`. The template includes the new columns. |
| `src/features/content/lib/bulk-images.ts`, `bulk-images.test.ts` (new) | Checks before upload: missing, unused, duplicate, wrong-case, wrong-type and too-big files. Uploads 5 at a time with 3 tries each. |
| `src/features/content/lib/zip-images.ts`, `zip-images.test.ts` (new) | Unzips in the browser with `fflate`. Limits: ZIP 100 MB, 1000 entries, 250 MB unzipped. Skips `__MACOSX` and rejects unsafe paths and duplicate names. |
| `src/features/content/pages/import-page.tsx` | Multi file picker and ZIP picker. Uploads images only for rows that passed the check, then sends the rows with image ids. Each created row links to its detail page. |
| `src/features/content/components/question-form.tsx` (second change) | The Publish button was a submit button whose handler published only when the question was already published, so Publish on a new or draft question saved a draft. It is now a normal button that publishes. Pressing Enter in a text box no longer saves anything. |
| `src/features/content/pages/import-page.tsx` (second change) | Fast import ran when the CSV was chosen, before any images were chosen, so every image row failed with "image was not selected" and adding the images later did not check again. Fast import now runs only when no image is missing. Otherwise the file is checked first and checked again when images or the ZIP are added. |
| `src/components/common/file-input.tsx` | Supports picking several files. |
| `package.json`, `package-lock.json` | Adds `fflate`. |

## Why

Admins need to add diagrams and graphs to SAT and ACT questions one by one or in large batches, and then check the result. The detail page lets them see a question exactly as saved, with its images, after a single save or a bulk import.

## How

**Single upload**: the admin types a description, picks a file, and `uploadImage()` runs presign, then a direct POST to S3, then confirm. The form keeps the returned image ids and sends them with the question. Editing a question sends the full image lists again, so they replace the old ones.

**Bulk with files**: the CSV names the images, for example `stem_images` = `a.png|b.png`. The admin picks the image files. The page checks every reference before anything is uploaded. Rows that pass the dry run get their images uploaded, then the rows are sent in the existing chunks with the image ids filled in.

**Bulk with ZIP**: the ZIP is unzipped in the browser and the files go through the same path as above. There is no server side ZIP step.

**Upload at import time**: images are uploaded only for rows that passed the check, so rejected rows do not leave files in S3.

**Kept the same on purpose**: the chunk size and byte budget for bulk rows, the dry run then import flow, and the existing Edit button on the list.

## Decisions made

**Alt text is required before the file is chosen.** The backend saves it when the upload is presigned. The Add image button stays disabled until the description box has text. It cannot be edited after upload.

**ZIP is unzipped in the browser, not on the server.** This reuses the same upload path and avoids a server job. The cost is the 100 MB and 250 MB limits.

**Passage images are not wired.** Only the stem, options and explanation have uploaders.

## Notes

- This needs the backend change in `PR-backend.md`. Deploy the backend first.
- The upload from the browser needs the S3 bucket to allow the admin origin in its CORS rules. Without it the browser preflight gets 403 and the form shows "Something went wrong".
- Thumbnails and the detail page images stay broken until `CDN_BASE_URL` points to something that serves the files.
- The files `exam-tabs.tsx`, `page-header.tsx`, `section-tab.tsx`, `section-tabs.tsx`, `dashboard-layout.tsx` and some analytics and exam pages show a newer modified time in this folder. They are not part of this work and are not listed above.
- Lint was not run in this session.

## Next steps

- Make alt text editable after upload.
- Add uploaders for passage images.
- Decide which bulk path is the default in the admin panel. ZIP is suggested for larger batches.

## Test plan

**Automated tests (89 passing):**
- [x] `src/features/content/components/image-uploader.test.tsx`: uploader behavior
- [x] `src/features/content/lib/bulk-images.test.ts`: image reference checks and upload retries
- [x] `src/features/content/lib/zip-images.test.ts`: ZIP limits and unsafe paths
- [x] `src/features/content/lib/bulk-file.test.ts`: the new image columns
- [x] `src/features/content/pages/question-detail-page.test.tsx`: the correct option is marked, images appear, and a question with no images shows no image area
- [x] `npx tsc --noEmit -p tsconfig.app.json`: no errors
- [ ] `npm run lint`: not run in this session

No test was added for the Publish button fix or the import re-check fix.

**Manual checks (local):**
- [ ] Publish on a new question saves it as published - not run in a browser, fixed by reading the code
- [ ] CSV plus ZIP import no longer reports "image was not selected" - not run in a browser
- [x] Add image on a question in the browser, against a real S3 bucket: the upload and confirm calls succeeded and the image row appeared in the form
- [ ] The thumbnail loading after the upload - not working yet, the bucket is private and returned 403 on the image URL
- [ ] Bulk import with several images and with a ZIP - not run
- [ ] The detail page and the list image badge in the browser - not run
