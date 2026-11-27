# 2026-10-10 - work-order-stages-and-pi-dialog (frontend)

## Summary

This PR carries on from the 2026-10-09 ui-consistency-and-pi-details PR. It cleans up the Work Order review and edit screens, the stage table on the piece page, the Proforma Invoice dialog tables, and a few shared styles (KPI numbers, Add Product labels). The screens now also match the backend change that removes Dispatch and HOLE from the stages.

None of this was checked in a browser. I ran the TypeScript check after each change, but I did not click through the screens. See "Test plan".

**The Work Order PDF print is not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/work-order-sheet-view.tsx` | The header is now two bordered cards (Work Order No, Proforma Inv No, Dispatch Type and Approval Date on the left, and Delivery Date, Project and Party on the right). Borders are lighter (`slate-300`), table values are semibold and column headers bold. Labels are one step bolder. The process route line shows full work-centre names joined with `->`. |
| `src/features/order-booking/work-order-pdf-mapping.ts` | The route now leaves out HOLE and Dispatch (Dispatch is matched by name). |
| `src/features/order-booking/factory-approval-sheet-dialog.tsx` | The Review and Approve modal is a little wider (`max-w-7xl`). Reject and Approve have no icons. Approve still shows a spinner while it runs. Unused icon imports removed. |
| `src/features/order-booking/work-order-form-fields.tsx` | Remark is a real text area (3 rows, resizable). Delivery Date cannot be a past date. This form is also used by the create form. |
| `src/routes/work-orders.tsx` | The Reject / Cancel remarks box is a 3-row text area. Unused `Input` import removed. |
| `src/features/production-scan/components/stage-work-panel.tsx` | The stage table always shows the same four columns: Sl. No., Size, Qty and Shape. The extra Holes, Centre Holes, Cutouts, Big Holes and Logo columns and their helper code were removed. The digit-spacing style was removed. |
| `src/features/production-scan/scan-detail-page.tsx`, `production-line-page.tsx`, `scan-dialog.tsx` | The digit-spacing style (`tabular-nums`) was removed so numbers use the plain app font. |
| `src/features/invoices/create-proforma-invoice-dialog.tsx` | Glass Specifications table: the body cells now have the same side padding as the headings. Additional Charges table: more space between Value and Type, values at 13px, amounts and the total moved left, and the total sits under the Amount column. GST rows (Total GST, CGST, SGST, IGST) use the same text style as Additional Charges. The Financial Calculation Summary text is slightly smaller. Unused `Plus` import removed. |
| `src/components/ui/kpi-card.tsx` | KPI numbers have wider letter spacing. This is shared, so every KPI card gets it. |
| `src/features/order-booking/add-product-modal.tsx` | Labels and table headings are Title Case at 13px. The font is unchanged. |
| `src/features/dispatch/components/dispatch-table.tsx` | Fixed a button that was closed with `</span>`, which stopped the file compiling. |

## Why

The review sheet and the edit dialog still looked different from the screens we cleaned up earlier (dark borders, one-line remark box, a route written with slashes and short codes). The Proforma Invoice dialog tables had misaligned columns and tight spacing. The stage table changed its columns from stage to stage, which made the operator screen hard to read.

## How

**Stage table**: `getFixedColumns` now returns only the Shape column, so the table is always Sl. No., Size, Qty and Shape.

**Route text**: the sheet builds its route from the work center names, and the mapping leaves out HOLE and Dispatch. Dispatch is matched by name because its code is `DIS` in some companies and `STR` in another.

**Table padding**: the shared table gives heading cells 8px of side padding and body cells 2px. In the Proforma Invoice glass table each body cell now carries the same side padding as its heading, and the first and last columns keep their own edge padding.

**Title Case labels**: the Add Product dialog already forced everything to uppercase in one place. I added rules on that dialog that apply `capitalize` to labels, table headings and uppercase text, so typed and selected values stay uppercase.

## Decisions made

**The Work Order PDF print is not changed.** It still builds its own header and route text. The route in the PDF comes from the same mapping, so it now leaves out HOLE and Dispatch too, but its layout and fonts were not touched.

**The hole and cutout counts were removed from the stage table.** You asked for only four columns. If operators still need those numbers, they would have to go in the stage details card.

**KPI letter spacing is a shared change.** It applies to every KPI card, not only the Proforma card where it was noticed.

**Dispatch is matched by name, not by code.** This avoids a hard-coded list of codes, but it means a work center that was renamed would no longer be recognised.

## Bugs found while testing (already there before, not fixed here)

**1. Several screens still use `tabular-nums`.** It was removed from the stage and piece pages and the PI details modal, but lists and other modals still use it.

**2. The printed Work Order PDF uses `helvetica`.** It is a generated file, so it cannot use the web font without embedding a font file.

## Notes

- Several of these changes were already committed or merged when this document was written, so the file list comes from the work done in this task, not from a fresh diff.
- Two of the earlier merges left broken code (a closing tag in `dispatch-table.tsx`, and a helper plus an import in the Proforma Invoice dialog). Those were fixed as part of this work.
- The backend part of the stage change is described in `PR-backend.md`. The two should go out together, because the screens now assume Dispatch and HOLE are not stages.

## Next steps

- Remove the remaining `tabular-nums` from lists and modals if the app should use one plain number style.
- Decide where hole and cutout counts should show for operators, now that they are out of the stage table.
- Embed the app font in the PDFs if the printed pages should match the screens.

## Test plan

**Automated tests (no test files for these screens):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passed at the end of the last change, with no errors.

**Manual checks (local):**
- [ ] Review and Approve: wider modal, two header cards, full route text with no Dispatch, and Reject and Approve without icons. Not checked in a browser.
- [ ] Edit Work Order: Remark is a text area and past delivery dates cannot be picked. Not checked in a browser.
- [ ] Reject / Cancel dialog: Remarks is a text area. Not checked in a browser.
- [ ] Stage table: only Sl. No., Size, Qty and Shape at every stage. Not checked in a browser.
- [ ] Proforma Invoice dialog: glass table columns line up, charges table spacing and 13px values, GST rows match. Not checked in a browser.
- [ ] Add Product modal: labels in Title Case at 13px. Not checked in a browser.
