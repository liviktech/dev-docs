# 2026-10-23 - sales-rep-report-module (frontend)

## Summary

The app had no Reports area. This PR adds a Reports page with the first report, the Sales Representative report. It opens on the common report for every sales rep, and picking one rep narrows everything to that rep. The page is one filter bar across the top and charts below: a sales pipeline bar chart, an enquiry trend line chart, donut charts for enquiry, order booking, work order and final invoice status, and (for all reps) a bar chart of enquiries per rep. A Custom button lets you pick the period day to day, week to week, month to month or year to year. The report can be downloaded as CSV or PDF.

The page was run in a real browser (filters, charts, the tooltip, the period popup in all four tabs, a weekly range). Four things were **not checked** after the last edits: the new Week to week layout (a month picker and one range calendar), the download buttons after they were re-enabled, the PDF with the new chart, and a role that has only the Reports right.

## What changed

| File | Change |
|---|---|
| `src/routes/reports.tsx` (new) | The `/reports` route. Search params `rep`, `from`, `to`, `status`, `q`, `by` go through `parseListSearch`. Guarded by `createPermissionGuard('reports','VIEW')`. |
| `src/components/layout/nav-items.ts` | Added the Reports sidebar item (BarChart3, `/reports`, module `reports`). |
| `src/features/auth/accessible-routes.ts` | Added the `/reports` entry. |
| `src/features/reports/reports-page.tsx` (new) | The page. Shows the report picker (inside the filter bar) and the selected report. |
| `src/features/reports/registry.ts` (new) | List of available reports. Adding a report later means adding one entry here. |
| `src/features/reports/sales-rep/sales-rep-report.tsx` (new) | The main screen: filter bar, charts, and the CSV and PDF handlers. |
| `src/features/reports/sales-rep/api.ts`, `queries.ts`, `types.ts` (new) | API calls, TanStack Query hooks and types. The page asks for the summary only. |
| `src/features/reports/sales-rep/status-meta.ts` (new) | Status labels and the chart colours for each status. |
| `src/features/reports/sales-rep/status.tsx` (new) | The status badge built on the shared `StatusLabel`. No longer used by the page, see "Notes". |
| `src/features/reports/components/report-layout.tsx` (new) | The full-width layout: filter bar on top with the download buttons at the right end, content below. |
| `src/features/reports/components/report-period-filter.tsx` (new) | The Custom period button and its popup with the four tabs. |
| `src/features/reports/components/report-pie-chart.tsx` (new) | Recharts donut with the total in the middle and a legend of count and share. |
| `src/features/reports/components/report-charts.tsx` (new) | Recharts bar chart and line chart in a shared titled card. |
| `src/features/reports/components/report-empty-panel.tsx` (new) | The error panel. |
| `src/features/reports/components/report-export-button.tsx` (new) | Download button with a busy spinner and an error toast. |
| `src/features/reports/export-csv.ts` (new) | Builds a UTF-8 CSV with a BOM so Excel opens it correctly, and protects cells that start with `=`, `+`, `-` or `@`. |
| `src/lib/pdf/sales-rep-report-pdf.ts` (new) | Landscape A4 PDF: centred company header, summary cards drawn with `drawModernCard`, a donut chart with legend, and a detail table with a line under each row. A Sales Rep column is added when the report covers every rep. |
| `src/components/ui/date-picker.tsx` | Added an optional `pickMode` (`day` default, `month` or `year`). A month or year picker opens on its own grid and fills the value with the first day of what was chosen. The default is the old behaviour. |
| `src/components/ui/date-range-filter.tsx` | Added optional `fullWidth` and `className` props. Defaults are unchanged. |
| `package.json`, `package-lock.json` | Added `recharts` (as asked, for the charts). |

## Why

The ask was a scalable Report module, Sales Representative report first, built in the existing theme with reusable components. It then went through many UI rounds. The report moved from a table, to cards, to charts: the user wanted a chart view in the style of a sales performance report, with total enquiries, converted, order booking, work order and final invoice shown. The easy part was the charts. The real work was the period filter, the speed of switching filters, and reusing what already exists.

## How

**Common and single rep report**: With no rep chosen (the default) the page shows every rep. Picking a rep narrows all charts to that rep, and the enquiries-per-rep chart is hidden.

