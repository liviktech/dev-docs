# 2026-11-17 - fir-delay-reason (frontend)

## Summary

The FIR form did not show when the related invoice was made or dispatched, and a complaint could be raised at any time with no explanation. This PR shows the Invoice Date and Dispatch Date (read only) in the FIR form and in the FIR details view. When a new FIR is raised more than 48 hours after the reference date, it shows a warning and a required "Reason for Delayed Complaint" box. The saved reason is shown when the FIR is viewed or opened for edit.

The type check passes. The screens were not opened in a browser, so none of this was checked visually.

## What changed

| File | Change |
|---|---|
| `src/features/fir/api.ts` | `Fir` gets `delayReason`, `invoiceDate` and `dispatchDate`. `FirInput` gets `delayReason`. New `FirReferenceDates` and `FirReferenceLinks` types and a `getFirReferenceDates` call to the new `/firs/reference-dates` endpoint. |
| `src/features/fir/queries.ts` | New `firReferenceDatesQueryOptions`. It always reads fresh, because whether a reason is needed depends on the clock. |
| `src/features/fir/fir-form-modal.tsx` | New read only "Invoice & Dispatch Information" section (Invoice Date and Dispatch Date). New amber warning and required "Reason for Delayed Complaint" box for a late new FIR, with a check before Save. An existing FIR shows its saved reason read only. |
| `src/features/fir/fir-view-dialog.tsx` | The details view shows Invoice Date and Dispatch Date, and the delay reason when there is one. |

## Why

The team raising or reviewing a complaint needs to see the invoice and dispatch dates next to it, and a late complaint needs a recorded reason. The dates come from the server (the backend PR for this task), so the form never lets anyone type or change them.

## How

**When the dates appear**: For a new FIR, the form asks the server once a customer and one linked document are chosen. It sends only one document, in the order invoice, then work order, then PI, which is the same order the server uses. Picking an invoice fills in the work order and PI automatically, but because only the invoice is sent, this does not trigger a second request. An existing FIR already comes with its dates in its details, so it makes no extra request.

**The late warning**: The server says whether the complaint is late. The form shows the amber warning ("Attention: This complaint is being raised more than 48 hours after the Dispatch Date (or Invoice Date). Please provide a valid reason for the delay.") and the required reason box only for a new FIR that is late. The wording says Dispatch or Invoice depending on the date the server counted from.

**Save is blocked until a reason is given**: If the FIR is late and the reason is empty or only spaces, Save shows "Reason for the delayed complaint is required" under the box and does not send anything. The reason is only sent when the FIR is late. The server also checks this on its own, so the form is a convenience, not the only protection.

**Left the same on purpose**: An existing FIR can still only change its status and internal remarks. The new fields on an existing FIR are read only.

## Decisions made

**The late result comes from the server, not worked out in the browser.** The browser clock and time zone can be wrong, and the server already has the exact reference moment. The cost is that the warning appears after the dates load, not instantly when a document is picked.

**The late check can differ by a moment at save time.** The warning is based on the time the dates were loaded, and the server checks again when saving. A complaint that was on time when the form was opened could cross 48 hours before Save is pressed. In that case the server refuses it with a clear message, and the form does not show the reason box until the dates are read again.

## Notes

- This needs the backend changes and the `143_fir_delay_reason.sql` migration to be in place. Until the migration is run on the database, the FIR screens will fail to load.
- The reason box and warning only appear for a new FIR. When opening an existing FIR, the saved reason is shown read only if there is one.

## Test plan

**Automated tests (none added):**
- [ ] No automated test was added. The frontend has no existing test for the FIR form to extend.
- [x] `npx tsc -b --noEmit`: passes with no errors.

**Manual checks (local):**
- [ ] New FIR, pick a customer and an invoice, check Invoice Date and Dispatch Date show and cannot be edited - not run, the screens were not opened in a browser.
- [ ] New FIR on an invoice more than 48 hours old, check the warning and reason box appear and Save is blocked with an empty or spaces only reason - not run in a browser. The backend side of this was checked (see the backend PR).
- [ ] New FIR on a recent invoice, check no warning and no reason box - not run in a browser.
- [ ] Open a late FIR from the register, check the dates and the saved reason show in the details view and in edit - not run, it needs the migration and a saved late FIR first.
