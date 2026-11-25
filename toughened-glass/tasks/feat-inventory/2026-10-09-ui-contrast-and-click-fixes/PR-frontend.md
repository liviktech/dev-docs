# 2026-10-09 - ui-contrast-and-click-fixes (frontend)

## Summary

Input boxes, dropdowns, date pickers, search bars, modal boxes and tables used a mix of very light borders (slate-100, slate-200, sky-200) and some fields had no visible border at all, so many screens looked washed out and uneven. The scrollbar thumb was also almost the same colour as its track. Three tables also opened a details modal when any part of the row was clicked. This PR puts every field on one border colour and every box and table on another, adds one app-wide scrollbar style, fixes two print and invoice layouts that showed a sideways scrollbar, and limits the modal click in the PI Status and Approved Invoices tables.

**Dark mode colours are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/index.css` | `--border` is now `#cbd5e1` (slate-300) and `--input` is `#94a3b8` (slate-400). Added an app-wide scrollbar style (thin, rounded, no arrows) driven by `--scrollbar-thumb`, `--scrollbar-thumb-hover` and `--scrollbar-track`. The old helper classes (`custom-scrollbar`, `thin-scrollbar`, `show-scrollbar`, `visible-scrollbar`) now read the same variables. |
| `src/components/ui/textarea.tsx` | Background changed from transparent to white so it matches Input. |
| `src/components/ui/date-picker.tsx` | Trigger border is now slate-400. Text, placeholder, prefix and calendar icon are darker. Hover is stronger. |
| `src/components/ui/select.tsx` | Chevron and search icon are darker. Search box border is slate-400. Dropdown panel border is slate-300. |
| `src/components/ui/search-bar.tsx`, `checkbox.tsx` | Search bar has a slate-400 border and a white background. Checkbox border is slate-400. |
| `src/components/ui/button.tsx` | Outline buttons (print options, zoom, export, Close, Cancel) now get a visible border. |
| `src/components/ui/table.tsx`, `data-table.tsx`, `card.tsx`, `kpi-card.tsx`, `filter-toolbar.tsx`, `filter-tabs.tsx`, `dialog.tsx`, `view-details-dialog.tsx`, `badge.tsx`, `toast.tsx` | Container, row, column, tab, footer and badge borders moved to slate-300 (badges and toast icon boxes to the -300 shade of their colour). |
| `src/features/**` (many `.tsx` files) | Every `border-slate-100` and `border-slate-200` was changed to `border-slate-300`. Field overrides with a light border (invoice dialog, enquiry form, party dialog, order discount, charges table, upload boxes, print options toggle) were changed to slate-400. Party dialog fields also changed from grey to white. |
| `src/features/invoices/create-proforma-invoice-dialog.tsx` | Glass specifications table no longer scrolls sideways. It is fixed width, headers and cells wrap, padding is tighter and the minimum column widths were removed. |
| `src/features/pi-status/components/invoice-preview-modal.tsx` | The fit-to-width maths now leaves room for the vertical scrollbar. Sideways scrolling is hidden at 100% zoom or less and only shows when zoomed in. |
| `src/features/pi-status/components/pi-status-table.tsx` | The PI Status badge and the Order No value are now plain displays, not buttons that open the PI details modal. |
| `src/routes/work-orders.tsx` | In the Approved Invoices tab the row click was removed. The PI No badge is now the only thing that opens the view modal (disabled when the row has no order booking). |

## Why

The request was to make field, box and table borders clearly visible and the same everywhere, make the scrollbar visible without hovering, fit the invoice screens without a sideways scrollbar, and stop whole rows or extra cells from opening modals. Most of it is a colour change, but it touches many files because many screens had their own light border classes. The real work was finding those overrides.

## How

**Two border colours**: Fields (input, textarea, select, date picker, search, checkbox, upload boxes) use slate-400. Boxes, cards, tables, modals, tabs and dividers use slate-300. The default `--border` and `--input` variables were set to the same two values, so anything without an explicit colour follows them.

**Scrollbar**: One rule on `*` in the base layer sets a thin rounded scrollbar from three CSS variables. `.no-scrollbar` still wins, so scrollbars hidden on purpose stay hidden. The default is 6px wide and the old helper classes stay 5px.

**Invoice fit**: The proforma table uses `table-fixed` with wrapping cells instead of horizontal scroll. The print preview takes 12px off the available width for the scrollbar so the fitted sheet never overflows.

**Click targets**: The PI Status table and the Approved Invoices tab no longer open the modal from the status badge, the order number or the row. The PI Number button and the existing action buttons are unchanged.

## Decisions made

**Dark mode left alone.** Only the light theme values were changed. The dark `--border` and `--input` values were not touched.

**Blanket replace for light borders.** Every slate-100 and slate-200 border in `src/features` and `src/components` became slate-300 with a search and replace instead of file by file. Some faint dividers (for example inside table headers) are now stronger than before. Tell the team to point out any screen that looks too heavy.

**PI No is the click target in Approved Invoices.** That tab has no Work Order No column, so the first ID column was used. It can be switched to Booking No.

**Equal column widths in the proforma table.** Long product names now wrap instead of using a minimum width. Some columns could be given set widths later.

## Notes

- `src/index.css` lost the scrollbar edits twice during the work and they were re-applied. Something outside the session rewrote the file. Check that the scrollbar variables are in the final merged file.
- Several of these changes were already committed by the time this document was written, so a diff against the last merge shows only the invoice, PI Status and work orders files.

## Next steps

- Look at the contrast on the screens with the busiest tables (order booking, PI Status, dispatch) and tone down any border that is too heavy.
- Add dark mode values for the new border, input and scrollbar variables.

## Test plan

**Automated tests (no test run for this change):**
- [x] Type check (`tsc -p tsconfig.app.json --noEmit`): no new errors. Existing unused-variable errors (TS6133) in dispatch files and a few others remain.
- [x] Production build (`vite build`): passed after the scrollbar change.

**Manual checks (local):**
- [ ] Field, dropdown, date picker, search bar, filter tab, upload box, remarks and table borders on the main modules - not checked in the browser, only checked in the code.
- [ ] Scrollbar thumb visible without hover - not checked in the browser.
- [ ] Proforma Invoice dialog and print preview show no sideways scrollbar - not checked in the browser.
- [ ] PI Status badge and Order No do nothing on click, and the PI Number still opens the modal - not checked in the browser.
- [ ] Approved Invoices tab: row click does nothing, PI No opens the view modal - not checked in the browser.
