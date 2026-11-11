# 2026-10-08 - photo-rotate-four-sides (frontend)

> **Linked PR (backend):** https://github.com/liviktech/rubine-toughened-glass-backend/pull/65

## Summary

Photos were uploaded as they came from the phone and sent straight to the AI, so a sideways sheet was read sideways. In the Read from photo window the photo pane could cut off the bottom of the photo and would not scroll when zoomed, and the results table only had one width and one height. This PR adds a rotate step before upload (the photo is saved in the orientation staff choose and keeps it for good), shows all four sides of every piece (top width, bottom width, left length, right length), fixes the photo pane, and turns the photo list into a one row "Recents" with a "See more" window.

Checked live in a browser on the local app (including a fresh Gemini read): rotate preview, upload of a rotated photo, the four side table, Add Product form values, Recents and See more, and the photo pane on a tall photo at 1024x768, 1280x600, 1440x900 and a 375 px phone.

**Rotating photos that were uploaded before this change is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

**Debug logging in `src/lib/api-client.ts` and a host change in `vite.config.ts` that were already uncommitted are not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/ai-extraction/shrink-photo.ts` | Photo drawing split into one function that turns the photo (0, 90, 180 or 270 degrees) and saves it as JPEG. New `previewPhoto()` (1200 px copy for the rotate step), `turnPhoto()`, and a `PhotoRotation` type. `shrinkPhoto(file, rotation)` now bakes the turn into the pixels before the 2000 px shrink. |
| `src/features/ai-extraction/photo-review-panel.tsx` | New rotate step between choosing a photo and uploading it (Rotate left, Rotate right, Choose another, Use this photo). Four size columns (Top W, Bottom W, Left L, Right L) with the original inch value under each one on inch sheets. The photo list on an order booking is now "Recents": one row of 3 photos with a "See more" button at the right end of the title row that opens a "Photos for this order" window. Photo pane layout fixes (below). |
| `src/features/ai-extraction/types.ts` | `ReadingRow` has `topWidthMm`, `bottomWidthMm`, `leftLengthMm`, `rightLengthMm` and an optional `originalSides`. |
| `src/features/order-booking/add-product-modal.tsx` | `sizeFromReadingRow` fills Top Width, Bottom Width, Left Length and Right Length from the four sides. Actual Width is the larger of top and bottom, Actual Height the larger of left and right. |
| `src/features/order-booking/draft-products.ts` | Two rows count as the same piece when all four sides, quantity and code match (was width and height). |
| `src/lib/api-client.ts`, `vite.config.ts` | no real change from this PR (already uncommitted debug logging and an allowed host, not touched) |

## Why

The AI is now told to treat the photo exactly as it receives it, so the person who knows which way up the sheet is has to set that before the photo is saved. Doing it in the browser, before upload, means the stored file, the fingerprint used for reuse, the AI and every later viewer all see one orientation. The four sides match the fields the Add Product form already has. The pane fixes came from the Read from photo window being hard to use on some screens.

## How

**Rotate before upload**: choosing or taking a photo opens a preview. Rotate left and right redraw a smaller copy so the preview is exactly what will be saved. Nothing is uploaded until Use this photo. The same step applies on the Enquiry screen, because those photos are read later on the order booking.

**The turn is baked into the file**: the canvas is rotated and saved as JPEG, so no orientation value has to be stored anywhere. The same sheet uploaded turned a different way is a different image and is read again, which is wanted.

**Four sides in the table**: one set of columns generated from a list, each with the "unsure" highlight for its own field name. On inch sheets each field shows its own original value ("36.25 in") taken from the saved `originalSides`, not converted back from mm.

**Photo pane**: on wide screens the photo and the table share one fixed height (70% of the window) and one row that may shrink to it, so a long table can no longer stretch the photo pane past the window. The photo area scrolls both ways when zoomed. The table has a minimum width and scrolls sideways inside its own box instead of squeezing the columns. On small screens the grid is capped to the screen width.

**Enquiry preview**: the zoomed photo is centred with `m-auto` on the image instead of centring the box, so it can be scrolled back to its top and left edges.

**Kept the same on purpose**: nothing is written to the order from the dialog, drafts still go through the Add Product form, the Enquiry window keeps its full photo list, and `api-client.ts` still sends only JSON (photos go as base64).

## Decisions made

**The turn is drawn into the image, not saved as a setting.** A saved angle would need a new column and every viewer would have to apply it. Drawing it once means everything downstream just works. The cost is that a turned copy has a different fingerprint than the original.

**Photos already uploaded cannot be turned.** A stored photo is fixed bytes, and turning it would create a new image. Old photos open as before. If an old enquiry photo is sideways it has to be uploaded again.

**Recents always shows "See more".** It is shown even when the order has 3 photos or fewer, so the window behaves the same every time.

**Recents is on the Order Booking window only.** The Enquiry window keeps its full list and its "Photos on this enquiry" title.

**Table scrolls sideways instead of hiding columns.** All four sides, quantity, thickness and the hole counts are needed for the review, so the table keeps a minimum width of 820 px.

## Notes

- The frontend `.env` points `VITE_API_URL` at an ngrok address that no longer answers. For the live test the address was overridden on the command line (`VITE_API_URL=http://localhost:5100/api/v1`), and the file was not changed.
- Photos in Recents show a broken thumbnail when the image file is not in the local `uploads/` folder (seen on an enquiry photo copied from the shared database). This is a local data gap, not a code problem.
- Already uncommitted in this repo before this task and **left out of this PR**: debug `console.log` lines in `src/lib/api-client.ts` and an extra allowed host in `vite.config.ts`.
- The inch text under each field was only type checked and linted. It was not seen in the browser, because the photos used for the live test were not inch sheets and the stored readings have no `originalSides`.

