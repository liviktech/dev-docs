# 2026-10-29 - lot-code-column-booked-colour (frontend)

## Summary

Two things were missing. In the work order approval screen the user had to pick the glass colour by hand, even though the colour is already booked on the order. And on the Stock page there was no way to see the lot codes of a product or open one lot on its own. This PR removes the manual colour picker, so "Select stock" shows only lots of the colour booked for that product. It also adds a Lot Code column to the Stock table, next to Product Code, with a view dialog for each lot code. The order preview, order details and order PDF also show the booked colour now.

**Old orders are not changed on purpose** - see "Notes" below. Items booked before the colour field existed have no colour saved, so for those the screen lists every colour and says so. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/work-order-stock-section.tsx` | The colour picker ("All colours", "Clear") and its state are removed. The booked colour shows as a fixed chip in the Select stock dialog. If no colour was booked, a short note says lots of every colour are shown. Approval sends no colour choice any more, so an old saved choice is cleared. |
| `src/features/inventory/stock-tab.tsx` | New Lot Code column after Product Code. It lists the lot codes of the product (first code plus "+N" with a popover, same as Purchase Number). Clicking a code opens the lot view dialog. `PurchaseNumbersCell` got an optional `title` for the hover text. |
| `src/features/inventory/components/lot-code-view-dialog.tsx` (new) | View dialog for one lot, styled like the Stock Product view. Tab 1 "Lot Details" has lot code, purchase no, supplier, dates, product, type, colour, thickness, sheet size, sheet and SQM counts, created by, remark and an Offcuts table with totals. Tab 2 "Lot History" shows the work orders that used the lot, and opens the work order or PI from there. |
| `src/features/inventory/components/lot-detail-dialog.tsx` | Only added `export` to the small helpers (`text`, table class names, `tabClass`, `Detail`, `Section`, `HistorySections`) so the new dialog can reuse them. No change in behaviour. |
| `src/features/order-booking/booking-preview-modal.tsx` | Shows a Color field for each product (colour name looked up from the glass colour master, falls back to the code). Layout of the Note and Description cells adjusted to fit the new field. Passes the colours to the PDF. |
| `src/features/order-booking/order-details-modal.tsx` | Shows the Color field for each product. Remark and Description cells re-ordered, Description now takes the full row. |
| `src/features/order-booking/order-booking-pdf.ts` | The PDF takes the glass colours and prints a Color cell. The key value rows are re-flowed to four cells per row (Shape No, Logo, Note and Remark share one row). |

## Why

The colour is booked per product on the order (migration 084 on the backend). Asking the user to choose it again when approving was extra work and allowed the wrong colour to be picked. On the Stock page, stock is shown one row per product, so the individual lots were hidden unless you opened the product view.

The backend already filters candidate lots by the booked colour and checks it when stock is saved, so the real work here was removing the picker and showing the booked colour clearly. The lot view dialog was mostly reuse of the existing lot history pieces.

## How

**Booked colour in Select stock**: each group already comes from the backend with `bookedColorCode`. The screen reads that, looks up the colour name, and shows it as a fixed chip. The candidate lots are already filtered by the backend, so nothing is filtered again in the browser.

**No colour choice sent**: the effect that reports the selection now sends an empty `choices` list. The backend then ignores any manual choice and clears an older saved one.

**Lot Code column**: built from the `items` of the stock product row (`lotId` and `lotCode`). The click opens `LotCodeViewDialog` with that lot id, which loads the full lot (offcuts, supplier, remark, created by) and its history with the existing lot hooks.

## Decisions made

**Picker removed completely, no "pick if nothing booked" fallback.** A fallback picker is what the reviewer still saw after the first attempt, because old orders have no colour. Showing all colours with a note is clearer than a control that does nothing useful.

**Lot code list reuses the Purchase Number cell.** It already handles several values with "+N" and a popover, so the two columns look and behave the same.

## Notes

- Items booked before migration 084 have `color_code` empty (checked in the dev database: the 8 mm Clear Glass items of 21 and 27 Oct have none). For these, Select stock lists lots of every colour. To restrict them, open the booking, set the Colour on the product and save.
- The order preview, order details and PDF colour changes were already in the working tree when this doc was written. They are described from the diff and were not re-tested by me.
- No backend files were changed for this work, so there is no `PR-backend.md`.

## Test plan

**Automated tests (no test files for this area, type check and lint only):**
- [x] Frontend `npx tsc --noEmit -p tsconfig.app.json`: no errors after the last edit.
- [x] `npx eslint src/features/order-booking/work-order-stock-section.tsx`: no errors.

**Manual checks (local):**
- [x] A screenshot from the reviewer showed the colour picker still visible for an old order (Annealed Glass, 8 mm, Clear Glass). It was removed after that.
- [ ] Select stock for an order booked with a colour, to see the chip and only that colour's lots, was not checked in a browser by me.
- [ ] The Lot Code column, the "+N" popover and the lot view dialog (both tabs) were not checked in a browser by me.
- [ ] Order preview, order details and the order PDF with the Color field were not checked in a browser by me.
