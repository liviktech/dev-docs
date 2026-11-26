# 2026-11-25 - fir-workflow-fixes (frontend)

## Summary

The FIR screens had a status dropdown in the edit form that allowed any jump, a Processing Type field that only looked like it did something, and a Closed tab nothing ever reached. Editing a FIR saved only the internal remarks. A Resolved FIR could still be edited and deleted, and its status icon was not a green tick. Pieces not yet dispatched could be picked for a Rework/Replacement FIR. The +Add buttons in the replacement specs opened the separate master pages instead of the Items page. On Dispatch, Replace Order shipments did not show their FIR, and a cancelled one could not be reopened. This PR fixes all of these on desktop and mobile. Everything was checked with the type check only, not live in the app.

## What changed

| File | Change |
|---|---|
| `src/features/fir/fir-page.tsx` | Processing Type column replaced by FIR Type. Row menu: "Move to In Progress" on Open, "Move to Resolved" on In Progress (Complaint FIRs only). Edit and Delete only while Open or In Progress. A Resolved FIR shows "View" instead. Closed tab removed. |
| `src/features/fir/fir-form-modal.tsx` | Status and Processing Type fields removed. Every field is editable when editing (a Rework/Replacement FIR keeps its customer and documents locked, with a note saying why). Delay reason and the client-fault charge can be edited. The save sends every field. Reference dates are looked up again only when the linked documents change. |
| `src/features/fir/fir-piece-config-dialog.tsx` | The four spec dropdowns (glass, thickness, colour, glass type) get "+ Add Item", which opens the Items form prefilled with the values from the FIR. The saved item fills back into the FIR. Needs the Items ADD right. The nested form is wrapped so its submit does not submit the FIR form. |
| `src/features/fir/fir-ro-section.tsx` | Pieces that are not dispatched yet are shown disabled with "not dispatched yet". |
| `src/features/fir/fir-view-dialog.tsx` | Shows FIR Type. Edit button is disabled when the FIR is Resolved. |
| `src/features/fir/fir-options.ts` | Closed removed from the status options (old rows still label as Closed). New FIR Type options and label. Processing type helper removed. |
| `src/features/fir/capabilities.ts` | New `isFirEditable(status)`: true only for Open and In Progress. |
| `src/features/fir/api.ts`, `src/features/fir/queries.ts` | `FirUpdateInput` takes every field as optional. New `moveFirStatus` and `useMoveFirStatus`. |
| `src/components/ui/status-label.tsx` | Resolved shows the green tick. |
| `src/features/pwa/fir/fir-format.ts` | Mobile: Resolved is green, Closed is grey. |
| `src/features/pwa/fir/mobile-fir-form.tsx` | Mobile: status and processing type removed. A Complaint FIR is fully editable. A Rework/Replacement FIR edits only its remarks. |
| `src/features/pwa/fir/mobile-fir-detail.tsx` | Mobile: Edit and Delete hidden on a Resolved FIR, with a banner saying it is resolved. |
| `src/features/pwa/fir/mobile-fir-list.tsx` | Mobile: shows FIR Type. |
| `src/features/dispatch/components/dispatch-table.tsx` | New FIR Number column for Replace Order shipments, which opens the FIR. New "Reopen Dispatch" row action on a cancelled Replace Order dispatch. |
| `src/features/dispatch/components/reopen-dispatch-dialog.tsx` (new) | Confirm dialog for the reopen. Lists the server's reasons when it is refused. |
| `src/features/dispatch/components/cancel-dispatch-dialog.tsx` | For a Replace Order dispatch, says it is the only dispatch and can be reopened later from Cancelled. |
| `src/features/dispatch/dispatch-page.tsx` | Wires the FIR view dialog and the reopen dialog. Reopen needs the dispatch edit or approve right. |
| `src/features/dispatch/api.ts`, `src/features/dispatch/queries.ts`, `src/features/dispatch/types.ts` | `firId` on the dispatch record, `reopenShipment` and `useReopenShipmentMutation`. |

## Why

These fix problems found in the FIR screens while testing them, and follow the user's decisions during review: no Closed tab, status moves only from the row menu, every field editable until Resolved, no edit or delete after Resolved, a green tick for Resolved, and +Add opening the Items page prefilled from the FIR. The Dispatch changes give Replace Order shipments their FIR number and a way back from an accidental cancel. The fiddly part was the edit form, which had to keep a Rework/Replacement FIR's customer and documents fixed while letting everything else change.

## How

**Status moves from the row menu, not the form**: the form no longer has a status field. The menu shows only the next step for the tab you are on, and the server checks it again.

**One rule for "can still change"**: `isFirEditable` is used by the desktop menu, the view dialog and the mobile detail, so Edit and Delete disappear together once a FIR is Resolved.

**+Add opens Items, prefilled**: the product name, glass type, colour and thickness from the FIR are passed into the Items form. When it is saved, the new item's values are put back into the FIR spec fields.

**Nested form submit**: React submit events bubble through portals, so the Items form sits inside a wrapper that stops the event. Without it, saving the item could also submit the FIR.

## Decisions made

**+Add opens the Items page only**, not the separate glass thickness, colour or type pages. The user asked for this so new values are added from one place.

**Mobile keeps a Rework/Replacement FIR to remarks only.** The mobile form has no piece or Replace Order section, so editing the other fields there could not be shown safely.

## Notes

- The Reopen action needs backend migration `184_dispatch_reopen.sql` to be run first.
- Lint still reports some errors in the FIR files. They were there before this change and are not from it.

## Test plan

**Automated tests (no frontend tests for these screens):**
- [x] Type check (`tsc --noEmit -p tsconfig.app.json`): no errors.
- [ ] Unit tests: none exist for the FIR or Dispatch screens.

**Manual checks (local):**
- [ ] Row menu on Open, In Progress and Resolved tabs shows the right actions: not checked live.
- [ ] Edit a Complaint FIR and a Rework/Replacement FIR and see every field saved: not checked live.
- [ ] "+ Add Item" in the replacement specs opens Items prefilled and fills back the FIR: not checked live.
- [ ] FIR Number column and Reopen Dispatch on the Dispatch page: not checked live, needs migration 184.
- [ ] Mobile FIR list, detail and form: not checked live.
