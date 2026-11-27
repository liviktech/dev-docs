# 2026-11-20 - fir-and-reports-rbac-reports-menu-and-inventory-location-status (backend)

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
| `src/shared/rbac/permissions.ts` | The `FIR` and `REPORTS` modules, the 10 Reports groups, and the 4 FIR and 30 Reports rights in the catalog. |
| `src/modules/fir/fir.routes.ts` | All 8 routes ask for one exact FIR right instead of the old derived check. |
| `src/modules/report/report-rights.ts` (new) | The 10 sections and their rights, `requireReportView` and `requireReportRows`. |
| `src/modules/report/report.routes.ts` | Every route names its right itself (there is no router-wide check). The sub-routers are listed in `REPORT_MOUNTS` so a test can walk them. |
| `src/modules/report/sales-income/sales-income-report.routes.ts` | Same, for the five Sales Income reports. |
| `src/middlewares/permission.middleware.ts` | `RbacMeta` gets `fullRowsOnly`, so the "every row" check can be told apart from the main right in tests. |
| `src/database/migrations/162_fir_rights_catalog.sql` (new) | Registers the 4 FIR rights with their group, action, order and dependencies. Changes nothing for anyone yet. |
| `src/database/migrations/163_fir_rbac_cutover.sql` (new) | Backs up FIR assignments, maps old holders to the new rights, switches the catalog on and retires `fir_control`. |
| `src/database/migrations/164_reports_rights_catalog.sql` (new) | Registers the 30 Reports rights, inactive. |
| `src/database/migrations/165_reports_rbac_cutover.sql` (new) | Backs up Reports assignments, maps old holders, switches the catalog on and retires `reports_view` and `reports_download`. |
| `tests/fir-route-rights.test.ts` (new) | The FIR route table, 200 and 403 per right, nothing implied, catalog shape and migration 163 shape. |
| `tests/report-route-rights.test.ts` (new) | The Reports route table, 200 and 403 per right, the every-row rule, catalog shape and migration 165 shape. |
| `tests/rbac-permission-catalog.test.ts` | Adds migrations 162 and 164 to the list it checks against the catalog, and the screen-only Reports rights. |

## Why

The owner asked for FIR to follow the same role-based access structure as Inventory, Order Booking, Enquiry, PI Status and Final Invoice, and then for the same on Reports. The menu changes and the hidden buttons came from review feedback during the same day.

FIR was the easy one: it has one screen and no tabs, so it follows Enquiry. The real work was the role data. The old rules let Add and Edit open the register, so the migration has to give those holders View, or they lose access. Reports was harder. The owner wanted rights per section instead of per report (30 rights instead of 63), and wanted Export and Download as separate rights. The real work there was finding what the server can and cannot enforce, because every report builds its file in the browser.

## How

**FIR rights**: four exact rights, one lookup each. Add, Edit and Delete depend on View. The Roles dialog ticks View with them, and the server refuses to save a role that leaves it out. Super Admin and Admin pass every check.

**Shared endpoints**: the Production Plan filter lists and the work order list also feed the Work Order Progress dropdown, so they accept either section's View.

**Role data**: both cutover migrations back up every assignment first, then map. FIR: Add, Edit and Delete holders also get View, and the old Control profile becomes View, Add and Edit. Reports: the old View becomes View on all 10 sections, and the old Download, for someone who can also view, becomes Export and Download on all 10 sections. Super Admin and Admin are skipped.

**Kept the same on purpose**: the FIR right codes did not change (`fir_view` and the rest), so Order Booking and Final Invoice shared-read lists that name `fir` View keep working. The legacy shared rule in `check_role_access.ts` was not touched.

## Decisions made

**FIR has four rights and no more.** There is no status workflow, preview, print or export on FIR, so nothing else would be enforced. I did not add a right that nothing checks.