**Filters and API calls (same pattern as Enquiries)**: Filters live in the URL through `useUrlListState`. One report request runs and it re-runs only when a value really changes. The company name comes from the signed in user, so there is no extra company request. The request carries `summaryOnly=true`, so the server skips the row list.

**Custom period**: One narrow "Custom" button opens a popup with four tabs. Day to day uses the two date pickers. Month to month and Year to year use two pickers that open straight on the month or year grid. Week to week uses a month picker and one range calendar (two months side by side) where the start day and then the end day are clicked. Whatever the tab, the value is a plain from and to date, so the API did not change. The button shows only the kind of period ("Month to month"), never the dates. The tab also sets how the trend chart groups its points.

**Download buttons**: Both call the unpaged `/export`, so they carry every row. They are enabled when the summary says there is at least one record.

**Reused shared pieces**: `FilterToolbar`, `FilterTabs`, `Popover`, `Calendar`, `DatePicker`, `DateRangeFilter`, `OptionSelect`, `TableLoadingPanel` and `Button`.

## Decisions made

**Charts use Recharts.** The user asked for it, and no chart library was in the app. It was added to `package.json`.

**No row list on the page.** The table and then the cards were removed on request, so the page only draws charts. The rows are still loaded for the CSV and PDF.

**Week to week shows a month picker and one range calendar.** The user asked for a month picker first, then a day picker where two days are chosen. The weekly grouping is only for the trend chart. This was the last thing built and is **not checked in a browser**.

**Section checkboxes, summary cards and the report heading were removed.** The page starts with the charts. The PDF still has its own header and summary cards.

**Kept report-only components.** The layout, error panel, download button, period filter and chart wrappers have no shared equivalent. `export-csv.ts` is also kept. PI Status and Final Invoices each build their own CSV inline, and moving them onto this helper would mean touching those modules, so it was **not** done.

**Enquiry module files were not touched.** They were only used as a reference for the filter and API call pattern.

## Bugs found while testing (already there before, not fixed here)

**1. Lint warning in the shared date picker.** `src/components/ui/date-picker.tsx` has a `react-hooks/set-state-in-effect` warning on the effect that resets the view when the popup opens. It is in the committed version too, and it affects every page that uses the date picker.

## Notes

- `src/features/reports/sales-rep/status.tsx` is not used by anything now. It is left in place in case a record list comes back, and can be deleted if not.
- Switching a filter takes about 0.75 to 1 second. About half a second of that is the shared login and permission check on the server.
- When the trend grouping is changed, the axis labels can show the old values for a moment until the new data arrives.
- Stale editor buffers overwrote edits to the report page a few times during the work. The files were re-applied from disk each time. Please check that the saved files are the final ones before committing.
- Checked with a separate test server and dev server on ports 5199 and 5174, not the usual dev ports.

## Next steps

- Open the Week to week tab and the download buttons in a browser and fix anything that looks off.
- Check the Reports page with a non-admin role that only has the Reports right.
- Add the next reports as entries in `registry.ts`.

## Test plan

**Automated tests:**
- [ ] No automated tests were added for this feature.
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes with no errors.
- [x] `npx eslint src/features/reports`: no problems.

**Manual checks (local):**
- [x] All reps and one rep (Arun Kumar): KPI numbers, pipeline, trend and the four donut charts show the right counts. The by-rep chart shows only for all reps.
- [x] Chart hover tooltip no longer covers the number in the middle of a donut.
- [x] Custom popup: Day to day, Month to month (month grid with year arrows) and Year to year (year grid) each filled the filter. A month range of Sep to Oct 2026 gave `from=2026-09-01&to=2026-10-31`, and 2025 to 2026 gave `from=2025-01-01&to=2026-12-31`.
- [x] Weekly grouping: the trend showed weekly points (28 Sep, 05 Oct, 12 Oct, 19 Oct) and the charts narrowed to the range. This was the earlier version of the Week tab.
- [ ] Week to week as it is now (month picker and one range calendar) - only type checked, not opened in a browser.
- [ ] Export CSV and Download PDF after the buttons were re-enabled, and the PDF with the donut chart - not clicked or opened since the change. An earlier version of both was checked.
- [ ] A non-admin role with only the Reports right - not checked, only the admin user was used.
