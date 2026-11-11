# 2026-11-10 - pwa-reports-item-colours (frontend)

## Summary

This PR does two things in the frontend. First, it adds the Reports module to the mobile app (PWA): a reports home and a screen for each of the 17 desktop reports. Second, it makes the item, product, purchase and order booking screens work with items as a full combination (product + glass type + thickness + colours), and adds a Colours column to product rates. Both parts were type checked with `tsc -b` (clean). I did not open any screen in a browser or on a phone.

**No backend change for the mobile reports is included on purpose** - see "How" below. The mobile screens call the same API functions as the desktop reports. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/pwa/reports/mobile-reports-home.tsx` (new) | Reports home, lists the 7 groups and 17 reports from `REPORT_MENU`. |
| `src/features/pwa/reports/mobile-report-view.tsx` (new) | Opens one report by group and view, with the access check. |
| `src/features/pwa/reports/registry.ts` (new) | Lazy loaded screen for each report. |
| `src/features/pwa/reports/report-model.ts` (new) | Types for a report description: columns, filters, config. |
| `src/features/pwa/reports/report-ui.tsx` (new) | Shared cards, filter fields and the Export XLSX / Download PDF bar. |
| `src/features/pwa/reports/report-filters.ts` (new) | Shared filter state helpers. |
| `src/features/pwa/reports/table-report.tsx` (new) | `MobileTableReport`, one screen that draws 16 of the 17 reports from a config. |
| `src/features/pwa/reports/performance-report.tsx` (new) | Sales Rep Performance, its own screen (Enquiry Wise / Amount Wise tabs). |
| `src/features/pwa/reports/order-status-report.tsx`, `price-list-report.tsx`, `production-reports.tsx`, `raw-glass-reports.tsx`, `sales-income-reports.tsx`, `sales-rep-reports.tsx` (new) | Report configs for the groups, copying the desktop columns and formatting. |
| `src/routes/m.reports.tsx`, `src/routes/m.reports_.$group.$view.tsx` (new) | Thin routes that wrap the screens in `MobileNoAccess`. |
| `src/routes/reports.tsx` | In the installed app on a phone, `/reports` redirects to the mobile report (or the report list). Desktop and browser are unchanged. |
| `src/features/pwa/mobile-modules.ts` | Adds the Reports module to the mobile menu. |
| `PWA.md` | Documents Mobile Reports, the new routes, the folder, and what is not on mobile. |
| `src/features/reports/reports-page.tsx` | Title shows just "Reports · group" when a group has only one report. |
| `src/features/reports/order-status/order-status-report.tsx` | Removed the WO / PI / Customer search bar and its PDF filter line. |
| `src/features/products/product-colors-cell.tsx` (new) | `RateColorsCell`, the Colours dropdown of a rate row (first colour plus "+N", with Add Item / Add Colour). |
| `src/features/products/product-dialog.tsx` | New Colours column in Rates. Add Item starts with the row's thickness, type and the product's HSN. Rates only offer combinations that exist as items. |
| `src/features/products/product-view-modal.tsx` | New Colours column in the Rates table, with the colours of the matching items. |
| `src/features/items/items-page.tsx` | New product filter and a reset button, kept in the URL (`product`). |
| `src/features/items/item-dialog.tsx` | New `defaultValues` prop to start Add Item with some fields filled. |
| `src/features/items/queries.ts`, `src/features/items/types.ts` | Send `productName` to the items list. |
| `src/features/inventory/components/purchase-dialog.tsx` | A line's item is the exact product + type + colour + thickness match. Add Item starts with the row's values. A new type, colour or thickness re-looks up the item. An edited purchase no longer skips the item check. |
| `src/features/order-booking/product-detail-panel.tsx` | Colour options are only the colours set up for the picked product + thickness + type. The default "Clear" colour logic is removed. |
| `src/features/order-booking/add-product-modal.tsx` | New product starts with no colour. |

## Why

Reports are used by managers on the move, so the mobile app needed them with the same filters and downloads as desktop. The item changes follow from items now being one combination with a list of colours: the screens should only offer what really exists, and tell the user to add the item when it does not.

## How

**Mobile reports are descriptions, not copies**: 16 reports are only a `TableReportConfig` (columns, filters, formatting) drawn by one shared screen. Only Sales Rep Performance has its own screen. The configs copy the desktop columns and use the desktop's own API functions and filter option hooks, so no request changed.

**Filters and downloads**: Every report keeps its desktop filters in a bottom Filters sheet with a count of applied filters. Export XLSX and Download PDF give the same file as the desktop buttons (all matching rows, same columns, totals and filter lines).

**Phone redirect**: `/reports` uses `redirectToMobileScreen`, the same way the other mobile modules do, and keeps the `group` and `view` from the URL when it can.

**Strict item match**: In the purchase dialog, product Rates and order booking, an item only matches when it has a glass type, a thickness and the colour. Blank no longer means "any". This matches the backend change.

**Add Item with starting values**: `ItemDialog` takes `defaultValues` (it must stay the same object while the dialog is open, or the form resets). The purchase and product dialogs use it to carry over the row's thickness, type, colour and the product's HSN.

**Kept the same on purpose**: Desktop report pages and their API calls are not changed except the Order Status search bar and the page title above.

## Decisions made

**Removed the search bar from Order Status.** No other report page has one, and the mobile version would have had to copy it. The PDF filter line for it went too.

**No charts and no per-column sorting on mobile.** No registered report uses charts, and tables are shown as cards.

**Product filter list comes from all items.** The Items page builds the product dropdown from the items lookup (active or not), one entry per name ignoring case. A product named in the URL stays selectable even if no item has it.

## Notes

- The "Clear" default colour on a new order booking product was removed, so the user has to pick a colour from the ones set up for that product, thickness and type.
- `PWA.md` already lists the older limits. Reports are added to it as not having charts or column sorting.
- The matching backend change is in `PR-backend.md` of this folder.

## Next steps

- Try the mobile reports on a real phone or in device emulation, which was not done here.
- Fill in blank glass type, thickness or colours on old items, since a blank one now matches nothing.

## Test plan

**Automated tests (no tests added, type check clean):**
- [x] `npx tsc -b` (frontend): ran in this session, no errors.
- [ ] `eslint` on the new `src/features/pwa/reports` folder: not run in this session.

**Manual checks (local):**
- [ ] Open `/m/reports` on a phone width, open each of the 17 reports, apply filters, and download XLSX and PDF - not run, no browser check done.
- [ ] Installed app on a phone: `/reports?group=...&view=...` redirects to the mobile report - not run.
- [ ] Items page product filter and search by thickness or colour - not run (needs the backend change).
- [ ] Purchase dialog: pick type, colour and thickness with no matching item and see the message and Add Item starting filled - not run.
- [ ] Product Rates tab and product view show the Colours column - not run.
- [ ] Order booking colour list only shows the colours of the picked product, thickness and type - not run.