**FIR stays in the shared legacy "flow modules" rule.** FIR users open PI and Work Order records through it, and those reads still accept a legacy View. Removing `fir` from it now would break those links. It can go when PI Status and Work Order drop their legacy reads.

**Reports rights are per section, not per report** (owner's choice). A user cannot get one report of a section without the others. Sales Income stays one section (owner's choice).

**Export and Download are two rights** (owner's choice): Excel and PDF.

**Existing roles are mapped, not reset** (owner's choice, after first picking reset and then changing to mapping).

**A Download-only role gets nothing in the Reports migration.** The old Download never opened a report by itself, and the new Export and Download need View. Those assignments stay in the backup table. I did not give them View, because that would invent a grant.

**Export and Download are only partly enforced by the server.** They are marked screen-only for Sales Rep, Work Order Progress and Department Daily, which always answer with their whole result, so a download cannot be told from a view. A user with View can still read all rows by paging, as the screen does. Making this complete means a separate download endpoint per report.

**The old unused Sales Rep report screen is not removed.** Only its server check moved to the new rights.

**Sales Income was placed directly after Production Plan, as asked.** That puts Work Order Progress, which was split out of Production Plan the same day, after it, not next to Production Plan.

## Bugs found while testing (already there before, not fixed here)

**1. Seven backend tests fail in areas this PR does not touch.** `list-paging-schemas.test.ts` imports `modules/dispatch-plan/dispatch-plan.schema.js`, which does not exist. Two purchase schema tests fail with "Item is required". Four PI reminder slab tests fail on the default slabs.

## Notes

- Migrations 162, 163, 164 and 165 were run on an empty throwaway Postgres built from every migration. 163 and 165 were also run on a copy with sample roles and users, then rolled back, and the database was deleted.
- For Reports, the sample check showed: a View-only role got the 10 Views; a View plus Download role got 10 Views, 10 Exports and 10 Downloads; a Download-only role got nothing; an unrelated role and Admin were untouched; a user whose View comes from their role got Export and Download; no dependency was left broken.
- The backend `.env` points at the shared Neon database. Nothing was written there. One read-only query checked which migrations were applied (162 and 163 were free then).
- The repository is not a git repository at the top level, so nothing is committed.
- Deploy the new backend and frontend together with the cutover migrations. The old frontend asks for the old derived right names.

## Next steps

- Trim roles in Roles & Access after the Reports cutover: every old Reports View holder now sees all 10 sections.
- Decide when to remove `fir` from the shared flow rule.
- Decide whether to give each report its own download endpoint, so Export and Download are enforced for every report.
- Fix the missing dispatch-plan files behind the failing tests.

## Test plan

**Automated tests:**
- [x] `tests/fir-route-rights.test.ts`: every FIR route needs its exact right; 200 with the right and 403 without for each of the four; Add, Edit and Delete do not give View; Control and other modules' rights open nothing; admins pass; migration 163 maps correctly.
- [x] `tests/report-route-rights.test.ts`: every report route is in the table with its right; each section's View opens that section and no other; Export and Download do not give View; the old `reports_view` and `reports_download` open nothing; the work order list accepts both Production Plan and Work Order Progress; a request with no page is refused with View alone and allowed with Export or Download; catalog has 30 rights; migration 165 maps correctly.
- [x] `tests/rbac-permission-catalog.test.ts`, `rbac-admin-gate.test.ts`, `enquiry-route-rights.test.ts`: still pass with the new catalog and migrations 162 and 164.
- [ ] Full backend suite: 840 tests, 814 passing, 7 failing in unrelated areas (see "Bugs found").
- [ ] `tests/dispatch-plan-capabilities.test.ts`: fails, missing file (see "Bugs found").

**Manual checks (local):**
- [x] Migrations 162 to 165 on a throwaway Postgres with sample roles and users: results as described in Notes. Sample data rolled back; database deleted.
- [ ] The API called as a real non-admin user, not done. The route tests cover the checks with fake users instead.
