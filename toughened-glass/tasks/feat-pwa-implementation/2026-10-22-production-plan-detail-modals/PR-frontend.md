# 2026-10-22 - production-plan-detail-modals (frontend)

## Summary

There was no way to plan production for a work order, and the Final Invoice and Dispatch detail modals used their own fonts and a different layout from the Proforma Invoice drawer. This PR adds a Production Plan tab on the Work Orders page, restyles the two detail modals to match the PI drawer, and tidies the loading state on the Final Invoices and Dispatch lists. The Proforma Invoice sheet also now lists product charges before the order level extras.

The Production Plan screen is wired to the new backend endpoints (see PR-backend). Everything compiles and lints clean for the new files, but **none of the screens were opened in a browser** (see "Test plan").

**Notifications for plan approval are not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/production-plan/api.ts` (new) | API calls and types for plans: list, get, save, submit, approve, reject, delete. |
| `src/features/production-plan/queries.ts` (new) | React Query options and mutations. Each mutation writes the fresh plan into the cache and refreshes the plan list and the work order list. |
| `src/features/production-plan/production-plan-panel.tsx` (new) | The Production Plan tab: filter tabs (Pending, Approved, All), search, a table of plans, a Create Plan button, and a row menu (View, Edit, Delete). The work order number opens the work order view modal. |
| `src/features/production-plan/production-plan-modal.tsx` (new) | The modal used to create a plan (pick a work order, then fill it in) or open an existing plan in View or Edit mode. |
| `src/features/production-plan/production-plan-tab.tsx` (new) | The plan editor: line table, Save draft, Submit, Approve, Reject with reason, Delete. Field widths are fixed so the modal does not scroll sideways. |
| `src/routes/work-orders.tsx` | Adds the Production Plan tab (only for users with the plan view right) and a Plan status column in the work order list. |
| `src/features/order-booking/work-order-list-state.ts` | The `tab` URL value can now be `plan` as well as `approved`. |
| `src/features/order-booking/api.ts` | `WorkOrderListItem` gets an optional `planStatus`. |
| `src/components/ui/status-label.tsx` | Adds SUBMITTED (pending look) and REJECTED (rejected look) to the status tones. |
| `src/components/ui/detail-modal.tsx` (new) | Shared pieces for read-only detail modals: modal shell classes (full and compact), section card, "Label : value" field, amount rows. Copied from the PI drawer so all detail views use the same sizes and the app font. |
| `src/components/ui/table-loading-panel.tsx` (new) | The white bordered loading panel (same as the Work Orders list) with the loader in the middle of the screen. |
| `src/features/final-invoices/components/final-invoice-detail-modal.tsx` | Rebuilt to match the PI drawer: same typography, sections and amount breakdown order (Glass & Surcharges card, Tax card with Grand Total and advance block). Each line item has an expandable "Product N Charges" row with a charge table and subtotal, read from the order booking. Removed the hard coded Hanken Grotesk font and `font-mono`. |
| `src/features/dispatch/components/dispatch-detail-modal.tsx` | Restyled with the shared detail modal pieces and a compact (shorter) shell. Shows the dispatch code in the header. |
| `src/features/final-invoices/final-invoices-page.tsx` | Shows the shared loading panel while loading or switching tabs, instead of the loader inside the table. |
| `src/features/dispatch/dispatch-page.tsx` | Same loading panel change as Final Invoices. |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx` | The charge rows in the amount breakdown are now ordered: product charges first, then order level extras (anything in the Order Charges master, plus anything with "Insurance" in its name). The T&C block also now starts with the delivery note, in spaced capitals, and the separate "Payment Terms & Delivery Conditions" box is removed. |

## Why

Planning had no place in the app, so planners and the Production Manager had nothing to work from. The detail modals were also hard to read side by side with the PI drawer because they used a different font, label sizes and amount order. The loading and charge order changes came from review feedback on those same screens.

