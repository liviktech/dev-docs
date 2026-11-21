# 2026-11-20 - fir-rework-replacement (frontend)

## Summary

The FIR page could only record a complaint. This PR lets the user create a Rework / Replacement FIR: pick the original work order, pick pieces, choose Rework or Replacement for each, pick the fault, and see the Replace Orders (RO) that were created. It also adds a View Replace Order window, an RO progress window in Dispatch, and fixes the Dispatch list for RO rows.

## What changed

| File | Change |
|---|---|
| `src/features/fir/fir-form-modal.tsx` | Kind switch, pieces, Rework / Replacement per piece, reason, rework stages, fault, charge, penalty employees. Fonts and boldness match the rest of the form. |
| `src/features/fir/fir-ro-section.tsx` (new) | RO section shown inside the FIR. |
| `src/features/fir/fir-view-dialog.tsx`, `src/features/fir/fir-document-views.tsx` | Show pieces, fault, penalty or charge, and ROs. |
| `src/features/fir/fir-page.tsx` | RO codes in the Work Order column as a "+N" chip. |
| `src/features/fir/api.ts`, `queries.ts`, `fir-options.ts` | New types, calls and options. |
| `src/features/order-booking/replace-order-view-modal.tsx` (new) | Read-only View Replace Order window, same look as View Work Order. |
| `src/features/order-booking/api.ts` | `getReplaceOrder` call. |
| `src/routes/work-orders.tsx` | RO code opens the View Replace Order window. The Approve Work Order dialog lets a replacement choose its glass lot. |
| `src/features/dispatch/components/replace-order-progress-modal.tsx` (new) | RO progress window: pieces done, and each piece's stages. |
| `src/features/dispatch/components/dispatch-table.tsx`, `dispatch-page.tsx`, `types.ts`, `api.ts` | Progress button opens the RO window for RO rows. Approve is enabled when ready. Total Amount shows the client charge, or a dash for company fault. |

## Why

Users needed to raise a fix from a complaint without leaving the existing screens. Most of it reuses existing dialogs and tables. The real work was the FIR form and making the Dispatch list behave for a shipment that has no PI.

## How

**FIR form**: the kind switch shows the pieces section only for Rework / Replacement. Rework asks for the stages to repeat. Fault Company asks for employees, fault Client asks for the charge.

**Progress**: a piece counts as done when all of its stages are completed or skipped. This is the same rule that makes Approve Dispatch available.

**Dispatch list**: the newest pending dispatch is at the top, whether it came from a PI approval or an RO. The ordering itself is done by the backend.

## Notes

- The scan screen's replacement lot dialog was not checked for RO pieces.
- No kind badge was added to the FIR list yet.

## Next steps

- Walk the full flow in the browser after backend migration 166 is run.
- Add the kind badge to the FIR list.

## Test plan

**Automated tests:**
- [x] TypeScript check: clean.
- [ ] No frontend test files were added or run for this change.

**Manual checks (local):**
- [ ] Not possible yet: the backend migration is not run, so no screen was opened. Needs a pass on FIR create, View Replace Order, approve RO, RO progress and the Dispatch list.
