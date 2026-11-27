# 2026-10-15 - final-invoice-tax-sheet-and-gst (frontend)

## Summary
The Final Invoice screens showed wrong amounts (Amount to Collect was 0 while the balance was 1,024.50), the preview did not look like the company's printed tax invoice, and the dispatch page had no vehicle checks, no auto-filled addresses and no e-Way Bill. This PR fixes the amounts, makes the Final Invoice preview and print look like the supplied tax invoice (header lines, ISI mark, QR, Note/Reference box under the products, terms), adds a loader on the Generate e-Invoice menu item, removes the blank space and the old form flash in the preview, and adds the dispatch changes: auto-filled gate pass addresses, vehicle number validation, a Dispatch Date column after Approval Date, and e-Way Bill (Road only).

Layout and the ISI mark were checked by rendering the sheet in headless Chrome. The ISI mark was redrawn several times from the picture and the last version was not rendered or checked in the running app. Nothing was checked against the Sandbox provider.

## What changed
| File | Change |
|---|---|
| `src/features/final-invoices/components/create-final-invoice-panel.tsx` | Amounts use the invoice total, the advance paid (total times advance ratio) and the amount due on this invoice. Labels and the cap message were changed to match. |
| `src/features/final-invoices/components/approve-final-invoice-modal.tsx` | Amount to Collect is the remaining balance, read-only. |
| `src/features/final-invoices/components/final-invoices-table.tsx` | "Paid in Full" shows only when approved and the balance is zero. New `generatingEinvoiceId` prop drives the loader on the Generate e-Invoice item. |
| `src/features/final-invoices/final-invoices-page.tsx` | Tracks which invoice is generating and clears it when the call finishes. |
| `src/features/final-invoices/components/final-invoice-preview-modal.tsx` | Always shows the new tax invoice sheet in portrait. Shows only a spinner until the invoice, company, parties and e-Invoice data are loaded, so the old or half filled sheet is never drawn. The empty space under the sheet is gone. Print and download are off while loading. |
| `src/features/final-invoices/components/final-invoice-detail-modal.tsx`, `final-invoice-as-proforma-data.ts` | Show the discount line and the advance paid, and use the record's remaining amount. |
| `src/features/final-invoices/api.ts`, `types.ts` | Map `advanceRatio`, `advanceAmount` and `discountAmount`. |
| `src/features/e-invoice/components/einvoice-tax-invoice-sheet.tsx` (new) | The tax invoice sheet. Header with ISI mark, two contact lines under the title, products with the Note/Reference box directly under the last product (closing line, then an open full width box), terms in the lower left box, footer with jurisdiction, E&O.E. and the signature. Print and PDF styles are scoped to the sheet. |
| `src/features/e-invoice/components/einvoice-sheet-data.ts` (new) | Builds the sheet data from the Final Invoice record. Holds the terms text and the contact lines. |
| `src/features/e-invoice/auto-generate.ts` (new) | Generates an e-Invoice on request and waits for the refetch, so the loader stays until the preview shows the new IRN. |
| `src/features/e-invoice/` other files (new) | API calls, queries, types, the preview modal, cancel modal, audit log modal and the compliance panel. |
| `src/components/ui/row-actions-menu.tsx` | New `loading` flag on an item. The item is disabled with a spinner and the kebab button spins while it runs. |
| `src/features/dispatch/components/gate-pass-modal.tsx` | From and To addresses fill in automatically (delivery or shipping address of the party, else the default, else the first) and stay editable. A user edit is never overwritten. |
| `src/features/dispatch/components/approve-dispatch-modal.tsx` | Vehicle number is validated against the road vehicle pattern, cleaned up, and sent in the cleaned form. |
| `src/features/dispatch/components/dispatch-table.tsx` | Dispatch Date column moved to after Approval Date. Also holds the e-Way Bill actions in the row menu. |
| `src/features/dispatch/dispatch-page.tsx` | Loads which invoices have a live e-Way Bill, and wires the generate dialog and the slip modal. |
| `src/features/dispatch/components/gate-pass-sheet.tsx` | Prints the e-Way Bill number and valid-upto date when one exists. |
| `src/features/e-way-bill/` (new) | e-Way Bill API calls, queries, generate dialog and modal, slip, and update vehicle modal. Road is the only mode, with no mode selector. The vehicle is filled from the dispatch approval. |
| `src/features/e-way-bill/utils.ts` (new) | `normaliseVehicleNumber` strips everything except letters and digits. |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx` | New optional `headerRight` slot for a QR. The layout is unchanged when it is not passed. |

## Why
The task list asked for the Final Invoice to look exactly like the supplied tax invoice, for the money figures to be right, and for the dispatch fixes. The sheet is the real work: the layout is table based and has to print on one page.

## How
**One sheet for preview, print and PDF**: the preview, print and PDF all use the same `.einv-sheet` markup and CSS (`EINVOICE_SHEET_CSS`), so what is on screen is what prints.

**Loader**: the row menu item gets a spinner and the kebab button spins while the call runs. The generate function awaits the cache refresh, so the spinner stops only when the list and the preview have the new data.

**Preview loading**: the sheet is not drawn until all four queries are done, so there is no flash of an old or empty form and nothing to scroll.

**Gate pass addresses**: two refs (`fromEdited`, `toEdited`) record when the user typed in a box, and the auto fill skips that box from then on.

**Kept the same on purpose**: the Proforma Invoice sheet looks the same unless `headerRight` is passed.

## Decisions made
**Terms text is worded exactly as on the printed invoice.** The wording is copied from the image, including its spelling, so it matches what the company already prints.

**The complaints email is a constant.** The company profile has no field for it, so `complaint.rubine@yahoo.com` is a constant in `einvoice-sheet-data.ts`.

**The ISI mark is a drawn SVG.** No official file was available. It was redrawn from the picture and may still differ in small details.

## Notes
- Blank fields on the sheet are not filled yet (HSN, payment conditions, transporter and driver) because the data is not on the invoice.
- Replace the ISI mark with the official SVG file if the company has one.

## Next steps
- Add a field for the complaints email on the company profile.
- Fill the blank sheet fields once the data exists.

## Test plan
**Automated tests (no test suite run, type check only):**
- [x] `npx tsc -b` on the frontend: no errors after every change, including the last ISI change.
- [ ] No component tests were added for the sheet or the gate pass modal.

**Manual checks (local):**
- [x] Rendered the tax invoice sheet in headless Chrome and compared the layout to the supplied image (header lines, Note/Reference box, terms, footer).
- [ ] Final ISI mark - not rendered after the last redraw, so the spacing between letters is unchecked.
- [ ] Generate e-Invoice loader, preview loading state, gate pass auto fill, vehicle number validation and e-Way Bill dialog - not clicked through in the running app.
