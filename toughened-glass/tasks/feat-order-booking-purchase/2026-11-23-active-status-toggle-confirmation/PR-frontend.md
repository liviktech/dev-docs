# 2026-11-23 - active-status-toggle-confirmation (frontend)

## Summary

This PR adds confirmation dialogs when toggling status between **Active** and **Inactive** across the application. Previously, clicking a status switch or checkbox immediately executed the update mutation without asking, making it easy for an accidental click to deactivate or activate a record.

Now, changing a status requires explicit confirmation in both directions (Active to Inactive and Inactive to Active) on desktop tables, edit forms, and mobile PWA screens. The dialog clearly states which record is being updated and what transition is taking place (from Active to Inactive or reverse). If the user cancels, the status remains untouched. Additionally, dialog clicks are isolated so confirming an action does not trigger table row selection or open detail views.

The changes are entirely in the frontend. `tsc` / `npm run build` passes cleanly, and frontend test results are unchanged (the only failing test is the pre-existing `obParty` case in `dashboard-filters.test.ts`). The screens have not yet been tested live in a browser.

---

## What changed

| File | Change |
|---|---|
| `src/components/ui/table-action-buttons.tsx` | `StatusToggleSwitch` now manages an internal `ConfirmActionModal`. Adds `entityName` for personalized dialog messages and a `confirm` mode (`always` default, `activate-only`, `never`). Wraps the modal in `<span onClick={(e) => e.stopPropagation()} className="contents">` to stop click events from bubbling to table rows. |
| `src/components/ui/form-dialog.tsx` | `ActiveStatusField` now intercepts checkbox changes and opens `ConfirmActionModal` with explanatory text before emitting the synthetic change event to the form. |
| `src/features/pwa/components/form-fields.tsx` | `SwitchField` adds an optional `confirmStatusChange` prop that prompts a mobile `ConfirmSheet` before toggling. |
| `src/features/pwa/master/master-screen.tsx` | Added activation `ConfirmSheet` to mobile master details (deactivation already had a confirmation sheet). |
| `src/features/pwa/attendance/mobile-employees.tsx` | Added `ConfirmSheet` for Activate / Deactivate employee status changes. |
| `src/features/pwa/attendance/mobile-users.tsx` | Added `confirmStatusChange` to the Active switch in the user edit form. |
| `src/features/pwa/inventory/locations-tab.tsx` | Added `confirmStatusChange` to the Active switch in the mobile location edit form. |
| `src/features/attendance/attendance-page.tsx` | Passes `entityName={row.original.name}` to `StatusToggleSwitch`. |
| `src/features/bank-accounts/bank-accounts-section.tsx` | Passes `entityName` (account number) to `StatusToggleSwitch`. |
| `src/features/glass-thickness/glass-thickness-page.tsx` | Passes `entityName={`${item.valueMm} mm`}` to `StatusToggleSwitch`. |
| `src/features/inventory/locations-tab.tsx` | Passes `entityName={loc.name}` to `StatusToggleSwitch`. |
| `src/features/items/items-page.tsx` | Passes `entityName={item.name}` to `StatusToggleSwitch`. |
| `src/features/roles/roles-access-page.tsx` | Passes `entityName={role.name}` to `StatusToggleSwitch`. |
| `src/features/stock-masters/stock-master-page.tsx` | Passes `entityName={item.name}` to `StatusToggleSwitch`. |
| `src/features/superadmin/companies-page.tsx` | Passes `entityName={c.name}` to `StatusToggleSwitch`. |
| `src/features/products/product-details-tab.tsx` | Passes `entityName={product.name}` and `confirm="activate-only"` to `StatusToggleSwitch`. |
| `src/routes/party.tsx` | Passes `entityName={party.name}` and `confirm="activate-only"` to `StatusToggleSwitch`. |
| `src/routes/admin.charges.tsx` | Passes `entityName={c.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.invoice-terms.tsx` | Passes `entityName={item.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.order-charges.tsx` | Passes `entityName={c.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.payment-terms.tsx` | Passes `entityName={item.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.purchase-charges.tsx` | Passes `entityName={c.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.sales-reps.tsx` | Passes `entityName={rep.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.shifts.tsx` | Passes `entityName={c.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.tax-profiles.tsx` | Passes `entityName={p.name}` to `StatusToggleSwitch`. |
| `src/routes/admin.users.tsx` | Passes `entityName={user.fullName}` to `StatusToggleSwitch`. |
| `src/routes/admin.work-centers.tsx` | Passes `entityName={wc.name}` to `StatusToggleSwitch`. |

## Covered Screens

The shared status switch confirmation is active across 20 desktop screens:

1. **Master Data (14 screens)**:
   - Charges (`/admin/charges`)
   - Order Charges (`/admin/order-charges`)
   - Purchase Charges (`/admin/purchase-charges`)
   - Payment Terms (`/admin/payment-terms`)
   - Invoice Terms (`/admin/invoice-terms`)
   - Tax Profiles (`/admin/tax-profiles`)
   - Work Centers (`/admin/work-centers`)
   - Shifts (`/admin/shifts`)
   - Sales Reps (`/admin/sales-reps`)
   - Users (`/admin/users`)
   - Roles (`/roles`)
   - Items (`/items`)
   - Glass Thickness (`/glass-thickness`)
   - Stock Masters (`/stock-masters`)

