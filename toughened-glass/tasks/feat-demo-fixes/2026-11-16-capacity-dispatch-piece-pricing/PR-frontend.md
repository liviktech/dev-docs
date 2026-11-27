# 2026-11-16 - capacity-dispatch-piece-pricing (frontend)

## Summary
This branch (`feat/demo-fixes`, three commits on top of `dev`) groups several demo fixes in the frontend. The production plan screens had no capacity view, progress, or summary. A dispatch plan could not be postponed and did not show if its glass was ready. A Final Invoice priced the pieces you tick as an equal share. The product page let charges be saved with the wrong quantity source. The same email could be typed for two users or two sales reps. This PR adds all of these on the screens, using the backend figures. Only the type check was run. I did not open the changed screens in a browser, except for the dispatch plan view and postpone dialog, where the user saw a server error that was fixed in the backend.

## What changed
| File | Change |
|---|---|
| `src/features/production-plan/daily-capacity-meter.tsx` (new) | A meter for one day: how much of the limits (templates, cutouts, holes, machined SQM, total SQM) is used. It can step to the next or previous day. |
| `src/features/production-plan/production-plan-editor.tsx` | Shows the capacity meter for the plan's date at the top of the editor. |
| `src/features/production-plan/production-plan-panel.tsx` | Capacity meter above the list, a date range filter with a reset button, and a new Progress column. |
| `src/features/production-plan/production-plan-progress.tsx` (new) | The Progress cell and its dialog. It lists each work order, only the product lines and stages the plan covers, with pieces done, SQM, progress bar, downtime and status. |
| `src/features/production-plan/production-plan-view-modal.tsx` | New summary tiles (work orders, plan lines, pieces done and planned, area done and planned) and a "Planned vs Completed Tracking" card. The card now uses the same card style as the other cards. The Reject button is a plain outline button with an X icon. |
| `src/features/production-plan/api.ts`, `queries.ts` | Types and calls for the plan summary, the daily capacity, the capacity check, the date range, and the plan lines on each work order in the list. |
| `src/features/production-scan/types.ts`, `src/features/production-scan/components/work-order-progress.tsx` | Stage progress now has an SQM column that shows the stage's total SQM. The same row is used by the work order progress dialog, so it shows there too. |
| `src/features/dispatch-plan/postpone-dispatch-modal.tsx` (new) | The Postpone Dispatch Plan dialog: new date, one of six reasons, remarks (10 characters minimum). It warns first if the plan is already scheduled for a future date. |
| `src/features/dispatch-plan/dispatch-plan-view-modal.tsx` | A Readiness column per line (Ready for Dispatch or In Production, Not Ready), a Postponement History table, a Postpone Dispatch button, and the Reject button restyled. Reject now needs a reason. |
| `src/features/dispatch-plan/dispatch-plan-panel.tsx` | Each plan gets View, Approve, Postpone (once approved), Edit and Delete in its menu. Buttons that are not allowed show a reason when you hover instead of disappearing. |
| `src/features/dispatch-plan/api.ts`, `queries.ts` | Types and the call for postponing, plus the readiness and history fields. |
| `src/features/pi-status/components/approve-party-modal.tsx`, `pi-status-page.tsx`, `queries.ts`, `api.ts` | Factory approval asks for a promised delivery date, delivery type and express surcharge, and checks capacity for same-day delivery. |
| `src/features/order-booking/work-order-edit-dialog.tsx`, `work-order-form-fields.tsx` | The delivery date comes from the PI approval, so it shows as read only on approval. |
| `src/features/final-invoices/api.ts`, `components/create-final-invoice-panel.tsx` | Ticked pieces are priced by what each piece is worth from its own size, using the new per-piece amounts. All pieces ticked still shows exactly what the server priced. |
| `src/features/invoices/create-proforma-invoice.tsx`, `src/features/pi-status/components/pi-status-drawer.tsx`, `src/features/order-booking/api.ts` | Each size row's Charges column uses the server's per-size share when it matches the item's charges, otherwise the old split by SQM. |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx` | PI sheet columns use fixed widths and the customer name is removed from the sign box, matching the emailed PDF. |
| `src/features/order-booking/add-product-modal.tsx`, `charges-table.tsx`, `shape-config-card.tsx` | The per-product charge list hides charges that the screen's own fields already handle (holes, cutouts, logo, work centers). "Add New Charge" now opens the normal Charge dialog. The unit (NOS or SQM) comes from the charge and cannot be changed on the row. An SQM charge starts at the product's chargeable area. The Fabrication hint text under Shape is removed. |
| `src/features/products/charge-quantity-source.ts` (new) | Works out the usual Quantity Source of a charge from its code and name (Big Hole is Big Hole Qty, Polish is Area, and so on) and writes the warning text. |
| `src/features/products/product-dialog.tsx` | Picking a charge fills in its usual Quantity Source. Saving with an unusual one shows a warning, and a second Save click saves it as is. A "Usually ... - use it" link fixes a row. |
| `src/features/attendance/employee-dialog.tsx`, `src/features/pwa/attendance/mobile-employees.tsx` | Base salary starts empty and is held as text. The employee dialog checks for a duplicate email and shows the server's email message on the email field. The mobile field has a +91 prefix. |
| `src/features/sales-reps/sales-rep-dialog.tsx` | A duplicate email among sales reps is refused. The employee picker shows the mobile number only when two employees share a name. Phone numbers are cleaned to 10 digits. |
| `src/features/inventory/components/purchase-dialog.tsx`, `src/features/pwa/inventory/mobile-purchase-form.tsx`, `src/lib/lookup.ts` | Purchase only offers items that the product page offers (an active rate row exists), using the new `inProduct` flag. An item that a row already uses stays selectable so old rows can still be edited. |

