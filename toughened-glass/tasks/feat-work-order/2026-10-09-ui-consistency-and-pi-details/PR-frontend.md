# 2026-10-09 - ui-consistency-and-pi-details (frontend)

## Summary

This PR carries on from the 2026-10-08 work-order-and-list-polish PR. The Work Order, Enquiry, PI Status and Order Booking screens still had different headers, tab colours, fonts and table styles. This PR makes the stages and piece pages, the shared tabs and sidebar, the Enquiry and Order Booking pages, and the Proforma Invoice screens follow the same rules. The most work went into the Proforma Invoice Details modal (new card headers, two-card top section, label and value layout) and the Work Order stages table.

None of this was checked in a browser. I ran the TypeScript check after each change, but I did not click through the screens. See "Test plan".

**The Proforma Invoice print and PDF fonts are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/routes/work-orders_.$workOrderId.tsx` | The stages page has one white header bar with the back button, the work order number, the status badge and party, and Scan Barcode on the right. The loader is centered in the page. |
| `src/features/production-scan/work-order-pieces-table.tsx` | Rebuilt to a sketch from the team: columns are Piece, Barcode, Size, Current stage, Status and the open icon. The product line sits in the first column only, and its Pending Review badge is in the Status column. Every column has a vertical line (`slate-300`), and the Piece and Barcode columns are centered. Rows are not clickable, only the open icon opens a piece. Shared with the Production Line pieces popup. |
| `src/features/production-scan/scan-detail-page.tsx` | The piece page has a header bar with the back button, work order number and piece label, and Print label and Reject piece on the right. Both sections are now separate cards with solid colours (`sky-100` and `slate-100`). KPI card text is `text-slate-800`. |
| `src/features/production-scan/scan-dialog.tsx` | The Open button is at the bottom right, no longer full width. Monospace font removed. |
| `src/features/production-scan/components/stage-work-panel.tsx`, `production-line-page.tsx` | Monospace font removed. Merge conflict in the stage panel resolved (kept both the `slate-300` border and the `slate-800` label). |
| `src/features/order-booking/work-order-summary-modal.tsx` | The print menu is at the top, to the left of the close button. The footer has a Close button with text. |
| `src/components/ui/filter-tabs.tsx` | The selected tab colour is one shared constant, `ACTIVE_TAB_CLASS` (dark cyan). The per-page overrides were removed from Work Orders, Enquiries and Order Booking. The `activeClassName` option is still there for one-off use. |
| `src/components/ui/module-tabs.tsx` | Unselected tabs have a white tint, the selected tab is a stronger sky blue with a darker underline. |
| `src/components/layout/app-sidebar.tsx`, `src/components/layout/superadmin-sidebar.tsx` | The glow around the selected menu item is gone. |
| `src/components/ui/confirm-action-modal.tsx` | New optional `icon` prop, used by the Convert confirmation. |
| `src/features/enquiries/enquiries-page.tsx` | Approve, Convert and Reopen stay outside. Edit and Close moved into a kebab menu that is always last. Convert asks for confirmation. The selected status tab uses the shared colour. |
| `src/features/enquiries/enquiry-row-menu.tsx` (new) | The kebab menu with Edit Enquiry and Close Enquiry. |
| `src/features/enquiries/enquiry-form-modal.tsx` | Remarks sits to the right of Enquiry Source. Convert in the footer asks for confirmation. Fixed a closing tag (`</Button>` to `</button>`) and a duplicate wrapper div that stopped the file compiling. |
| `src/features/order-booking/booking-page.tsx` | The booking page has one white header bar with the back button and the page title. |
| `src/features/order-booking/header-detail-tab.tsx` | Date and Grand Total columns are centered. The selected tab uses the shared colour. |
| `src/features/order-booking/booking-header-form.tsx`, `review-tab.tsx` | Monospace font removed from the booking number. |
| `src/features/pi-status/pi-status-page.tsx` | The New PI button now matches the Enquiry page button. Unused `Plus` import removed. |
| `src/features/pi-status/components/pi-status-table.tsx` | Only the PI Number is clickable. Order No, Party, Work Order and the status badge are plain text. Hard-coded font classes removed. Fixed two buttons that were closed with `</span>`. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | The top section is now two cards: General Details, and one card holding Bill-To and Ship-To side by side. General Details fields are 3 columns and show as `Label : value` with no boxes, and long values wrap. Attached Files sit on the same line as the heading. Booked Glass Line Items, Glass and Surcharges Breakdown and Tax and Settlement Summary all have the same card header strip as General Details. Left and right dimensions now match the top and bottom line. Number columns use the same semibold weight as Product Specification. The digit-spacing style (`tabular-nums`) was removed from this modal. |
| `src/features/pi-status/components/pi-status-badge.tsx`, `invoice-preview-modal.tsx` | Hard-coded font class removed. |
| `src/features/invoices/create-proforma-invoice-dialog.tsx` | The order picker table fills the full height of the dialog. The search box is on the right. Monospace font removed from the picker columns and the selected total. Unused `Plus` import removed. |
| `src/routes/invoices.tsx`, `src/components/ui/table-action-buttons.tsx` | Hard-coded font class removed. |
| `src/features/order-booking/products-tab.tsx` | Unused `Plus` import removed. |
| `src/features/parties/party-dialog.tsx`, `src/features/sales-reps/sales-rep-dialog.tsx` | Monospace font removed from the fields that open from the New Enquiry modal. |

## Why

The earlier PR cleaned up the Work Order list. After that, the pages next to it (stages, piece, Enquiry, PI Status, Order Booking and the Proforma Invoice dialogs) still looked different, and the fonts were set in several different ways: a monospace class, an inline font, or a hard-coded font class. One selected-tab colour and one app font mean the next change only has to be made in one place.

## How

**One font**: the app font (Hanken Grotesk) is set once in `src/index.css` and inherited. The monospace classes and the `font-['Hanken_Grotesk_Variable',sans-serif]` classes were removed, so nothing overrides it. Digit spacing (`tabular-nums`) was also removed from the Proforma Invoice Details modal, because it made numbers look like a different font.

**One selected-tab colour**: `ACTIVE_TAB_CLASS` in `filter-tabs.tsx`. Change that one value and every status tab bar changes.

**Stages table**: it follows the team sketch. The product row holds its text only in the first column so the Barcode column starts right where the product text ends.

**Proforma Invoice Details**: every section card uses the same grey header strip (15px bold title, optional chip on the right) with the content below it.

## Decisions made

**The Proforma Invoice PDF and the printed Work Order PDF still use `helvetica`.** They are generated files, not screens, so they cannot use the web font without embedding a font file.

**The status badge on the PI table is no longer clickable.** You asked that only the PI Number be clickable, so Order No, Party, Work Order and the status are plain text.

**The new header bars are written inline in each page.** The stages page, the piece page and the booking page each have their own copy of the header markup. A shared header component would be a good clean-up, but it was left out to keep this change small.

**Rows in the stages table are no longer clickable.** Only the open icon opens a piece. This also applies to the Production Line pieces popup, which uses the same table.

## Bugs found while testing (already there before, not fixed here)

**1. Some screens still use `tabular-nums`.** It was kept on the lists and other modals (for example the Work Orders list and the Proforma Invoice dialog tables), where equal-width digits matter. Only the PI Details modal had it removed.

**2. The stages table has a fixed first-column width (`w-72`).** A very long product name could push the Barcode column out of line.

## Notes

- Several of these changes were already committed or merged when this document was written, so the file list comes from the work done in this task, not from a fresh diff.
- Two merge conflicts (`work-order-pieces-table.tsx` and `stage-work-panel.tsx`) were resolved by keeping both sides. The pieces table resolution was staged.
- Other commits on this branch (for example the `invoice` and `pi-status` merges) came from teammates and are not described here.

## Next steps

- Build one shared page header component for the stages, piece and booking pages.
- Decide whether to remove `tabular-nums` from the remaining lists too.
- Embed the app font in the PDFs if the printed pages should match the screens.

## Test plan

**Automated tests (no test files for these screens):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passed at the end of the last change, with no errors.

**Manual checks (local):**
- [ ] Stages page: the header bar shows the back button, work order number and Scan Barcode, and the loader is centered while pieces load. Not checked in a browser.
- [ ] Stages table: the Barcode column starts at the end of the product text, every column has a line, and clicking a row does nothing while the open icon works. Not checked in a browser.
- [ ] Selected tabs on Work Orders, Enquiries, Order Booking and PI Status all show the same dark cyan. Not checked in a browser.
- [ ] Enquiry list: the kebab holds Edit and Close and is the last button, and Convert asks first. Not checked in a browser.
- [ ] PI Status list: only the PI Number is clickable. Not checked in a browser.
- [ ] Proforma Invoice Details: two top cards, `Label : value` fields, matching card headers, and no font mismatch. Not checked in a browser.
- [ ] Proforma Invoice dialog: the table fills the dialog height and the search box is on the right. Not checked in a browser.
