# 2026-10-21 - pi-close-reopen-auto-close (frontend)

## Summary

The PI Status page had no way to close a PI or bring it back. This PR adds a Closed tab, a Closed PI's KPI card, a Close PI and a Reopen PI action in the row menu, and a badge that says Closed or Auto closed. It works with the new close and reopen endpoints in the backend PR.

Not checked live yet: only the TypeScript build was run, the page has not been opened in a browser.

**The "auto closes on <date>" hint and a reason box in the Close dialog are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/pi-status/types.ts` | `CLOSED` added to the status types. New fields for the close reason, close date and whether the system closed it. |
| `src/features/pi-status/api.ts` | Maps the closed fields from the API. New `closeProformaInvoice` and `reopenProformaInvoice` calls. |
| `src/features/pi-status/queries.ts` | New `useClosePIMutation` and `useReopenPIMutation`. Both refresh the PI list, the PI list used by other screens and the order bookings. |
| `src/features/pi-status/components/pi-status-badge.tsx` | `CLOSED` status. Shows "Closed" or "Auto closed", with the reason on hover. |
| `src/features/pi-status/components/pi-status-table.tsx` | Row menu gets **Close PI** (above Delete) and **Reopen PI**. For closed PIs, Approve, Edit, Delete, Clone and Request Change are hidden. The badge gets the auto closed flag and reason. |
| `src/features/pi-status/pi-status-page.tsx` | New **Closed** tab after Approved, a **Closed PI's** KPI card (the KPI row is now 5 cards wide), and the Close and Reopen confirm dialogs with their success and error messages. |
| `src/components/ui/confirm-action-modal.tsx` | no real change, reused as it is for both dialogs. |

## Why

Users needed to close PIs that are going nowhere and to bring them back, and to see which ones the system closed on its own. The Reopen behaviour copies the Enquiries page: a kebab item shown only for closed records, a confirm dialog in the info style, then a refresh and a toast.

## How

**Close PI**: shown in the row menu above Delete for any PI that is not fully approved and not closed. It opens the existing confirm dialog. The backend also checks the rule, so the hidden menu item is only a convenience.

**Reopen PI**: shown only for closed PIs, in any tab where they appear (Closed and All). It opens the confirm dialog in the info style, and the PI goes back to Pending.

**Closed tab and KPI card**: the tab count and the card count both come from the `CLOSED` status count the backend already sends with the page, so there is no extra request.

**Kept the same on purpose**: the Pending and Approved tabs, the approval modals and the Change Requests button are untouched. Closed PIs do not show in Pending or Approved because the backend filters by status.

## Decisions made

**No reason box in the Close dialog.** The shared confirm dialog has no input field. Adding one would change a component used on many pages. A manual close is saved without a reason, and the auto close reason is shown from the backend.

**No "auto closes on <date>" hint on pending rows.** It was in the plan as nice to have. It is left for later so this PR stays small.

**KPI row is 5 cards on wide screens.** The grid changed from 4 to 5 columns so the new card sits in the same row.

**Reopen shows on the All tab too.** The menu decides by the PI status, not by the tab, so a closed PI can be reopened wherever it is listed.

## Notes

- The new menu items are disabled for users who only have view access, the same as the other items.
- The files use CRLF line endings like the rest of the repo.

## Next steps

- Add the "auto closes on <date>" hint to pending rows.
- Add an optional reason field to the shared confirm dialog and use it for Close.

## Test plan

**Automated tests (no tests added, build only):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes with no errors.

**Manual checks (local):**
- [ ] Open PI Status, close a Pending PI from the row menu, and check it moves to the Closed tab and the card count goes up - not run yet, needs the backend with migration 075.
- [ ] Check a fully approved PI has no Close PI item - not run yet.
- [ ] Reopen a closed PI from the Closed tab and check it returns to Pending - not run yet.
- [ ] Check an auto closed PI shows "Auto closed" and the reason on hover - not run yet.
- [ ] Check the 5 KPI cards fit on a narrow screen - not run yet.
