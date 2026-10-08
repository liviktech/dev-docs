# 2026-10-08 - final-invoice-ui-fixes (frontend)

## Summary

The Final Invoice page had a party dropdown that did not work, an Admin Approve button that looked different from the one on the PI Status page, a table where the whole row opened the details modal, and a plain details modal that did not match the other modules. This PR fixes the party filter call, copies the PI Admin Approve button style, makes only the FI number open the details modal, and redesigns the details modal to match the Proforma Invoice details modal. The page also gets KPI cards, status tabs with counts, date and party filters, and a CSV export.

**The Final Invoice preview and print sheet changes are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/final-invoices/api.ts` | The party filter is no longer sent to the backend when "All Parties" (`ALL`) is selected. |
| `src/features/final-invoices/final-invoices-page.tsx` | Added KPI cards (total, pending, confirmed, collected), status tabs with counts, start and end date pickers, a searchable party dropdown with a reset button, a loading state and an Export CSV button. The party list is built from the parties master plus the invoices, so it does not shrink when a party is picked. The list is also filtered by party on the page. |
| `src/features/final-invoices/components/final-invoices-table.tsx` | The green Admin Approve button now uses the same solid green style as the PI Status table. Only the FI number button opens the details modal, the row click was removed. Added Party and Admin approval columns with name and status, an invoice date column and a loading state. The file was also reformatted by the editor (double quotes and semicolons), so the diff looks bigger than the real change. |
| `src/features/final-invoices/components/final-invoice-detail-modal.tsx` | Rebuilt to match the PI details modal. Wide dialog with a white header, invoice number, work order and status pills. Cards for Invoice Reference, Bill-To and Payment Summary. A billed line items table with totals, an Additional Charges card, an Invoice Totals card, Party and Admin approval cards, and decline reason and remarks when present. The footer Preview / Print button is now cyan instead of violet. |

## Why

The party dropdown showed an error on every pick because the backend returned a 500 (see the backend PR). The approve button, the table behaviour and the details modal were also out of step with the Proforma Invoice screens, which the team uses as the reference design. The real work was the details modal. The rest were small changes.

## How

**Admin Approve button**: Copied the class list from `ApproveActionButton` in the PI Status table (solid emerald, white text, bold, small shadow, press effect). The Party "Approve" button and the "Approved" label are unchanged.

**Details modal opens only from the FI number**: The table no longer passes `onRowClick` to `DataTable`, so clicking any other cell does nothing. The FI number button still calls `onViewDetails`.

**Details modal data**: The modal loads the full invoice with `finalInvoiceDetailQueryOptions` so it has the items, charges and taxes. While that loads it shows the same spinner the PI modal uses. The list row is used as a fallback so the header shows straight away.

**Totals**: Glass value is the sum of item amounts and charges is the sum of charge amounts. If the invoice has no tax rows, tax is worked out as the total minus glass value and charges. Advance paid is total minus remaining balance. These come from the existing detail data, nothing new is calculated on the server.

## Decisions made

**The details modal Bill-To card shows party name and party code only.** The Final Invoice data has no address, GSTIN or contact details. Getting them needs the linked Proforma Invoice, which was not wired in.

**Preview and print sheet left as it was.** An earlier attempt to load the PI into the preview sheet (real address, GSTIN, piece sizes, holes, taxes) is not in this working tree, so it is not part of this PR. The sheet still uses placeholder customer details, a fixed 18% tax split and sizes worked out from area.

**Row click removed instead of kept.** The request was that only the FI number opens the modal. We did not add extra clickable cells either.

## Bugs found while testing (already there before, not fixed here)

**1. Preview sheet shows placeholder details.** `final-invoice-as-proforma-data.ts` hard codes the customer address ("Customer Delivery Address, Tamil Nadu"), empty GSTIN, phone and email, a TRICHY destination, zero holes and cutouts, and piece sizes guessed from area at a 1 to 1.5 ratio. This was already there before and affects every printed Final Invoice.

## Next steps

- Return the linked PI id from the final invoice API and build the preview sheet from the PI, so the printed invoice shows the real address, GSTIN, sizes, holes and taxes.
- Add the customer address and GSTIN to the details modal Bill-To card once that data is available.

## Test plan

**Automated tests (no test files for this feature, type check only):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: no errors in the final-invoices files after the changes (checked after the details modal rewrite; the unused `Eye` import in the table was removed to clear the one error).
- [ ] `npm run build`: not run.

**Manual checks (local):**
- [ ] Party dropdown, Admin Approve button style, FI-number-only click and the redesigned details modal were not opened in a browser by me, so they are untested live. The party filter 500 was reproduced from the browser console log you shared and traced to the backend query.
