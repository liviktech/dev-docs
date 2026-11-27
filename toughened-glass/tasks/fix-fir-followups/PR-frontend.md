# 2026-11-26 - dispatch-eway-fir-fixes (frontend)

## Summary

The Dispatch screens allowed picking part of an invoice, showed Create Gate Pass to users the server would refuse, filled the gate pass with the party's address instead of the PI's, and did not show when a dispatch was held or closed or how many e-Way Bills existed. This PR fixes these, adds a PI Number column, removes Extend e-Way Bill validity, limits the FIR date, and redesigns the View FIR dialog. Most dispatch changes were checked live on the local app, the FIR date limit and the new FIR layout were not formally checked (see the test plan).

## What changed

| File | Change |
|---|---|
| `src/features/dispatch/components/approve-dispatch-modal.tsx` | Pieces of one final invoice are ticked and unticked together, with a line saying so. Clearer message when no invoice is ready. |
| `src/features/dispatch/components/dispatch-table.tsx` | New PI Number column that opens the PI. Status shows On Hold or Closed. Approve is not offered on a Closed row and explains why on a held one. Gate pass tooltip says when the permission is missing. |
| `src/features/dispatch/dispatch-page.tsx` | Create Gate Pass needs the Gate Pass right only. Opens the PI drawer from the new column. |
| `src/features/dispatch/components/dispatch-detail-modal.tsx` | e-Way Bill heading shows how many bills are generated. A dispatch without a final invoice says "Not applicable". Create Gate Pass is disabled without the right. Extend validity button and dialog removed. |
| `src/features/dispatch/components/gate-pass-modal.tsx` | "To Address" starts with the PI delivery address, else the party's. It stays empty until the details load. |
| `src/features/dispatch/components/delivery-order-modal.tsx` | Packing Slip / Delivery Order address order is now gate pass, then PI delivery address, then party. |
| `src/features/dispatch/components/pi-dispatches-modal.tsx` | "Closed Dispatch" title and separate messages for closed and on-hold dispatches. |
| `src/features/dispatch/piece-stage.ts`, `src/features/dispatch/types.ts`, `src/features/dispatch/api.ts` | New On hold and Closed states, `hold` and `deliveryAddress` fields. |
| `src/features/e-way-bill/queries.ts` | Generating or cancelling a bill refreshes the dispatch data too, so the new bill shows at once. Removed the extend hook. |
| `src/features/e-way-bill/api.ts`, `src/features/e-way-bill/types.ts` | Removed the extend validity call and types. |
| `src/features/e-way-bill/components/extend-ewaybill-validity-modal.tsx` | Deleted. |
| `src/features/fir/fir-form-modal.tsx`, `src/features/fir/api.ts` | The FIR date cannot be in the future or before the dispatch date (invoice date when nothing is dispatched). The FIR date is sent when checking the 48-hour delay. |
| `src/features/fir/fir-view-dialog.tsx` | Rebuilt to match View Work Order: summary tiles, detail cards, and tables for pieces and Replace Orders instead of long text lines. |
| `src/features/fir/fir-ro-section.tsx` | Piece table restyled (text size, weight, spacing, product name casing). No change in behaviour. |

## Why

The dispatch changes come from the same audit as the backend PR: the screen should say what the server will accept, and show why something is held or closed. The e-Way Bill section existed but gave no way to tell whether a bill was generated or how many were left. The FIR date limit matches the new server rule, and the old FIR view packed all pieces into one text block that was hard to read.

## How

**The server decides, the screen explains**: `hold` comes from the API and drives the status label, the disabled Approve button and the popup message.

**Whole invoice picking**: the dialog groups ready pieces by final invoice when ticking, so a partial selection cannot be made.

**One address order**: gate pass and packing slip use the same PI delivery address as the e-Way Bill. An address the user has typed is not overwritten.

**Existing dialog parts reused**: the new FIR view uses `FormDialog` and the `SectionCard`, `StatTile` and `Field` parts from order booking.

## Decisions made

**A Closed dispatch keeps its row but offers only View Details.** It sits under the Cancelled filter, so it can still be found without cluttering Pending.

**An empty e-Way Bill section is shown with a reason.** A Replace Order has no final invoice, and leaving the section out looked like a missing feature.

## Bugs found while testing (already there before, not fixed here)

**1. The FIR piece reason box sits inside a small scrolling table.** It is easy to miss on a smaller screen. Affects anyone raising a Rework or Replacement FIR.

**2. The Replace Order dispatch popup does not show the FIR number.** The table shows it, the popup does not.

**3. Lint errors in changed files.** `eslint` reports 46 problems across the changed files, the same count as before this change. Most are in `delivery-order-modal.tsx`.

**4. `gst-compliance-panel.tsx` is not used anywhere.**

## Notes

- Needs the backend PR of the same branch (`hold`, `deliveryAddress`, FIR date in the reference dates call).
- Users who have only the Approve or Control right no longer get Create Gate Pass. The server already refused them. Give them the Gate Pass right if they need it.
- Git warns that `fir-ro-section.tsx` and `fir-view-dialog.tsx` use LF and will be changed to CRLF on the next touch.

## Next steps

- check the Gate Pass button with a user who lacks the right.
- show the FIR number in the Replace Order dispatch popup.
- remove `gst-compliance-panel.tsx` if it is not planned for use.

## Test plan

**Automated tests (type check and build passing, no frontend test files):**
- [x] Type check (`tsc -b`): no errors.
- [x] `vite build`: built successfully.
- [ ] `eslint` on changed files: 46 problems, all old, none added.

**Manual checks (local):**
- [x] Approve Dispatch: pieces of one invoice ticked together.
- [x] Dispatch table: PI Number column shows and Extend validity is gone.
- [x] Dispatch with two invoices: bill count moved from 0 to 1 to 2 of 2. A Replace Order popup showed "Not applicable".
- [x] Gate pass and packing slip showed the PI delivery address.
- [x] Generate e-Way Bill was disabled before the dispatch was approved.
- [x] Cancelled filter: a declined PI's dispatch showed Closed, with only View Details and the "Closed Dispatch" popup.
- [ ] On-hold popup message: not viewed in the app after it was added.
- [ ] Create Gate Pass for a user without the right: no non-admin login was used.
- [ ] FIR date limit and the new View FIR and piece table layout: not checked as part of this work, covered only by the type check and build.
