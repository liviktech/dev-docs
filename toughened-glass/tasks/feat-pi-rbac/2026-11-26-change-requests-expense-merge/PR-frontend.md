# 2026-11-26 - change-requests-expense-merge (frontend)

## Summary

This covers everything on `feat/pi-rbac` that is not yet in `dev`. Approved records can now be changed through a Change Request, the expense screens from `fix/expense-ui` are merged in, and several modules get new row actions (edit, delete, clone, cancel, reopen). Toast messages now come from shared files, and technical error text is no longer shown to users. The type check passes. 3 capability tests that pass on `dev` fail here (see "Test plan"). The screens were not tried in the running app.

**Delete on approved expenses is not offered on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/change-requests/*` (new) | API calls, the Change Request popup, the header button and the review list. |
| `src/features/expense-management/*` | Draft and Approved only. Approved rows show Change Request and Clone, drafts show Edit, Approve and Delete. Clone is dated today with a blank bill number and date. No category codes. |
| `src/features/expense-management/expense-reason-dialog.tsx` (removed) | Cancel with a reason no longer exists. |
| `src/features/auth/permissions.ts`, `roles/rights-picker.tsx`, `components/layout/nav-items.ts`, `routes/expense-management.categories.tsx` | New expense rights, with a Type group on the Categories page. |
| `src/features/sequences/sequence-dialog.tsx`, `routes/admin.sequences.tsx` | Expense sequence added. View row action removed. |
| `src/features/archive/*` | Expense and Purchase Return in the Archive. |
| `src/features/pi-status/*`, `src/features/pwa/pi-status/*` | New PI action rules (see "How"). Delete removed, Cancel added for pending PIs. Reopened PIs show party and admin approval as done. |
| `src/features/order-booking/*`, `pwa/order-booking/mobile-order-detail.tsx`, `routes/orders.tsx` | Change Request on confirmed bookings with a pending invoice. Delete disabled when manager approval or the linked PI locks it. |
| `src/features/enquiries/*` | Change Request on approved ones, Clone on converted or cancelled ones. "Close" and "Closed" now read "Cancel" and "Cancelled". "Convert" reads "Convert Order". |
| `src/features/procurement/*`, `routes/inventory.tsx` | Indent delete and Change Request. Purchase order cancel, reopen and a Cancelled tab. Actions show only when they apply. Labels read "Convert PO" and "Convert Purchase". |
| `src/features/inventory/purchase-return-*`, `components/purchase-return-dialog.tsx`, `capabilities.ts` | Edit, delete, clone and reopen for returns. Reject action and tab removed from the screen. |
| `src/features/inventory/customer-material-tab.tsx` | Empty tab removed. |
| `src/features/final-invoices/*` | Edit, cancel and reopen, with a Cancelled tab. Approve only when pending, e-Invoice only when confirmed. |
| `src/features/invoices/create-proforma-invoice.tsx` | Party outstanding is fetched only on the form step. |
| `src/features/payment-details/*` | Transaction date starts as today. |
| `src/features/dispatch/components/dispatch-table.tsx` | View Details row action removed. |
| `src/routes/work-orders.tsx` | No print actions on a draft work order. |
| `src/components/ui/data-table.tsx` | The Actions column is hidden when no row has an action. |
| `src/lib/messages/*` (new) | Shared success and error messages, plus `getApiReason`. |
| `tests/pi-status-capabilities.test.ts` | Updated for the PI rules. |
| Remaining files under `src` | Messages swapped for the shared ones and formatting, no real change in behaviour intended. |

## Why

Users needed a way to correct approved records with a reviewer in between, and the action menus showed items that did not apply or were missing. Messages were written differently on each screen and some showed technical text.

## How

**Change Request**: one popup and one review list are shared by enquiry, indent, expense and order booking. The menu item is disabled while a request is waiting.

**PI rules**: a pending PI is locked for editing once the party approves it, and a change is requested from the Pending tab. A request is disabled when the work order is not in draft. A PI that had been fully approved before it was reopened shows only Manager Approval as pending.

**Locked records**: approved expenses and indents have no Edit action.

**Error messages**: `getApiReason` shows the server's message when a user can act on it, otherwise a fixed sentence. The original error goes to the console.

## Decisions made

**Expense screens follow `fix/expense-ui`.** Only Change Request and Clone were kept from our side. The read only expense modals and the removal of row Approve were dropped.

**No Delete on approved expenses.** Only drafts show it. The API still allows the delete, so nothing new reaches the Archive from this screen.

## Bugs found while testing (already there before, not fixed here)

**1. Dashboard URL filter test.** `tests/dashboard-filters.test.ts`, "a hand-edited URL cannot break the page", also fails on `origin/dev`.

**2. PI Clone test.** `tests/pi-status-capabilities.test.ts`, "Clone needs Clone and Add (the API allows it through PI Add)", also fails on `origin/dev`.

## Notes

- Needs the backend of this branch and migration `192`.
- Roles other than the admins need the new expense rights before the expense screens show for them.
- Approving a reopened PI does not recalculate its amounts. Save the PI after editing the order booking.
- Cancel is still offered on a PI reopened by a change request.
- The frontend has no test runner, so tests were run with `tsx` from the backend's `node_modules`.

## Next steps

- Update the final invoice and purchase return capability tests to the new rules.
- Decide if Cancel should be disabled on a PI reopened by a change request.

## Test plan

**Automated tests (115 passing, 5 failing, 120 total):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes.
- [ ] `tests/final-invoice-capabilities.test.ts`, "Generate e-Invoice shows only while there is no IRN and the invoice is live": fails here, passes on `dev`. The rule is now confirmed invoices only.
- [ ] `tests/inventory-capabilities.test.ts`, Purchase Return "a return row follows its right AND its status": fails here, passes on `dev`.
- [ ] `tests/inventory-capabilities.test.ts`, Purchase Return "Approve covers Reject (one right), and Approve does not cancel": fails here, passes on `dev`. Reject was removed from the screen.
- [ ] The other 2 failures: also fail on `origin/dev` (see "Bugs found while testing").
- [ ] Lint on all changed files: not confirmed, the run was stopped. Lint on the expense, permission and message files gave no errors and 2 warnings.

**Manual checks (local):**
- [ ] Expense add, edit, approve, change request, approve again, clone, delete and restore. Not done.
- [ ] PI, order booking, enquiry, procurement, return and final invoice actions in the running app. Not done.
