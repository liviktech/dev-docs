# 2026-10-10 - order-booking-ui-polish (frontend)

## Summary
The Order Booking screens had a mix of styles: capital letters in places, different font sizes and weights between sections, a centred Grand Total column, two loaders after a save, a yellow warning that stayed after the product was picked, and an order preview that could only be printed. This PR cleans these up, all in the Order Booking module, and adds a Download PDF button to the order preview. The "+ Add New" item in dropdowns is now styled the same way across the whole app, and the table component got two opt-in settings that leave every other table unchanged.

Everything below was checked by the TypeScript compile only (see the test plan). **The screens were not clicked through in a browser for this PR**, so the visual points still need a quick look from the reviewer.

**The product-specific "Photos" badge is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/format.ts` | Added `toTitleCase` (every word starts with a capital, one and two letter words like "PJ" are kept) and `FORM_INPUT_TEXT` (the shared text style for form inputs). |
| `src/features/order-booking/header-detail-tab.tsx` | Grand Total column is right aligned. Party filter options, party column and product column show names in title case. Date, product and amount text colour and weight match the enquiry table. |
| `src/features/order-booking/booking-header-form.tsx` | Field names show in normal case (Party Name) instead of all caps. Party and project options use title case. Remarks value uses the shared input text style. Enquiry No and Party Name fields are narrower, Remarks is wider. Party and Project add-new items are highlighted. |
| `src/features/order-booking/add-product-modal.tsx` | Typed text in inputs is no longer forced to capitals. The yellow "Action Required" banner hides as soon as a product is picked. Charge dropdown names use title case. The save confirmation was tried and then commented out, so Save works directly. |
| `src/features/order-booking/product-detail-panel.tsx` | Text boxes use the shared input style. Product and Type options use title case. The "not recorded in our master" banner hides once a product is selected. Product add-new item is highlighted. |
| `src/features/order-booking/shape-config-card.tsx` | Shape No input uses the shared input style. |
| `src/features/order-booking/remarks-others-panel.tsx` | Note input and Logo options use the shared style and title case. |
| `src/features/order-booking/size-details-panel.tsx` | Size table inputs, read-only cells and dropdowns are 12px medium dark slate (were 11px). |
| `src/features/order-booking/entry-table.tsx` | Column headers are 13px bold like the form labels. Saved rows use the same 12px medium text. |
| `src/features/order-booking/charges-table.tsx` | Charge names in title case. Cells and inputs use the shared text style. The totals (Charges, Material, Product) are now rows of the table itself so they line up with the Amount column. With more than 4 charges in the Add Product popup only the rows scroll. Add-new charge item is highlighted. |
| `src/features/order-booking/products-tab.tsx` | Product names in title case. Only one loader shows after a save. The Order Totals section uses the same heading, label and value styles as the other sections, and the Grand Total box uses the sidebar colour. |
| `src/features/order-booking/booking-preview-modal.tsx` | Close button removed from the footer (commented out). New Download PDF button next to a renamed Print button. Headings use normal letter spacing. Units read MM, PCS and NOS. Party name in title case. Summary labels read Material Charges, Charges, Subtotal and Discount. |
| `src/features/order-booking/order-booking-pdf.ts` (new) | Builds the order sheet as a PDF with real text, using jsPDF and jspdf-autotable. |
| `src/features/order-booking/review-tab.tsx` | Party and product names in title case. |
| `src/features/order-booking/work-order-tab.tsx` | Party and product names in title case. |
| `src/features/order-booking/work-order-summary-modal.tsx` | Party and product names in title case. |
| `src/features/order-booking/work-order-edit-dialog.tsx` | Party name in title case. |
| `src/features/order-booking/import-from-history-modal.tsx` | Description line under the title removed and the bottom padding reduced. |
| `src/features/ai-extraction/photo-review-panel.tsx` | "Use this photo and read it" and "Add N row(s) to products" buttons are cyan like the other primary buttons. The description under the dialog title is hidden (commented out). |
| `src/components/ui/select.tsx` | The "+ Add New ..." item in a dropdown is now set apart from the real options (divider above it, cyan bold text) in every dropdown that has one. New `highlightAddNew` setting, on by default, to turn it off. |
| `src/components/ui/data-table.tsx` | New optional `footerRows` and `scrollClassName` settings. Both are off by default, so every other table is unchanged. |

## Why
Users noticed that the same kind of value looked different from one place to another, for example product names in capitals in tables but title case elsewhere, typed text in different sizes, and a Grand Total that sat in the middle of its column. Some behaviour also got in the way: the yellow banner stayed after the product was chosen, two loaders showed at once after saving the first product, the add-new items looked like normal options, and the order preview had no way to save a PDF file. Most of the work is small style fixes. The real work is the PDF and the totals that now sit inside the charges table.

## How
**Title case for names**: `toTitleCase` is applied only where text is shown (dropdown options, table cells, summary lines). Saved values and anything sent to the backend are not changed. Short all caps words such as "PJ" are kept as they are.

**Shared input text style**: `FORM_INPUT_TEXT` (12px, medium, dark slate) is passed to the plain text boxes so they match the dropdown values beside them. The shared Input component itself was not changed.

**Add-new item style**: In the shared dropdown, the "+ Add New ..." item has a divider above it and cyan bold text, so it is clearly not one of the real options. This applies to every dropdown in the app that offers an add-new item (the booking, product, charge, party, project, sales rep and source dropdowns). `highlightAddNew={false}` turns it off for one dropdown.

**Opt-in table settings**: `footerRows` and `scrollClassName` on the table both default to off. Only the charges table turns them on.

**Charges totals inside the table**: The three totals are table footer rows. Label and amount sit in one right aligned cell that ends at the Amount column, so the figures line up with the amounts above. The footer is sticky, and the table scrolls (header and totals stay put) once there are more than 4 charges in the popup.

**Banner and loaders**: The yellow banner now depends only on whether a product is selected. The "Updating products" overlay shows only when the table already has rows, and the table's own loader covers the empty case.

**Download PDF**: `generateOrderBookingPdf` draws the same sections as the preview (header, order details, product blocks with specification grid, size table and amount breakdown, and the order total summary). It follows the orientation chosen in the preview, repeats table headers on new pages, and adds page numbers. Its wording matches the preview (Material Charges, Charges, Subtotal, Discount, MM, PCS, NOS). The built in PDF fonts cannot draw the rupee sign, so the PDF writes it as "Rs." and the routing arrows as ">".

## Decisions made
**The Save confirmation dialog was built and then removed.** It asked "Save this product?" after Save. It was commented out on request, so the code is still in `add-product-modal.tsx` if it is wanted back.

**"Remove" means comment out.** The Close button in the preview footer, the Save confirmation and the photo dialog description were commented out, not deleted, so they are easy to restore.

**The currency symbol is spelled "Rs." in the PDF.** Drawing the real symbol would need a custom font file added to the app. That was not worth it for this change.

**Photo dialog description.** The description line is hidden for both the order booking and the enquiry photo dialogs, because both use the same component.

## Notes
- The cyan button colours in the photo dialog live in a component shared with the enquiry photos dialog, so the cyan shows there too.
- The product-specific Photos badge is not included. Photos belong to the whole order and nothing records which photo a product was read from, so showing one photo per product needs a new link in the database, the draft and the save. That was discussed but not built.
- The green Submit button in Add Product still saves straight away, as before.
- The preview shows the order's enquiry id field in the Enquiry No box, and the PDF copies that. It was not checked whether that is the readable enquiry code.
- The backend change in `PR-backend.md` (charge duplicates) is a separate repo and needs its migration run.

## Bugs found while testing (already there before, not fixed here)
**1. Two unused variables fail the type check.** `src/features/production-scan/components/stage-work-panel.tsx` has `edge` (line 111) and `dash` (line 120) declared but never used, so `tsc` reports two TS6133 errors. This file is not part of this PR and was not touched. It affects anyone running `npm run build`.

## Next steps
- Decide whether to link each product to the photo it was read from, so the Photos badge can be product specific.
- Fix the two unused variables in `stage-work-panel.tsx` so the type check is clean.

## Test plan
**Automated tests (no test files cover this change):**
- [ ] `npx tsc --noEmit -p tsconfig.app.json`: no errors in any file changed here. It still reports the two unused variable errors in `stage-work-panel.tsx` listed above, which were there before.
- [x] PDF generator run from a script with sample data (3 products, 14 size rows, charges, in both landscape and portrait): created 3 pages in landscape and 2 pages in portrait, no error. The text was read back from the file and every section was present, table headers repeated on the next page, and no section heading was left alone at the bottom of a page.

**Manual checks (local):**
- [ ] Open the order booking list and the header form, confirm title case in the party and product values, the right aligned Grand Total and the narrower Enquiry No and Party Name fields. Not done in a browser for this PR.
- [ ] Add Product: type in the fields (no forced capitals), pick a product (yellow banner hides at once), add more than 4 charges (rows scroll, header and totals stay), check the totals line up with the Amount column. Not done in a browser for this PR.
- [ ] Order Preview: check Print, Download PDF (open the file in both orientations) and the MM, PCS and NOS wording. Not done in a browser for this PR. The PDF was only checked from the sample run above, including its text, and not looked at as a page image.
- [ ] Save the first product on an empty booking and confirm only one loader shows. Not done in a browser for this PR.
