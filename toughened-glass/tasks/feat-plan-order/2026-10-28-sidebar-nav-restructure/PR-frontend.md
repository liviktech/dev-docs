# 2026-10-28 - sidebar-nav-restructure (frontend)

## Summary

The sidebar was a long flat list: Master Data listed every screen in one group, Reports had its sub reports as separate menu entries, and Enquiries and Sales Visits were two unrelated places. This PR groups Master Data into clear categories, moves Sales Rep reports into one page with two tabs, makes the menus open one at a time, and puts Enquiries and Sales Visits behind a shared tab strip. All sidebar text now uses the same lighter colour as the Master Data category headings.

Backend is not touched. Only the type check was run (passes). **The screens were not clicked through in the browser**, see "Test plan".

## What changed

| File | Change |
|---|---|
| `src/components/layout/nav-items.ts` | Master Data is now `MASTER_DATA_CATEGORIES`: Partners (Party, Suppliers), Product Details (one direct link), Plant & Operations, Billing & Finance, System (User, Roles & Access), Settings (Company Profile, Sales Reps, Sequences, Notifications). Added a `direct` flag for single page categories. `MASTER_DATA_ITEMS` is still exported, built from the categories |
| `src/components/layout/app-sidebar.tsx` | Renders the categories as accordions with no count badges. Only one category open at a time, and Master Data and Reports close each other. Direct categories render as a plain link. Item text colour changed from `slate-400` to `slate-300`. Enquiries menu item also lights up on Sales Visits pages and falls back to the Sales Visits page if the user can only see that |
| `src/components/layout/pre-sales-nav-tabs.tsx` (new) | Tab strip that switches between Enquiries and Sales Visits. Hidden when the user can only see one of them |
| `src/features/enquiries/enquiries-page.tsx` | Shows `PreSalesNavTabs` under the page header |
| `src/features/sales-visits/sales-visits-page.tsx` | Shows `PreSalesNavTabs` under the page header |
| `src/features/reports/report-menu.ts` | Added `TABBED_REPORT_GROUPS` (currently only `sales-rep`) |
| `src/features/reports/reports-page.tsx` | For a tabbed group, shows the group's reports as tabs (same `ModuleTabs` strip as Product Details) and the page title drops the report name |

## Why

The Master Data list had grown too long to scan, and Glass Types, Glass Colors and Glass Finishes already live as tabs inside the Product Details page, so they no longer need their own menu entries. Sales Rep reports are two views of one topic, so tabs fit better than two sidebar rows.

## How

**Categories**: each category has an id, label, icon and items. Items are still filtered by the user's `VIEW` right for their module, and a category with no visible items is hidden.

**Direct categories**: a category with `direct: true` is shown as a link to its first item instead of an expandable group. Product Details uses this.

**One menu open at a time**: `openCategories` holds at most one id. Opening a category replaces it. Navigating to a Master Data page opens its category and closes Reports. Opening Reports closes Master Data and the other way round. The Reports sub groups already worked this way and were left alone.

**Tabbed report groups**: the sidebar links straight to the group (first report), and the page switches reports through the same `group` and `view` URL params as before, so old links still work. To tab another group later, add its id to `TABBED_REPORT_GROUPS`.

## Decisions made

**Tab list is a separate constant.** `TABBED_REPORT_GROUPS` sits next to `REPORT_MENU` instead of a flag on each group, because `REPORT_MENU` is declared `as const` and adding a field to one group only would break the typing for the others.

**Product Details stays one entry.** Glass Types, Colors and Finishes were not given sidebar entries again. They remain tabs inside the page, as requested.

## Bugs found while testing (already there before, not fixed here)

**1. Type check via `tsc --noEmit -p .` checks nothing.** The root `tsconfig.json` has no files of its own, so it prints no errors even when there are some. Use `tsconfig.app.json`. This was already the case before this PR.

## Notes

- The commit also carries the Enquiries and Sales Visits tab strip, which was done as separate work before the sidebar changes in this PR.
- Old links such as `/admin/glass-colors` and `/admin/glass-finishes` still redirect into the Product Details tabs, nothing changed there.

## Test plan

**Automated tests (type check passing, no unit tests added):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: no errors
- [ ] No unit tests cover the sidebar or the reports page, so none were added or run

**Manual checks (local):**
- [ ] Open Master Data, open one category, then another: the first should close. **Not checked in the browser.**
- [ ] Open Reports while Master Data is open: Master Data should close, and the other way round. **Not checked in the browser.**
- [ ] Reports, Sales Rep: two tabs (Sales Visit Register, Sales Rep Performance) on one page, switching keeps the filters per tab. **Not checked in the browser.**
- [ ] Log in as a role that can only see Sales Visits: the Enquiries menu item should open Sales Visits and no tab strip should show. **Not checked, needs a test role.**
