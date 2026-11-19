# 2026-11-16 - enquiry-confirmation-date (frontend)

## Summary

Enquiries had no Date of Confirmation on screen. This PR adds it to the desktop and mobile (PWA) enquiry create and edit forms, and shows it on the desktop view dialog and the mobile detail screen. An enquiry with no date shows nothing for the field and saves `null`. This was type checked only. The screens were not opened in a browser for this change.

**The Follow-ups tab in the notification panel is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/enquiries/api.ts` | Adds optional `confirmationDate` to the `Enquiry` type and to the create and update input type. |
| `src/features/enquiries/enquiry-form-modal.tsx` | New Date of Confirmation date picker next to Delivery Date. It is filled from the saved enquiry on edit, cleared when the form resets, and sent as `confirmationDate` (empty becomes `null`). The earliest pickable date is the enquiry date. |
| `src/features/enquiries/enquiry-view-modal.tsx` | Shows Date of Confirmation in the details, formatted like Delivery Date. Hidden when empty. |
| `src/features/pwa/enquiry/mobile-enquiry-form.tsx` | New Date of confirmation field using the same date input as Delivery date. Prefilled on edit, sent as `confirmationDate` (empty becomes `null`). |
| `src/features/pwa/enquiry/mobile-enquiry-detail.tsx` | Shows Date of confirmation in the Enquiry card, formatted like the enquiry date. |

## Why

Sales needs to record when a customer is expected to confirm an enquiry. The backend now stores the date (see `PR-backend.md`), so the forms need to send it and the view screens need to show it.

## How

**Copied the delivery date pattern**: each form already handles delivery date (state, prefill on edit, reset, payload). The new field is wired the same way, using the same date picker on desktop and the same date input on mobile, so the look matches.

**Empty means null**: an empty date is sent as `null`. This lets a user remove a date when editing, and the backend then clears it.

**Date display**: the saved value comes back as a date string. The forms cut it at `T` to fill the picker, the same as delivery date, and the view screens format it with the existing `formatDate`.

## Decisions made

**The in-app follow-up panel was built and then taken out.** An earlier version added a Follow-ups tab to the mobile Notifications screen (Overdue, Due today and Upcoming, per enquiry). It was reverted together with its endpoint, so it is not in this diff. It will come back with the notification PR.

## Next steps

- Follow-ups tab in the notification panel, together with the backend notification work.

## Test plan

**Automated tests (no new tests added):**
- [ ] No test was added. The enquiry screens have no existing test file for this.
- [x] `npx tsc -p tsconfig.app.json --noEmit`: passes with no errors.

**Manual checks (local):**
- [ ] Create an enquiry with a Date of Confirmation on desktop and on mobile - not done in a browser in this session.
- [ ] Edit and change the date, then clear it, and confirm it saves as empty - not done.
- [ ] Open the view dialog and the mobile detail screen and confirm the date shows, and that it is hidden when empty - not done.
