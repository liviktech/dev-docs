# Question images: what is built and how to show them in web and mobile (Phase 6)S

## 1. Summary

- **Admin panel and backend: done**
- **Student web app and mobile app: not done.** This is Phase 6, described in section 5.
- **Important:** the student endpoints (practice and mock exams) do **not** return images yet. The backend must change first (section 4).

## 2. What is already built

### Backend (`learning-app-backend`)

- New `media` module. All routes are admin only.
  - `POST /admin/media/presign` gives a presigned S3 POST. The browser sends the file straight to S3.
  - `POST /admin/media/:id/confirm` checks the real file (type, size, width, height) and marks it confirmed.
  - `DELETE /admin/media/:id` removes an image that is not used.
  - Without S3 settings these routes answer `503`.
- Migration `0017` adds `upload_status`, `file_hash` and `confirmed_at` to `media_assets`.
- The question create and update APIs accept image ids:
  - `stemAssetIds` (max 5), `explanationAssetIds` (max 5), and `options[].assetIds` (max 2 each).
- The admin question detail API returns the images.
- Rules the backend enforces:
  - Only JPEG, PNG and WebP. No SVG. Max 5 MB each.
  - An image must be confirmed.
  - An image used by one question cannot be used by another.
  - An edit makes a new revision. The new revision links the images it lists. The old revision keeps its own images.
- The database stores only the S3 key (for example `questions/2026/11/<uuid>.jpg`). The URL is built at read time as `CDN_BASE_URL + "/" + key` by `MediaService.publicUrl(key)`.
- New env vars: `S3_BUCKET`, `AWS_REGION`, `CDN_BASE_URL`, `UPLOAD_MAX_BYTES`, `UPLOAD_PRESIGN_TTL_SECONDS`. See `.env.example`.

### Admin panel (`learning-app-admin`)

- **Single question:** the question form has an image uploader for the stem, for each option and for the explanation. Each image needs a short description (alt text). Images have progress, reorder and remove.
- **Bulk, CSV plus images:** new CSV columns `stem_images`, `option_a_image` to `option_e_image`, `explanation_images`, `image_alt`. Several file names are separated by `|`. The admin selects many image files. The page checks missing, unused, duplicate, wrong-case, wrong-type and too-big files before anything is uploaded.
- **Bulk, CSV plus ZIP:** the ZIP is unzipped in the browser, then follows the same path. Limits: ZIP 100 MB, 1000 entries, 250 MB unzipped. Unsafe paths and duplicate file names are rejected.
- Images upload at import time, 5 at a time, 3 tries each, and only for rows that passed the dry run.
- **Question detail page** at `/questions/:id`. It shows the stem, options (correct one marked), explanation and all images. The list links to it, and it opens after a save or an import.

### Known gaps

- Confirmed images that are never used stay in S3. A cleanup job is a follow-up.
- Passage (stimulus) images are not wired in the admin.
- Alt text cannot be edited after upload.
- Bucket, CloudFront, CORS, IAM and the Lambda resize step are outside the repos and still to do.

## 3. The image object (the contract)

This is the shape used in the admin detail response. Use the same shape for students.

```json
{
  "id": "uuid",
  "url": "https://cdn.example.com/questions/2026/11/abc.jpg",
  "width": 800,
  "height": 600,
  "altText": "Graph of y = 2x + 1"
}
```

Where images appear in a question:

| Place | Field |
| --- | --- |
| Question text | `stemImages[]` |
| Each option | `options[].images[]` |
| Explanation | `explanationImages[]` |

Images are separate fields. They are **not** written inside the markdown text. Show them in the array order.

Note: `width` and `height` can be null. The admin response also has `imageCount` on list rows.

## 4. Backend work needed first

The student endpoints build their own queries and know nothing about images. Today they return `stem`, `passage` and option text only.

1. Find every student route that returns question content. Start with:
   - `src/modules/practice/` (the query that selects `stem_markdown` and `question_options`)
   - `src/modules/mock-exams/`
   - check `passages/` and anything that shows a question after an attempt (review).
2. For each one, add `stemImages`, `explanationImages` and `options[].images`, using the same image object as section 3.
3. Reuse the code that already exists in `questions.repository.ts`:
   - the `IMAGE_JSON` SQL snippet
   - the joins on `question_revision_assets` (usage `question_stem` and `explanation`) and `question_option_assets`
   - `MediaService.publicUrl(key)` to turn the key into a URL. Import `MediaModule` where needed.
