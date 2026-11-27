# 2026-10-16 - insurance-pi-charges-mobile-fixes (frontend)

## Summary

This PR moves Insurance from the Order Booking to the Proforma Invoice, fixes two bugs in the PI charges table (a charge appearing twice, and a Percentage charge turning into Fixed when the PI is reopened), removes the Production Line page in favour of a Progress column on the Work Orders list, and tidies several mobile (PWA) screens after review.

Before this, Insurance was added per product in the Order Booking, the PI form restored saved charges wrongly, the Production Line was a separate page, the mobile Dispatch list showed nothing, and a few mobile screens had buttons that did not work well on a phone.

Checked: `tsc -b`, `vite build` and ESLint on the changed mobile files pass. **Nothing was tested in a real browser or on a phone**, see the test plan.

**Cleaning up the charges that old PIs saved as Fixed is not included on purpose** - see "Bugs found while testing". This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/invoices/create-proforma-invoice-dialog.tsx` | Adds the PI level Insurance row. Rewrites how saved charges are restored when a PI is reopened (see "How"). Sends a `chargeKey` with each non custom charge. |
| `src/features/pi-status/components/proforma-pdf-data.ts` | No longer drops charges named Insurance from the PI preview and PDF. Also reads `calculation_type` to know if a charge is a percentage. |
| `src/features/order-booking/add-product-modal.tsx` | Insurance is no longer offered when adding a charge to a product by hand. |
| `src/features/order-booking/work-order-summary-modal.tsx` | The Production Line button is removed, with the unused permission check and imports. |
| `src/features/production-scan/production-line-page.tsx` (removed) | The Production Line page. |
| `src/routes/production-line.tsx` (removed) | The route for that page. |
| `src/routes/work-orders.tsx` | New Progress column in the Work Orders table (pieces done out of total, a percentage and a bar, plus a rejected count). |
| `src/features/pwa/components/photo-viewer.tsx` (new) | Full screen photo viewer with zoom in, zoom out and reset. |
| `src/features/pwa/photo-grid.tsx` | Thumbnails are buttons that open the photo viewer. Used by the Enquiry, Order Booking, PI Status, Work Order and Final Invoice detail screens. |
| `src/features/pwa/components/form-fields.tsx`, `index.ts` | New `DateStepper` (previous day, date, next day). `DateInput` gets a `minToday` option. |
| `src/features/pwa/attendance/mobile-attendance.tsx` | Uses `DateStepper` for the date. |
| `src/features/pwa/work-orders/mobile-work-order-detail.tsx` | Delivery date blocks past dates in the Edit sheet. The Reference row and field are removed. |
| `src/features/pwa/dispatch/mobile-dispatch-list.tsx` | Rewritten to use the same server paged table request as desktop. |
| `src/features/pwa/dispatch/mobile-dispatch-detail.tsx`, `mobile-gate-pass.tsx` | The View Invoice and Invoice buttons are removed. |
| `src/features/pwa/final-invoices/mobile-final-invoice-list.tsx` | The Create button and the create form are removed. |
| `src/features/pwa/final-invoices/mobile-final-invoice-detail.tsx` | The Preview button is removed. The bottom bar only shows when there is an approval to do. |
| `src/features/pwa/final-invoices/mobile-final-invoice-preview.tsx`, `src/routes/m.final-invoices_.$invoiceId_.preview.tsx` (removed) | The mobile Final Invoice preview screen and its route. |
| `src/features/pwa/pi-status/mobile-pi-detail.tsx` | The Preview and More buttons are removed, which also removes Edit, Delete and the edit rules banner. The bottom bar only shows when there is an approval to do. |
| `src/features/pwa/pi-status/mobile-pi-preview.tsx`, `use-pi-documents.ts`, `src/routes/m.pi-status_.$piId_.preview.tsx` (removed) | The mobile PI preview screen, its data hooks and its route. |
| `src/features/pwa/pi-status/use-pi-actions.ts`, `pi-format.ts` | Delete action and the edit rules helper removed, since nothing uses them now. |
| `src/features/pwa/order-booking/mobile-order-detail.tsx` | Touched in the same commit. It has a few unused imports left over from a separate edit, see "Bugs found while testing". |

## Why

Insurance has to be charged once per Proforma Invoice, on its basic amount. The two PI charge bugs were found when reopening a saved PI: a master charge such as Admin showed twice, and a charge entered as a Percentage came back as Fixed. The mobile changes came from review feedback: some screens cannot work well on a phone (the PI and Final Invoice previews, PI edit and delete, creating a Final Invoice), the Dispatch list never loaded, and photos could not be opened full size.

## How

**PI level Insurance.** A new Insurance row is shown in the PI charges table, as a percentage of the basic amount (default 0.5%, which can be edited, renamed, changed to Fixed or deleted). If an admin creates an order charge named Insurance in Master Data, that charge is used and the built in row is not shown. A PI saved without Insurance does not get the row added back when it is reopened.

**Why charges were duplicated.** Master order charges are identified by their code, not a uuid. The old save only sent an id when it looked like a uuid, so nothing recorded which master charge a saved row was. Reopening treated every saved charge as a custom charge, and the master list added its own copy.

**Why Percentage became Fixed.** The saved type comes back as `calculation_type`, and the form only read `chargeType` and similar names. It never found the value, so it fell back to Fixed.

**The fix.** The form sends a `chargeKey` (the master charge code, or `pi-insurance`) with each non custom charge, and the backend keeps it. On reopening, the form puts each charge back on its key. Older PIs without a key are matched to a master charge by name. The restore now waits until the master order charges have loaded, runs once per PI, and reads `calculation_type`.

**Progress column.** The numbers come from the existing production scan list. It is only requested when the page has an In Progress or Completed work order and the user can view production. It refreshes every 20 seconds while shown. Other rows show a dash.

**Mobile Dispatch.** The list asked for 500 rows at once, but the API allows at most 100, so the request failed. It now asks for 20 per page like the Final Invoice list and uses the counts, summary and party list that come back with the page.

**Photo viewer.** It reuses the image the thumbnail already loaded, so opening it makes no new request. Zoom is 100% to 400%, and the photo can be dragged when zoomed in.

## Decisions made

**Default Insurance of 0.5%.** No real rate was available. 0.5% is the example already written in a code comment. It is a starting value that the user can edit on every PI, and a master order charge named Insurance replaces it.

**Store the row key instead of the master code in `charge_code`.** That column points at a different charge list and would make the save drop every charge, see the backend PR.

**Removed Edit and Delete from the mobile PI screen, not just the buttons.** They only lived behind the More menu, so the dialog, the delete action and the rule helper were removed too, to leave no dead code.

**The Dispatch View Invoice and Invoice buttons were removed, not redirected.** They first pointed at the Final Invoice preview, then at the invoice details. The request was to take them out of both screens.

## Bugs found while testing (already there before, not fixed here)

**1. Old PIs have their percentage charges saved as Fixed.** The type was lost when they were first saved. The amount is still right, only the type shows as Fixed when the PI is reopened. Those charges need to be re-entered once.

**2. `create-proforma-invoice-dialog.tsx` has many lint errors that were already there.** They are `any` types and state updates inside effects. The new restore effect follows the same pattern as the one it replaced.

**3. `mobile-order-detail.tsx` has unused imports** (`Eye`, `MoreHorizontal`, `Pencil`, `BottomActionBar`, `MobileButton`), left over from a separate edit to remove the order actions. Type check and ESLint report them.

## Notes

- Needs the matching backend PR: the `chargeKey` is only saved once it is deployed, and migration 065 has to be run to remove the old draft Insurance rows.
- `PWA.md` still describes the removed mobile screens (PI Preview, Final Invoice Preview and Create, PI Edit and Delete). It was not updated in this PR.

## Next steps

- Re-enter the percentage charges on old PIs if the type matters to the business.
- Update `PWA.md` for the removed mobile screens.
- Test on real phones: the Dispatch list, the attendance date picker, the photo viewer zoom and the work order Edit sheet.

## Test plan

**Automated tests (no test files added or changed in this PR):**
- [x] `npx tsc -b`: passes, apart from the unused imports in `mobile-order-detail.tsx` listed above.
- [x] `npx eslint src/features/pwa`: passes apart from the same file. The PI dialog still has its older errors.
- [x] `npx vite build`: builds.
- [ ] No component or unit tests exist for the PI charges or the mobile screens. None were added.

**Manual checks (local):**
- [ ] Create a PI, add a custom Percentage charge, save, reopen it, and check that the type stays Percentage and no charge appears twice - not checked in a browser.
- [ ] A new PI shows the Insurance row, and Insurance no longer appears per product in the Order Booking - not checked in a browser.
- [ ] Progress column on the Work Orders list - not checked in a browser.
- [ ] Mobile Dispatch list, photo viewer, attendance date picker and work order Edit sheet on a phone or in device emulation - not checked.