## Why
The screens needed to show the new backend figures: capacity before a plan or same-day order is promised, progress and SQM on a plan, readiness and history on a dispatch plan. The Purchase fix follows the intended flow that items come from the product page, not from the item list. The product charge warnings stop a charge from being saved with a quantity source that silently drops or mis-prices it. Most of the work was the final invoice panel and the product dialog, because both had to keep old behaviour when the new data is not there.

## How
**Plan progress shows only what the plan covers**: the stage progress endpoint returns every stage of every product of a work order. The plan list now sends the product lines and stages of each plan, and the progress dialog keeps only those. A line with no stage keeps the whole product. Piece counts, the work order header and the overall percentage are unchanged.

**SQM column**: the stage row shows the stage's total SQM from the backend, rounded to two decimals.

**Server numbers first**: the Final Invoice panel, the PI form and the PI drawer use the per-piece and per-size figures from the backend. They fall back to the old way only when those figures are missing or do not add up to the item's charges, so old data keeps working.

**Two step save on the product page**: the first Save with an unusual quantity source shows the warning and stops. Any edit of the charges clears the warning. A second Save click saves as is.

**Kept the same on purpose**: Order Booking's Add Product options still come mainly from the product's rate lines. The edit and delete rules for plans (only the person who raised it, and not once approved) are the same, now shown as disabled buttons with a reason.

## Decisions made
**Purchase hides items not on the product page, but keeps ones already used.** I did not hide every item without a rate row, because that would break editing and cloning old purchases.

**Progress filter is on the screen, not in the endpoint.** The endpoint is shared with the work order screens. I did not change it so those screens keep showing the full route.

**SQM column shows the total only.** I first showed done / total, then changed it to total only on request.

## Notes
- The three branch commits are `30355cd` (employee dialog), `2f1770c` (product charge setup) and `e589a1a` (capacity meter). The other changes listed here are part of these commits.

## Next steps
- Confirm the progress percentage rule and the shift question with the team.
- Add tests for `charge-quantity-source.ts` and the Final Invoice piece pricing on the screen. The frontend has no test script at the moment.

## Test plan
**Automated tests (no new tests added):**
- [x] Frontend type check (`npx tsc --noEmit -p tsconfig.app.json`): passes.
- [ ] Lint (`eslint .`) - not run for this document.

**Manual checks (local):**
- [x] Dispatch plan view and Postpone dialog were opened in the browser by the user, which showed two backend errors. They are fixed in the backend PR and not yet re-checked.
- [ ] Capacity meter in the plan editor and the plan list - not checked.
- [ ] Plan Progress dialog with a plan that covers only some stages, and the SQM column - not checked.
- [ ] Planned vs Completed Tracking card style in the plan view - not checked after the change.
- [ ] Factory approval with a same-day date over the limit - not checked.
- [ ] Final Invoice with some pieces ticked, with different sized pieces - not checked.
- [ ] Product dialog charge warning and the "use it" link - not checked.
- [ ] Duplicate email on employee and sales rep dialogs - not checked.
- [ ] Purchase dropdown on desktop and mobile - not checked.
