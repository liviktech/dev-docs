# 2026-11-20 - fir-and-reports-rbac-reports-menu-and-inventory-location-status (frontend)

## Summary

This PR has four parts: role-based access for the FIR module, role-based access for the Reports section, a reshuffle of the Reports menu, and one small Inventory fix.

Before this PR, FIR used the old module-wide rights, where Add and Edit quietly counted as View and a right from another module could open the register. Reports had only two rights for all 20 reports: one View that opened every report, and one Download that was only checked by greying out buttons on the screen. The Reports menu had Work Order Progress hidden inside Production Plan, and the Order Status report was called "Order Status Register". On the Inventory Locations tab, the active/inactive switch showed greyed out to users without the right, and it checked Edit even though turning a location off needs Delete on the server.

What this PR changes:
- **FIR** now has 4 exact rights (View, Add, Edit, Delete), built the same way as Enquiry. Add, Edit and Delete each need View. Nothing is implied any more.
- **Reports** now has 30 exact rights: View, Export (Excel) and Download (PDF) for each of the 10 sections of the Reports menu. A section's View opens every report in that section. Export and Download each need that section's View. The server checks View on every report route, and checks Export or Download when a request asks for every row at once.
- **Reports menu:** Work Order Progress is its own entry, Sales Income sits right after Production Plan, and "Order Status Register" is now "Order Status".
- **Buttons without a right are hidden, not greyed out,** in Reports and on the Inventory Locations tab. I also checked the rest of Inventory (desktop, mobile and the Indent and Purchase Order tabs): every other control already worked this way.

Existing roles keep their access. Two migrations per module map old role data to the new rights: 162 and 163 for FIR, 164 and 165 for Reports. They were run on a throwaway local Postgres with sample roles. **They are not applied to any shared database, and nothing is committed.**

