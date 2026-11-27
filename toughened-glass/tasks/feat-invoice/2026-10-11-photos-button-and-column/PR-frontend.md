# 2026-10-11 - photos-button-and-column (frontend)

## Summary

Only the Enquiry screen had a proper photo preview. Order Booking, PI Status and Work Orders had no photo column (PI Status had an "Attachments" column that only listed file names), and a draft order's Products tab used a separate popup that stacked every photo in one long list. This PR adds one shared "Photos" button and one shared table column, and uses them on Enquiry, Order Booking, PI Status, Work Orders and the Products tab of an order. All of them open the same Enquiry style preview window (photo list on the left, opened photo with zoom on the right).

It also fixes the endless loader. When a photo's file is missing from storage, the preview used to spin forever. It now shows "This photo could not be loaded" and the other photos in the list still open.

I ran the TypeScript check and lint on the new files, and checked the screens in the browser. See "Test plan".

**Uploading and AI reading from the preview are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/ai-extraction/photos-button.tsx` (new) | The one "Photos" button for every table. Takes `enquiryId` or `orderBookingId`, plus `readOnly`. Opens the shared photo window. Shows a dash if it has neither id. |
| `src/features/ai-extraction/photos-column.tsx` (new) | `photosColumn(...)`, one shared table column (header "Photos", centered, stops row clicks). Each table adds it in one line and only says which record a row belongs to. |
| `src/features/ai-extraction/photo-review-panel.tsx` | New `view` mode that uses the Enquiry gallery layout with no upload, camera or reading controls. The gallery opens the first photo by itself. A photo that cannot be downloaded now shows a "could not be loaded" message instead of loading forever. |
| `src/features/ai-extraction/enquiry-photos-button.tsx` | Removed, replaced by `photos-button.tsx`. |
| `src/features/enquiries/enquiries-page.tsx` | The Photos column now comes from `photosColumn`. Same behaviour as before (locked when the enquiry is not OPEN). |
| `src/features/order-booking/header-detail-tab.tsx` | New Photos column in the Order Booking list, before Actions. |
| `src/features/order-booking/products-tab.tsx` | Saved product rows use `PhotosButton` for both draft and confirmed orders. Column header changed from "Read from photo" to "Photos". The photo list query used only by the old popup was removed. |
| `src/features/order-booking/booking-photo-cell.tsx` | Removed, the old stacked photo popup is no longer used. |
| `src/features/pi-status/components/pi-status-table.tsx` | The "Attachments" column was replaced by the Photos column. The unused `Paperclip` import was removed. |
| `src/routes/work-orders.tsx` | New Photos column in the Work Orders table, before Actions. |

## Why

Staff need to see the handwritten dimension sheet from every order screen, not only from Enquiry. The PI Status "Attachments" column was built from the same photos but only showed file names, so it was a weaker copy of the Photos column and was removed. The loader fix came from old photos whose files were lost before the storage fix, which made the preview spin forever. The easy part was reusing the existing preview window. The real work was making one button and one column so the five places cannot drift apart.

## How

**One button**: `PhotosButton` opens `PhotoReviewPanel`. With an enquiry id it opens the Enquiry photos as before. With an order booking id it opens the new `view` mode, which lists the booking's own photos plus the ones carried over from its enquiry.

**One column**: `photosColumn<Row>((row) => ({ ... }))` returns the whole column. Enquiry passes `enquiryId` and `readOnly`. Order Booking, PI Status and Work Orders pass `orderBookingId`.

**Same layout everywhere**: the panel treats the Enquiry screen and the new `view` mode as one "gallery" layout, so the list, zoom and auto-open work the same in both.

**Missing files**: if the file download fails, the right side shows a message and an icon. The photo list stays usable, so staff can open the next photo.

**Kept the same on purpose**: the "Read from photo" button at the top of the Products tab, the upload and AI reading flow, and the `extract` mode of the panel were not touched.

## Decisions made

**The preview is view only on order screens.** Photos are uploaded on Enquiry or through "Read from photo" on a draft order. The tables only show them, so there are no upload or reading controls in the preview.

**Draft orders use the same button as confirmed orders.** At first only confirmed (locked) orders used it. It now covers saved rows on draft orders too, so the old stacked popup could be deleted instead of living next to the new one.

**No photo count on the button.** It says "Photos" without a number, the same as Enquiry. A booking with no photos now opens a "No photos" message, where the old popup showed a dash. Showing counts would need the backend to send a count with each list.

**Attachments column removed from PI Status, backend untouched.** The `attachments` fields on the PI data are still sent, in case other screens use them.

**The "could not be loaded" message also applies on the Enquiry screen**, since it shares the same layout.

## Bugs found while testing (already there before, not fixed here)

**1. Lint errors in files this PR touches.** `pi-status-table.tsx`, `header-detail-tab.tsx` and `work-orders.tsx` already report `any` types, a `setState` in an effect and a few hook dependency warnings. None come from the new code. Anyone running lint on the whole project will see them.

**2. Old photos are still broken.** Photos uploaded before the storage fix have no file behind them. They now show the message instead of an endless loader, but they still need to be re-uploaded or recovered.

## Notes

- A fully empty "Photos" cell (a PI or work order with no order booking id) shows a dash.
- Only the frontend changed. No backend change was needed.

## Next steps

- Ask the backend to send a photo count with each list, so the button can show "Photos (3)".
- Fix the lint errors listed above in a separate cleanup.
- Find and re-upload or recover the photos that were lost before the storage fix.

## Test plan

**Automated tests (no test files for this change):**
- [x] TypeScript check (`tsc --noEmit -p tsconfig.app.json`): no errors after the last change.
- [x] Lint on `photos-button.tsx` and `photos-column.tsx`: clean.
- [ ] Lint on the other touched files: only the old errors listed above, nothing from this change.

**Manual checks (local):**
- [x] Photos column on Enquiry, Order Booking, PI Status and Work Orders opens the preview.
- [x] A draft order and a confirmed order, Photos on a saved product row opens the same preview.
- [x] An old photo with a missing file shows "This photo could not be loaded" and not a loader.
- [x] A booking with no photos opens a "No photos" message.