4. Return images from the **revision the student actually answered or will answer**, not always the newest. Old attempts point to old revisions.
5. Do not send the explanation images before the student is allowed to see the explanation. Follow the same rule as the explanation text.
6. Update the Zod response schemas, the interfaces, `architecture.md`, and the tests. Run `npm run typecheck && npm test && npm run test:integration`.
7. Never return `storageKey` to students. Return `url` only.

Keep the change additive: new fields with an empty array when there are no images, so old clients keep working.

## 5. Web app (`learning-web-app`, Next.js)

1. Add the new fields to the TypeScript types and Zod schemas that parse practice and exam responses.
2. Make one small shared component, for example `QuestionImages`, that takes `images[]` and renders them. Use it in:
   - the question view (stem images, under the stem text)
   - each option (images under or beside the option text)
   - the explanation and review view
3. Rendering rules:
   - Use `alt={altText}`. If the alt text is empty, use `alt=""` only for decoration. Question images are content, so alt text should exist.
   - Set `width` and `height` when known, so the page does not jump while loading.
   - Lazy load images that are not at the top of the page.
   - Keep images inside the layout. Use `max-width: 100%` and keep the aspect ratio.
   - Show a plain placeholder when an image fails to load, so the question is still usable.
   - Click or tap to see the image larger (same idea as the admin detail page).
4. If you use `next/image`, add the CDN host to `images.remotePatterns` in `next.config.ts`. A plain `<img>` also works.
5. Options with images must stay selectable. The whole option row should be the click target, not only the text.
6. Check keyboard use and a screen reader on the option list.
7. Tests: render a question with images in the stem, an option and the explanation, and one without images. Check that no empty image area appears when there are no images.

## 6. Mobile app

The mobile app code is not in this workspace. Use the same API fields and the same rules as the web app.

1. Update the API types for practice and exam responses.
2. Build one shared image component:
   - show a placeholder or skeleton while loading
   - cache images on the device (use the image library's disk cache)
   - set the size from `width` and `height` so the layout does not jump
   - tap to open a full screen viewer with pinch to zoom
   - set the accessibility label to `altText`
3. Use it for stem, options and explanation.
4. Handle slow or no network: show the placeholder, never block the question.
5. Option rows with images need a large tap target.
6. Test on a small phone and a tablet, in portrait and landscape.

## 7. Image sizes (optional, later)

The plan includes a Lambda that makes WebP, a thumbnail and a medium size. Until it exists, the original file is served. When it exists:

- add the sizes to the image object (for example `thumbUrl` and `mediumUrl`)
- web and mobile pick the size that fits the screen
- keep `url` as the full size

Do not change the contract in section 3 without updating this file and the admin app.

## 8. Order of work

1. Set up S3, CloudFront, CORS and IAM (infra). Do a live check of the admin upload.
2. Backend: add images to student endpoints (section 4).
3. Web app (section 5).
4. Mobile app (section 6).
5. Optional: resize Lambda and image sizes (section 7).

## 9. Checklist for any agent

- Do not change the admin upload flow. It is finished.
- Do not store full URLs in the database. Store the key only.
- Do not send `storageKey` or `isCorrect` to students before they should see them.
- Keep changes additive. Old app versions must keep working.
- Keep `architecture.md` in the backend up to date in the same change.
- Run typecheck, lint and tests in each repo you touch. (On this Windows machine the backend lint is blocked by Application Control, so say so in your report.)
- Four older tests fail and are not related to this work: 3 in `analytics.int-spec.ts` and 1 in `otp.service.spec.ts`.

## 10. Where to look in the code

- Backend: `src/modules/media/`, `src/modules/questions/repositories/questions.repository.ts` (`IMAGE_JSON`, `findDetail`), `src/modules/questions/questions.service.ts` (`detailOrThrow`), `src/config/storage.config.ts`.
- Admin: `src/features/content/` (`api/content.api.ts` for `uploadImage`, `components/image-uploader.tsx`, `pages/question-detail-page.tsx`, `pages/import-page.tsx`, `lib/bulk-images.ts`, `lib/zip-images.ts`).
- Student queries to change: `src/modules/practice/repositories/practice.repository.ts` and `src/modules/mock-exams/repositories/mock-exams.repository.ts`.
