# 2026-10-26 - sales-visit-register (frontend)

## Summary

There was no screen to record sales visits, and the Reports page only had the Sales Representative report. This PR adds a Sales Visits page (list, filters, new and edit form, view dialog), connects an enquiry to a visit, and turns the Enquiry Source dropdown into a table based list. It also changes the Reports page: the old Sales Representative report is commented out in the report list, and two new report shells are added (Sales Visit Register and Sales Rep Performance) that have filters, buttons and an empty table but no data yet.

The code compiles (`tsc -b` passes) and `vite build` finished earlier in the work. I did **not** open the pages in a browser in this session, and the backend migrations were not applied, so the screens were **not checked live**. In particular the empty message position, the title bar download buttons and the GPS capture still need a visual check.

**Report data is not included on purpose** - see "Notes" below. The two report tables are shells until the client confirms the core flow. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/sales-visits/api.ts` (new) | Types and API calls for visits, the customer location lookup and visiting purposes. |
| `src/features/sales-visits/queries.ts` (new) | TanStack Query options and mutations for the list, detail, save, delete, purposes and the visit picker used by the Enquiry form. |
| `src/features/sales-visits/sales-visits-page.tsx` (new) | The Sales Visits page. Four KPI cards (Total Visits, Immediate Orders, Later Orders, Visit Expense), All, Immediate and Later tabs with counts, search, start and end date, sales rep and customer filters, and a table with the Visit ID as a highlighted first column, server side paging and sorting, and a pager pinned to the bottom. Clicking a Visit ID opens a read only details dialog (with an Edit button). The row menu has Edit and Delete. |
| `src/features/sales-visits/sales-visit-form-modal.tsx` (new) | The Log Sales Visit and Edit form. The customer location fills from the party address when a customer is picked and can be corrected. Captures the browser location (latitude, longitude, accuracy) automatically for a new visit and with a Capture button. The Sales Rep, Customer and Visiting Purpose dropdowns each have an add new item. The expense box has no up and down arrows. |
| `src/features/sales-visits/visit-purpose-dialog.tsx` (new) | Small dialog to add a new visiting purpose. |
| `src/routes/sales-visits.tsx` (new) | The `/sales-visits` route, guarded by `createPermissionGuard('sales_visit', 'VIEW')`, with list state kept in the URL. |
| `src/components/layout/nav-items.ts`, `src/features/auth/accessible-routes.ts` | Adds the Sales Visits menu item and the route entry (module `sales_visit`). |
| `src/features/enquiries/api.ts`, `queries.ts` | Adds `visitId` to the enquiry types, and the enquiry source list and create calls and query. |
| `src/features/enquiries/enquiry-form-modal.tsx` | The Enquiry Source dropdown now reads from the database and "Add New Source" saves to it (before, a new source was lost when the form closed). A new Sales Visit dropdown, listing the selected customer's visits, shows only when the source is Visit. Changing to another source clears the visit. |
| `src/features/sales-reps/sales-rep-dialog.tsx` | Added an optional `onCreated` callback so a caller can select the rep that was just created. Behaviour for other callers is the same. |
| `src/features/reports/registry.ts` | The Sales Representative report entry and its import are commented out. Its files under `src/features/reports/sales-rep/` are untouched. Adds Sales Visit Register and Sales Rep Performance. |
| `src/features/reports/reports-page.tsx` | The report picker is a fixed narrow width, and the title bar now has a slot where a report's download buttons are shown. |
| `src/features/reports/components/report-layout.tsx` | New `fill` option so a table report fills the area and keeps its pager at the bottom. The download buttons are drawn into the title bar slot when there is one. |
| `src/features/reports/components/report-actions-slot.tsx` (new) | React context holding the title bar slot element. |
| `src/features/reports/components/report-actions.tsx` (new) | The Export XLSX and Download buttons, disabled when there are no rows. |
| `src/features/reports/components/report-select-filter.tsx` (new) | Reusable dropdown filter with an "All ..." entry. |
| `src/features/reports/components/report-table.tsx` (new) | Reusable basic report table on top of the shared `DataTable`. |
| `src/features/reports/components/report-options.ts` (new) | Hooks that load the sales rep and customer choices from their masters. |
| `src/features/reports/visit-register/visit-register-report.tsx` (new) | Sales Visit Register shell: date range, sales rep, customer and order type filters and the nine columns from the spec. |
| `src/features/reports/performance/sales-rep-performance-report.tsx` (new) | Sales Rep Performance shell: date range and sales rep filters and the ten columns from the spec. |
| `src/features/reports/export-xlsx.ts` (new) | Helper that builds an `.xlsx` file and downloads it. The Excel library loads only when someone exports. Not used by a report yet. |
| `src/components/ui/data-table.tsx` | New optional `emptyClassName` so a report can make the empty message area taller. The default is the old `h-44`. |
| `package.json`, `package-lock.json` | Added `exceljs` for the Excel export. |

## Why

The ask was a Sales Executive Visit Register, a visit link on the enquiry, and then a Report module that shows the register and a performance report with reusable filters, buttons and tables. The easy part was copying the Enquiries layout. The real work was making the dropdowns database driven with add new items, and building the report pieces so the next report only has to pass columns and filters.

## How

**Sales Visits page**: Follows the Enquiries page layout and uses the same shared parts (`PageHeader`, `KpiCard`, `FilterTabs`, `FilterToolbar`, `SearchBar`, `DatePicker`, `OptionSelect`, `DataTable`, `CodeLinkButton`, `ViewDetailsDialog`). The Sales Order No. column is filled by the backend from the visit's enquiries.

**GPS**: Uses the browser location API on the device that logs the visit. If the user blocks it, the form shows a message and still saves without a position.

**Visit field on the Enquiry form**: The visits are only requested when a customer is chosen and the source is Visit.

**Report pieces**: A report passes its columns, rows and filter controls to `ReportLayout`, `ReportTable`, `ReportSelectFilter` and `ReportActions`. The download buttons are drawn into the page title bar through a portal, so a report does not need to know where the title bar is.

## Decisions made

**Old report commented out, not deleted.** It can be switched back on by uncommenting the registry entry.

**Excel export uses `exceljs`, loaded on demand.** It was added after asking, and it keeps the main bundle small.

**Reports are shells.** The buttons are placeholders (they do nothing yet) and the tables are empty, as asked, until the core flow is agreed.

**Customer Location shows plain text.** The map pin icon that linked to the GPS position was removed from the list on request. The position is still saved and shown as a Google Maps link in the details dialog.

## Notes

- The Sales Rep Performance report, the report data and the download handlers wait for the client's rules on order attribution and the conversion period.
- During the report UI rounds, a month picker, a status dropdown on both reports, and a centred empty message overlay were tried, but those changes are **not in the final code**. The reports now have only a date range and the dropdown filters listed above, and the empty message is a taller table cell. If those are still wanted they need to be added again.
- The backend migrations (087 to 090) and the seed must be run before the Sales Visits page and the new Enquiry source list work.
- Mobile (installed app) pages for Sales Visits were not built.

## Next steps

- Check the reports page in the browser (title bar buttons, empty message position, pager at the bottom).
- Connect the Visit Register and Performance reports to real data and wire the Export XLSX and Download buttons once the core flow is told.

## Test plan

**Automated tests (no new tests added):**
- [x] `npx tsc -b`: passes with no errors.
- [x] `vite build` (output sent to a temp folder): finished earlier in the work, before the last edits. It also regenerates the route tree for the new `/sales-visits` route.
- [ ] No component or end to end test was added for the new screens.

**Manual checks (local):**
- [ ] Log a visit with GPS, edit it, view it from the Visit ID, and delete it - not done. The pages were not opened in a browser in this session.
- [ ] Add a new sales rep, customer and visiting purpose from the form dropdowns and see each one selected - not done.
- [ ] Set an enquiry source to Visit, pick a visit, convert the enquiry, and check the order number appears on the visit - not done. Needs the backend migrations applied.
- [ ] Open Reports, switch between the two new reports, and check the filters, the title bar buttons and the empty message - not done.