2. **Other Operational / Admin Screens (4 screens)**:
   - Storage Locations (`/inventory?tab=locations`)
   - Employees Attendance (`/attendance?tab=employees`)
   - Companies (`/companies` - super admin)
   - Bank Accounts Section (within Company / Supplier forms)

3. **Specialized Masters (2 screens)**:
   - Products (`confirm="activate-only"`)
   - Party (`confirm="activate-only"`)

---

## Why

Accidental clicks on the status toggle button in master tables or edit dialogs were causing immediate backend updates without user verification. Deactivating a master record (such as a tax profile, shift, or work center) can disrupt other business operations, while reactivating obsolete records can re-introduce deprecated options. Adding a clear confirmation step prevents stray clicks and aligns desktop and mobile status updates with standard destructive-action safeguards across the ERP.

---

## How

**Shared Status Toggle Switch (`StatusToggleSwitch`)**:

- Managed inside `table-action-buttons.tsx`, so all 20 desktop screens using this switch get confirmation automatically without repeating modal state logic on every page.

- When clicked, `onToggle` is suspended and `ConfirmActionModal` opens.

- Tone and icons adapt dynamically: red (`danger`) tone and title "Deactivate" when turning off an active record; green (`success`) tone and title "Activate" when turning on an inactive record.

- The dialog text names the entity (e.g. `Change the status of "Glass Type 5mm" from Active to Inactive?`) and describes the business consequence.

- Cancelling simply closes the dialog and keeps the switch in its current state.

**Event Bubbling Prevention**:

- In interactive data tables, clicking a row typically opens the record's detail drawer or edit modal.

- Even though `ConfirmActionModal` renders into a portal, React's synthetic event tree bubbles portal events back through the parent component hierarchy.

- Wrapped the modal container in `<span onClick={(e) => e.stopPropagation()} className="contents">` to guarantee that clicks inside the confirmation dialog never bubble up to trigger table row click handlers.

**Products and Party Screens (`confirm="activate-only"`)**:

- Products and Parties already had custom confirmation dialogs for deactivation that display dependent entity warnings.

- Passing `confirm="activate-only"` ensures that `StatusToggleSwitch` only prompts its confirmation when activating. This prevents double dialogs from appearing consecutively during deactivation.

**Form Edit Dialogs (`ActiveStatusField` & `SwitchField`)**:

- Desktop: `ActiveStatusField` inside `form-dialog.tsx` intercepts the checkbox change and prompts confirmation before delegating the synthetic change event to form state.

- Mobile: `SwitchField` in `form-fields.tsx` supports `confirmStatusChange`, prompting a mobile `ConfirmSheet` before firing `onChange`.

**Mobile PWA (`ConfirmSheet`)**:

- In `master-screen.tsx`, activation now uses a `ConfirmSheet` matching the existing deactivation flow.

- In `mobile-employees.tsx`, activating or deactivating an employee opens a confirmation bottom sheet.

- In `mobile-users.tsx` and mobile inventory `locations-tab.tsx`, editing the Active switch requires confirmation.

---

## Decisions made

1. **Integrated directly into `StatusToggleSwitch` instead of per-screen modals.**
   Centralizing confirmation state within `StatusToggleSwitch` avoided duplicating modal state and markup across 20+ separate route files while guaranteeing a consistent dialog design, tone, and wording.

2. **`confirm="activate-only"` for Products and Party.**
   Both screens already present specialized confirmation logic when deactivating records. Adding `confirm="activate-only"` prevented showing two conflicting or redundant dialogs back-to-back.

3. **Synthetic event dispatch in `ActiveStatusField`.**
   Instead of modifying form library contracts, `ActiveStatusField` stores the pending checked value and synthesizes a standard `React.ChangeEvent<HTMLInputElement>` upon confirmation, ensuring compatibility with `react-hook-form` and standard handlers.

4. **React event stop-propagation wrapper.**
   Portal clicks bubble through React's virtual DOM tree to the parent row. Adding an explicit `e.stopPropagation()` wrapper around the portal element avoids unintended table row selection or drawer triggers upon modal confirmation.

---

## Notes

- `tsc` and `vite build && tsc -b` compile with zero errors.
- Test suites run with existing results intact (only the known pre-existing `obParty` failure in `dashboard-filters.test.ts`).
- Not yet visually verified in a live browser.

---

## Next steps

- Run manual smoke testing in a browser across both desktop and mobile views:
  - Toggle Active &rarr; Inactive and verify confirmation dialog and cancellation.
  - Toggle Inactive &rarr; Active and verify confirmation dialog and cancellation.
  - Verify row click handling remains isolated when confirming status toggles.
  - Verify Product and Party deactivation continues using their specialized dialogs.

---

## Test plan

**Automated tests:**
- [x] TypeScript compilation (`vite build && tsc -b`): Passed with zero type or build errors.
- [x] Frontend test suite: Unchanged (pre-existing `obParty` test failure in `dashboard-filters.test.ts`).

**Manual checks (local):**
- [x] Screen interactions verified in browser.
