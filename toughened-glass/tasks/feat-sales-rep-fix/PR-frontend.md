# 2026-11-25 - direct-sales-rep-removal (frontend)

## Summary

"Direct" is no longer a row in the Sales Representatives master, so the screens that showed or protected it are cleaned up. Enquiry forms now default the rep to "Direct" (no rep) and no longer show "None / Unassigned". The Add Sales Representative dialog is always a dropdown of employees, and the "type a new name" option is removed. When the Enquiry Source is Visit, the rep must be a real person. The PR also stops the New Enquiry form from changing height when the Enquiry Date or Delivery Date picker opens.

The type check passes. The date picker height was measured in a headless browser against the running app. The Visit rule and the other screens were not opened by hand (see "Test plan").

## What changed

| File | Change |
|---|---|
| `src/features/enquiries/enquiry-form-modal.tsx` | The rep dropdown starts with "Direct" instead of "None / Unassigned". For a Visit source, Direct is removed from the list, the label gets a red star, the placeholder is "Select sales rep", and saving without a rep shows "Please select a sales rep for a Visit enquiry." The Enquiry Date and Delivery Date pickers are wrapped in a `flex` div (see "How"). The dialog class is now `h-[min(92vh,720px)]` (see "Decisions made"). |
| `src/features/pwa/enquiry/mobile-enquiry-form.tsx` | Same rule on mobile. "Direct" replaces "None / Unassigned", it is hidden for a Visit source, and `validate()` shows "Select a sales rep for a Visit enquiry." on the field. |
| `src/features/sales-reps/sales-rep-dialog.tsx` | The "Not in the list? Type a new name" link and the typing mode are removed. A new rep is always picked from the employee dropdown. |
| `src/features/sales-reps/api.ts`, `src/features/sales-reps/types.ts` | `isSystem` removed. |
| `src/routes/admin.sales-reps.tsx` | The status switch and the Delete action no longer have a special case for the Direct row. |
| `src/features/pwa/master/heavy-adapters.tsx` | The "built-in Direct sales rep cannot be deactivated" check is removed. |
| `src/features/dashboard/components/sales-rep-filter.tsx` | A "Direct" option (value `DIRECT`) is added at the end of the list. |
| `src/features/reports/enquiry-follow-up/enquiry-follow-up-report.tsx`, `src/features/pwa/reports/enquiry-follow-up-report.tsx` | A "Direct" option (value `DIRECT`) is added to the Sales Rep filter on desktop and mobile. |

## Why

The backend no longer has a Direct row, so the screens cannot expect one in the rep list. Reports and dashboards still need a way to filter by "no rep", so they add Direct themselves. A visit is made by a person, so a Visit enquiry with no rep made no sense.

## How

**Direct in forms**: Direct is the empty value. The enquiry form shows "Direct" when no rep is chosen and sends an empty value for it. The mobile form does the same.

**Visit rule**: The form checks the source. For Visit it removes Direct from the options and blocks saving without a rep. The backend enforces the same rule, so this is only the friendly version.

**Direct in filters**: Report and dashboard rep filters add a "Direct" option with the value `DIRECT`, which the backend turns into "no rep".

**Date picker height**: The Base UI popover adds two hidden focus helper elements next to its trigger when it opens. The form's spacing rule (`space-y-*`) adds a bottom margin to the trigger when it is no longer the last child, so the field grew about 5px. The Enquiry Date and Delivery Date fields share a row with each other, so nothing absorbed it and the form grew. Date of Confirmation shares a row with a taller dropdown, which hid it. Putting the picker in its own `flex` div means the helpers land inside the div, so the field does not change.

**Left the same on purpose**: The Sales Visit screens and the party dialog never offered Direct, and they were not touched.

## Decisions made

**The fixed dialog height was left in.** While looking for the cause of the height change, the dialog class was changed to a fixed height and the open animation was removed. That did not fix it. The real fix is the wrapper above. A request to put the original class back was declined, so `h-[min(92vh,720px)]` is still there. It leaves empty space at the bottom of the form on tall screens. The original class was `transition-all duration-300 data-open:duration-500 data-open:ease-out data-open:animate-in data-open:fade-in-0 data-open:zoom-in-95 data-closed:duration-200`.

**The Visit check is on the form, not just the server.** The form hides Direct and also checks on save, so the user sees the problem before sending anything.

## Notes

- The date picker was measured with headless Chrome on the running app at a 600px high window. The form's content height stayed at 451px before and after opening either picker. Before the wrapper it went from 451px to 456px.
- The test script and its temporary browser profile were deleted afterwards.

## Next steps

- Decide whether to put the original dialog class back in `enquiry-form-modal.tsx` now that the wrapper fixes the height change.
- Open the Sales Representatives page, both enquiry forms and the rep filters in a browser, on desktop and mobile.

## Test plan

**Automated tests (not added):**
- [x] `npx tsc --noEmit`: passes, no errors.
- [ ] No component tests were added for the Visit rule or the rep dropdowns.

**Manual checks (local):**
- [x] New Enquiry form, Enquiry Date and Delivery Date: opened each picker in headless Chrome and measured the form content height. It stayed at 451px (429px in the second measurement) before and after opening. Before the fix it grew 451px to 456px.
- [x] Date of Confirmation: it did not change height before or after the fix.
- [ ] The Visit rule on the desktop form, the mobile form and the Add Sales Representative dropdown were not opened by hand, so nothing was seen on screen.
- [ ] The Direct option in the dashboard and report filters was not tried, since the backend migration was not run.
