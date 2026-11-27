# 2026-10-31 - sales-rep-employee-sequences (frontend)

## Summary

Sales reps were typed in by hand, with a hand-typed rep code, and the Employee and Sales Rep numbers had no place in Master Data, Sequences. This PR makes the Add Sales Rep dialog pick a rep from the employees who hold the Sales Rep role (email and phone fill in on their own), shows the rep code as auto-generated, adds Employee and Sales Rep to the Sequences dialog, and points the Sales Reps screen at its own right. It also fixes the role edit dialog saving rights under a stale code after a rename.

Type check passes (`tsc --noEmit -p tsconfig.app.json`). The screens were **not** clicked through in a browser, see the test plan.

## What changed

| File | Change |
|---|---|
| `src/features/sales-reps/sales-rep-dialog.tsx` | Full Name is now a searchable dropdown of active employees with the Sales Rep role (only when adding). Picking one fills name, email and phone. Rep Code is a disabled "Auto-generated on save" field and is no longer sent on save. Editing a rep keeps a plain name field. |
| `src/features/sales-reps/types.ts` | `CreateSalesRepInput.code` is optional. |
| `src/features/attendance/types.ts`, `src/features/attendance/api.ts` | New `salesRepOnly` option on the employee list request, sent as `salesRepOnly=true`. |
| `src/features/sequences/types.ts`, `src/features/sequences/sequence-dialog.tsx` | `EMPLOYEE` and `SALES_REP` added to the module list. Both suggest start number 1 and date format NONE when chosen. |
| `src/routes/admin.sales-reps.tsx` | Guard and control check use the new `sales_rep` right. After adding a rep the list jumps to the page the new rep lands on (the list is in code order, so a new rep is last). |
| `src/components/layout/nav-items.ts`, `src/features/auth/accessible-routes.ts` | Sales Reps item uses `sales_rep` instead of `master_sales_rep` (which was folded into Master Data). |
| `src/features/roles/role-dialog.tsx` | After saving a role, the rights are saved under the role code that came back from the update, so a renamed role no longer saves its rights under the old code. |

## Why

Reps and employees are the same people, so the rep form should start from the employee instead of retyping their details. Codes should come from the one place the business already configures numbers (Master Data, Sequences). The Sales Reps screen needed its own right so it can be given to the Sales Rep role without handing over all of Master Data.

## How

**Rep dropdown**: reuses the existing employee list query with `salesRepOnly` and `status: ACTIVE`, and the shared `OptionSelect`. The query only runs while the dialog is open for a new rep.

**Rep code**: the field is only a display. The server numbers the rep from the Sales Rep sequence.

**Sequences dialog**: no new screen. The two modules are just new options in the existing list, so they get the same create, edit and preview behaviour as the rest.

## Decisions made

**Name stays editable when editing a rep.** Only new reps are picked from employees. Older reps may not have an employee record, so locking the name would block edits.

**List is in code order.** I first made it newest first so a new rep shows on page 1, then switched back to code order when ascending order was asked for. The jump to the right page after adding covers the visibility problem.

## Notes

- Email and phone stay editable after the autofill.
- The Sales Reps nav item only shows once the `sales_rep` right exists (needs `npm run seed` in the backend).

## Test plan

**Automated tests (no test files cover these screens, none added):**
- [x] `tsc --noEmit -p tsconfig.app.json`: no errors.

**Manual checks (local):**
- [ ] Add Sales Rep, pick an employee, check email and phone fill in, save, check the code and that the list lands on the right page. Not run in a browser.
- [ ] Master Data, Sequences, add Employee and Sales Rep, check the preview shows `EMP-001` and `SR-001` style codes. Not run in a browser.
- [ ] Edit a role, rename its code, save, check the rights stay ticked. Not run in a browser.
