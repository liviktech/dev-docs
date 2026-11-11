# 2026-10-13 - inventory-stock-and-operator-scan (frontend)

## Summary

The New Stock dialog in Inventory only showed thicknesses from the selected product's rates, had no Type field, let users add a thickness from inside the dialog, and did not work out Quantity (SQM) by itself. Product and location dropdowns also showed the code in brackets, and the Lot Code badge did not match the code badges on other pages. Separately, a user with the OPERATOR role landed on a full Work Orders table when all they need is to scan a barcode. This PR adds a Type dropdown that narrows the thickness list, removes "Add Thickness" from the dialog, fills Quantity (SQM) automatically, drops the codes from the dropdowns, reuses the shared code badge for Lot Code, and gives operators a single centred Scan Barcode button.

**Saving the selected Type with the stock lot is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/inventory/components/receive-stock-dialog.tsx` | Added a Type dropdown before Thickness. Thickness now lists only the thicknesses the selected product has rates for under the selected type, and stays disabled until a type is picked. The "+ Add Thickness" option, its popup dialog and its save code were removed. Quantity (SQM) is filled automatically from width x height x pieces. Product and Storage Location dropdown labels no longer show the code in brackets. |
| `src/routes/inventory.tsx` | Product and Location filter labels no longer show the code in brackets. The clickable Lot Code in the stock table now uses the shared `CodeLinkButton`. The Lot Code in the movements table and the Code in the locations table use the same cyan look (not clickable). The same commit also has two small wording edits: "Qty (Pieces)" is now "Qty (Pcs)" and the delete confirm title is now "Delete Stock Lot". The rest of the diff is reformatting only. |
| `src/features/production-scan/operator-scan-panel.tsx` (new) | A centred Scan Barcode button that opens the existing scan dialog. This is all an operator sees. |
| `src/features/production-scan/production-line-page.tsx` | Shows `OperatorScanPanel` instead of the KPI cards, tabs, search and table when the logged in role is OPERATOR. |
| `src/routes/work-orders.tsx` | Same OPERATOR check, placed just before the page's main return. The commit shows a very large diff, but it is almost all quote and semicolon reformatting. The real change is the new import, `user` from `useAuth()`, and the one-line OPERATOR check. |

## Why

The request was to show all thickness options in New Stock, then to show only the thicknesses that belong to the chosen type, to add a Type field, to remove Add Thickness, to fill Quantity (SQM) automatically (5 x 6 x 3 pieces gives 90), to remove codes from the dropdowns and table filters, to make the Lot Code badge look like the code badges on the other pages, and to show operators only a centred scan button. The easy part was the UI. The real work was finding where a thickness and a type are linked. Only a product's rate rows link them, so the Type and Thickness lists are built from those rates.

## How

**Type and thickness lists**: The Type dropdown lists the types the selected product has rates for. If the product has no rates, it lists all active types. The Thickness dropdown then lists only the thicknesses on that product's rates for the chosen type. A thickness already saved on an existing lot is kept in the list so editing does not lose it.

**Quantity (SQM)**: Width x height x pieces is worked out and written into the SQM field every time one of the three changes. The old "Auto: ... SQM" link still works. Viewing a lot does not recalculate anything. The old formula divided by 1,000,000 (treating sizes as mm). That division was removed so 5 x 6 x 3 gives 90 as asked.

**Lot Code badge**: The stock table uses `CodeLinkButton`, the same clickable badge used on the Products, Party and admin pages. It still opens the lot details when clicked.

**Operator view**: The page checks `user.role` (case insensitive) for OPERATOR. If it matches, the page returns only the scan panel. Everyone else gets the same page as before.

## Decisions made

**Type is not saved with the stock lot.** The inventory lot table and API have no glass type field. Saving it needs a database column plus backend changes, so this PR only uses Type to filter the thickness list. When an existing lot is opened, Type starts empty.

**Operators are found by role name.** Roles are created in the app, so there is no fixed list. The check is for a role called OPERATOR. I did not use "only has the barcode scanner right" because it would also catch other limited users by accident.

**Quantity (SQM) is overwritten when the size or pieces change.** If a user types their own SQM and then changes width, height or pieces, the typed value is replaced. I picked this so the field is always in step with the three inputs.

**The SQM formula no longer divides by 1,000,000.** This matches the example given (90) but means real sheet sizes in mm will give very large numbers. It needs a decision on the units.

## Notes

- The thicknesses 16, 18 and 20 mm and the 6 product rates that used them (on ANNE-GLS and TUFF-GLS) were deleted directly in the database with SQL, as a one off. There is no migration for it, so a fresh database seeded from `018_glass_type_thickness_seed.sql` only gets 6 to 14 mm. The backend repo has no code change in this task.
- The operator view only changes the page content. The sidebar still shows the Work Order menu item.
- The editor shows a "FormEvent is deprecated" hint in `receive-stock-dialog.tsx`. It was there before this change.

## Next steps

- Decide the unit for glass width and height, then restore or drop the division in the SQM formula.
- Add a glass type column to the stock lot (database, backend and API) if Type needs to be saved and shown on the lot.
- Decide whether operators should also lose the sidebar and be sent straight to the scan page after login.

## Test plan

**Automated tests (none run):**
- [ ] Type check and lint were not run. The editor showed no remaining errors on the edited files after the last edit.
- [ ] No test files cover these screens, so none were added or run.

**Manual checks (local):**
- [x] Checked in the database that 16, 18 and 20 mm and their 6 rate rows were removed and that a lookup for those values returns no rows.
- [x] Vite reported a parse error in the movements Lot Code cell (a missing `>`). It was fixed in the code, but the page was not reloaded afterwards to confirm.
- [ ] New Stock with a product and type: not opened in the browser. Needs a check that Thickness only lists that type's thicknesses and that Add Thickness is gone.
- [ ] New Stock with height 5, width 6 and 3 pieces: not tried in the browser. It should show 90 in Quantity (SQM).
- [ ] Logged in as OPERATOR: not checked after the last change. The screenshot earlier showed the full Work Orders table, which is what the new panel replaces. It should show only a centred Scan Barcode button.