## How

**Production plan flow**: a plan belongs to one work order and moves DRAFT, SUBMITTED, then APPROVED or REJECTED. The planner creates, edits and submits. The Production Manager or an Admin approves or rejects. A rejected plan can be edited and submitted again.

**Who can do what**: only the person who raised the plan can edit, submit or delete it, and only until it is approved. Nobody can approve or reject a plan they raised, admins included (the backend enforces this, the UI hides the buttons). The row menu shows Edit and Delete only to the owner of an unapproved plan.

**View and Edit**: View opens the saved plan read only (approvers can still approve or reject). Edit opens the same plan with the fields enabled and the saved values filled in. Create Plan is always editable.

**Plan list tabs**: Pending means every plan that is not approved (draft, submitted, rejected). The list is filtered in the browser. Switching tabs reloads the plans behind the loader, held for about 0.35 seconds so it is seen.

**Editor remount**: the editor is keyed by the plan id, status and update time, so it fills itself again from the server after every save, submit, approve or reject. Because of this, success toasts use `mutateAsync` instead of callbacks on `mutate`, which would be dropped when the editor remounts.

**Detail modal styling**: the section heading is 15px bold, labels are 14px bold with a colon, and values are 13px semibold, the same as the PI drawer. No font family is set on the modals, so they use the app font.

**Per product charges in the final invoice modal**: final invoice items have no charges of their own, so the charges are read from the order booking the item was billed from (matched by the order booking item id), the same way the PI drawer does it. The expand bars only appear once that booking has loaded.

## Decisions made

**Pending includes drafts.** A plan that is still a draft or was rejected also shows under Pending. I did not limit Pending to submitted plans so planners can still see their own unfinished work. It is a one line change if you want it narrower.

**Plan lines are free form.** Each line has a description, an optional work centre, quantity, dates and remarks. They are not tied to individual work order items or pieces.

**Work centre is stored by code.** The Work Centre master no longer has an `id` column (its key is company and code), so the plan stores the code. The API field is still called `workCenterId` and carries the code, matching how the work centre API already exposes it.

**"Total Charges" row left out of the final invoice breakdown.** The Proforma Invoice drawer shows it, but final invoice items do not say whether product charges are already inside the item amounts, so adding it could double count.

**Insurance is matched by name.** The Order Charges master has Admin Charge and Wooden Box Packing but no Insurance entry, so Insurance is treated as an extra by name. If other extras should also go last, add them to the master.

**Approving a plan does not change the work order status.** The plan is tracked on its own.

## Bugs found while testing (already there before, not fixed here)

**1. Lint errors in files this PR touches.** `final-invoice-detail-modal.tsx` still has `any` type errors in the address lookup code (around 20 errors), and `work-orders.tsx` and the work order route file have set-state-in-effect and fast-refresh warnings. These were there before and were left as they are.

## Notes

- The PI sheet T&C change (delivery note moved to the top, spaced capitals) was a request on its own, it is in the same file as the charge order change.
- Fixed field widths in the plan modal mean it will scroll sideways on screens narrower than about 1300px. This was asked for on purpose.

## Next steps

- Add notifications: tell the Production Manager and Admin when a plan is submitted, and tell Executive and Customer Care when it is approved. The backend service has marked hook points for this.
- Mobile (PWA) screens for production plans.

## Test plan

**Automated tests (no new tests added):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes with no errors.
- [x] `npx eslint src/features/production-plan/`: clean.
- [ ] No unit tests exist for these screens, none were added.

**Manual checks (local):**
- [ ] Production Plan tab: create, save, submit, approve, reject, edit, delete as different roles - **not checked in a browser**. Only the code and types were checked.
- [ ] Self approval block, plan locked after approval - **not checked live**. Covered by the backend rules in PR-backend.
- [ ] Final invoice and dispatch modals, loading panel, PI sheet charge order and T&C block - **not checked live**.