## Bugs found while testing (already there before, not fixed here)

**1. `npx tsc -b` reports errors in files this PR does not touch.** For example `booking-header-form.tsx` (`isActive` and string type errors) and the routes `admin.company.tsx`, `admin.sales-reps.tsx` and `final-invoices.tsx`. None are in the files changed here.

**2. ESLint reports errors in `add-product-modal.tsx`.** For example "setState synchronously within an effect" and unexpected `any`. They are in code this PR does not change (the only edit there is inside `sizeFromReadingRow`).

## Next steps

- Check the inch text under each field with a real inch sheet read after this change.
- Add a "turn and save as a new photo" button for photos uploaded before this change, if staff need it.
- Fix the two kinds of existing type and lint errors above in their own change.

## Test plan

**Automated tests (no automated test suite exists in this repo):**
- [x] `npx tsc -b`: no errors in `features/ai-extraction`, `order-booking/draft-products.ts` or `products-tab.tsx`. Errors remain in other files (see "Bugs found while testing").
- [x] `npx eslint src/features/ai-extraction src/features/order-booking/draft-products.ts`: clean.

**Manual checks (local, Vite dev server, backend on a migrated copy of the database):**
- [x] Rotate step: chose `1.jpg`, the preview showed. Rotate right changed the preview from 1200x675 to 675x1200, Rotate left changed it back. Use this photo uploaded and read it.
- [x] Rotated upload: `12.jpeg` (720x1280) turned right and uploaded. The saved file on disk is 1280x720, so the turn is in the stored image.
- [x] Four side table: a fresh read of `10.jpg` showed Top W, Bottom W, Left L and Right L filled for all 8 rows. The rotated `12.jpeg` read showed top 2762 and bottom 2742 as separate values.
- [x] Add Product form: a draft opened with Top Width 1895, Bottom Width 1895, Left Length 1971, Right Length 1971 and area 3.735.
- [x] Older readings: photos read before the change opened with top equal to bottom and left equal to right.
- [x] Recents and See more: one row of the 3 photos, "See more" at the right of the title row, the window listed all 3 (including the one from the enquiry), and picking `1.jpg` opened it and closed the list.
- [x] Photo pane, tall photo `10.jpg` (900x1600): the whole photo fits in its pane at 1024x768, 1280x600 and 1440x900. Zoomed, the pane scrolled to the very bottom and right edge.
- [x] Phone (375 px wide): after a fix the photo shows in full and the page does not scroll sideways. The first version of the table minimum width pushed the photo off screen on a phone, found and fixed during this test.
- [ ] Inch text under each field - not seen live (see "Notes"). Covered by the type check only.
- [ ] Rotate step on the Enquiry screen - not run in the browser. It uses the same panel code and the same step, and only the Order Booking window was driven.
- [ ] Camera ("Take a photo") with the rotate step - not run, needs a phone camera.