The screens were not checked in a browser. See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/features/auth/permissions.ts` | The 4 FIR rights and the 30 Reports rights, by their stored codes. |
| `src/features/fir/capabilities.ts` (new) | `getFirCapabilities` (view, add, edit, delete) and `canViewFirRights` for code that runs before the auth context exists. |
| `src/features/fir/fir-page.tsx` | "New FIR" shows only with Add. The row menu offers Edit and Delete only with their rights, and is not shown when the user has neither. |
| `src/routes/fir.tsx` | The route guard now needs `fir_view`. |
| `src/features/pwa/fir/mobile-fir-list.tsx`, `mobile-fir-detail.tsx` | Use the FIR capabilities for the New, Edit and Delete controls. |
| `src/routes/m.fir.tsx`, `m.fir_.$firId.tsx`, `m.fir_.new.tsx`, `m.fir_.$firId_.edit.tsx` | Mobile FIR routes need View; New also needs Add, and Edit also needs Edit. |
| `src/features/auth/auth-context.tsx` | `fir` removed from the "delete needs its own right" list. Nothing asks the module-level FIR check any more. |
| `src/features/reports/capabilities.ts` (new) | The rights of each Reports section, `getReportCapabilities` (view, export, download), `canViewAnyReport`, `canViewReportRights` and `viewableReportSections`. |
| `src/features/reports/components/report-section-context.tsx` (new) | Tells the download buttons which Reports section is open, and gives them that section's rights. |
| `src/features/reports/components/report-export-button.tsx` | The button takes a `format`: Excel needs Export, PDF needs Download. Without the right it is hidden. |
| `src/features/reports/components/report-actions.tsx`, `src/features/reports/sales-rep/sales-rep-report.tsx` | Pass the format to each button. |
| `src/features/reports/reports-page.tsx` | Lists only the sections the user can view, falls back to the first allowed one, and shows "No access" when there is none. Provides the open section to the buttons. |
| `src/routes/reports.tsx` | The route opens for a user who holds View of at least one section. |
| `src/components/layout/app-sidebar.tsx` | The Reports sub-menu lists only the sections the user can view. |
| `src/components/layout/nav-items.ts` | The FIR entry needs `fir_view`. The Reports entry shows if any section View is held. |
| `src/features/auth/accessible-routes.ts`, `src/routes/m.index.tsx` | Start-page redirects use the new FIR and Reports rules. |
| `src/routes/m.reports.tsx`, `src/routes/m.reports_.$group.$view.tsx` | Mobile Reports routes check the section's View. |
| `src/features/pwa/reports/mobile-reports-home.tsx` | Lists only the allowed sections. |
| `src/features/pwa/reports/mobile-report-view.tsx` | Provides the open section to the toolbar. |
| `src/features/pwa/reports/report-ui.tsx` | Export XLSX and Download PDF are separate buttons with their own rights, hidden without them. The old "You need the Reports Download right" banner is removed. |
| `src/features/roles/dependencies.ts` | Static fallback for the FIR and Reports dependencies (Add, Edit, Delete, Export and Download need View). |
| `src/features/roles/rights-picker.tsx` | A heading for each of the 10 Reports sections in the Roles dialog. |
| `src/features/reports/report-menu.ts` | Work Order Progress is its own entry. Sales Income moved right after Production Plan. "Order Status Register" is now "Order Status". |
| `src/features/reports/registry.ts`, `src/features/pwa/reports/registry.ts` | The Work Order Progress key moved with its entry. |
| `src/features/pwa/reports/order-status-report.tsx`, `src/features/pwa/reports/work-order-progress-report.tsx` | Mobile report titles follow the new names. |
| `src/features/inventory/locations-tab.tsx` | The status switch shows only to a user who can make the change it offers (Edit activates, Delete deactivates). Everyone else sees plain "Active" or "Inactive" text. |
| `tests/fir-capabilities.test.ts` (new) | The FIR flags. |
| `tests/report-capabilities.test.ts` (new) | The Reports flags, section lists and start-page rules. |
| `tests/roles-dependencies.test.ts` | New cases for the FIR and Reports dependencies. |

## Why

The owner asked for FIR to follow the same role-based access structure as Inventory, Order Booking, Enquiry, PI Status and Final Invoice, and then for the same on Reports. The menu changes and the hidden buttons came from review feedback during the same day.

FIR was the easy one: it has one screen and no tabs, so it follows Enquiry. The real work was the role data. The old rules let Add and Edit open the register, so the migration has to give those holders View, or they lose access. Reports was harder. The owner wanted rights per section instead of per report (30 rights instead of 63), and wanted Export and Download as separate rights. The real work there was finding what the server can and cannot enforce, because every report builds its file in the browser.

## How

**FIR rights**: four exact rights, one lookup each. Add, Edit and Delete depend on View. The Roles dialog ticks View with them, and the server refuses to save a role that leaves it out. Super Admin and Admin pass every check.

**Reports rights**: codes are `reports_<section>_<view|export|download>`. The 10 sections are Sales Rep, Production Plan, Sales Income, Work Order Progress, Department Daily, Price List, Order Status, FIR, Enquiry and Raw Glass. Sales Rep Performance's two tabs have no rights of their own. They fall under Sales Rep.

**What the server enforces for Reports**: View on every report route. Every report list returns all rows when `page` is left out, which is how the Export and Download buttons fetch, and one page of at most 200 rows otherwise. So a request without `page` also needs the section's Export or Download. The server cannot tell Excel from PDF on that request, so either right passes there. The buttons still check their own right on the screen.

**Shared endpoints**: the Production Plan filter lists and the work order list also feed the Work Order Progress dropdown, so they accept either section's View.

**Hidden, not disabled**: a button without its right is not rendered. A button is disabled only when the user holds the right but the record's state blocks it, which is how Inventory already worked.

**Kept the same on purpose**: the FIR right codes did not change (`fir_view` and the rest), so Order Booking and Final Invoice shared-read lists that name `fir` View keep working. The legacy shared rule in `check_role_access.ts` was not touched.

## Decisions made

**FIR has four rights and no more.** There is no status workflow, preview, print or export on FIR, so nothing else would be enforced. I did not add a right that nothing checks.

**FIR stays in the shared legacy "flow modules" rule.** FIR users open PI and Work Order records through it, and those reads still accept a legacy View. Removing `fir` from it now would break those links. It can go when PI Status and Work Order drop their legacy reads.

**Reports rights are per section, not per report** (owner's choice). A user cannot get one report of a section without the others. Sales Income stays one section (owner's choice).

**Export and Download are two rights** (owner's choice): Excel and PDF.

**Existing roles are mapped, not reset** (owner's choice, after first picking reset and then changing to mapping).

**A Download-only role gets nothing in the Reports migration.** The old Download never opened a report by itself, and the new Export and Download need View. Those assignments stay in the backup table. I did not give them View, because that would invent a grant.

**The FIR report and the Enquiry Follow-up report now follow their own Reports rights** (`reports_fir_view`, `reports_enquiry_view`). This replaces the open question of whether to add a separate FIR report right.

**The old unused Sales Rep report screen is not removed.** Only its server check moved to the new rights.

**Sales Income was placed directly after Production Plan, as asked.** That puts Work Order Progress, which was split out of Production Plan the same day, after it, not next to Production Plan.

**The Locations switch uses the right the server needs.** Turning a location off needs Delete on the server and turning it on needs Edit, so the screen now asks for the same. Before, it asked for Edit for both.

## Bugs found while testing (already there before, not fixed here)

**1. A frontend test imports a file that does not exist.** `tests/dispatch-plan-capabilities.test.ts` needs `features/dispatch-plan/capabilities.ts`. It failed before this PR and is the only failing frontend test.

**2. Existing lint errors.** `fir-form-modal.tsx` and the FIR route files had lint errors before. In `inventory/locations-tab.tsx` a `react-hooks/set-state-in-effect` error on an effect around line 80 was there before this PR.

**3. Other screens still show a disabled switch without the right.** The same status switch in Items, Glass Thickness and similar pages is greyed out for users without the right. I only changed Inventory.

## Notes

- For Reports, the sample check showed: a View-only role got the 10 Views; a View plus Download role got 10 Views, 10 Exports and 10 Downloads; a Download-only role got nothing; an unrelated role and Admin were untouched; a user whose View comes from their role got Export and Download; no dependency was left broken.
- The Enquiry follow-up bell reads from `/enquiries/follow-ups`, not from a report route, so it is not affected by the Reports rights.

## Next steps

- Check the screens by hand (see the manual list below), then apply 162 to 165 to the shared database, once the role mapping is confirmed.
- Trim roles in Roles & Access after the Reports cutover: every old Reports View holder now sees all 10 sections.
- Decide when to remove `fir` from the shared flow rule.
- Fix the missing dispatch-plan files behind the failing tests.
- Decide whether to hide the disabled-switch pattern in Items and Glass Thickness too.

## Test plan

**Automated tests:**
- [x] `tests/fir-capabilities.test.ts`, `tests/report-capabilities.test.ts`, `tests/roles-dependencies.test.ts`: the flags, the section lists and the Roles dialog dependency ticking.
- [x] Backend type check (`npx tsc --noEmit`) and frontend type check (`npx tsc -b`): no errors.
- [ ] `tests/dispatch-plan-capabilities.test.ts`: fails, missing file (see "Bugs found").

**Manual checks (local):**
- [ ] FIR in a browser, not done. To check: View only (no New FIR button, no row menu); View and Edit (Edit only); View and Delete (Delete only); a role holding only `pi_status_view` (no FIR in the menu).
- [ ] Reports in a browser, not done. To check: a role with only Sales Income View sees one menu entry and no Export or Download button; View and Export shows only the Excel button; View and Download shows only the PDF button; a Raw Glass-only role opening the Sales Income link falls back to its own section; the mobile list shows only allowed sections.
- [ ] Reports menu order and names in a browser, not done. To check: Work Order Progress is its own entry, Sales Income follows Production Plan, "Order Status" is the new name.
- [ ] Inventory Locations status switch in a browser, not done. To check: a user with Edit only sees a switch on inactive locations and plain "Active" text on active ones.
