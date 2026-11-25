# 2026-10-28 - storage-location-flow (frontend)

## Summary

The Storage Location field and the Storage Locations tab were hidden, the Purchase page had no location, and finishing a piece gave no way to say where the finished glass goes. This PR brings back the Storage Locations tab with a location type (Raw Glass or Finished Goods), shows a Storage Location dropdown on each Purchase row and on New Stock (raw glass locations only), and opens a "Store finished glass" box when Complete Stage is pressed on a piece's last stage (finished goods locations only). It also adds a "Finished Pieces Produced" column to the Batch Consumption report, and includes three small fixes: the dialog close button no longer sits on top of a long title, the Cutting entry box has some spacing and scrolls on a short screen, and the Sales Visits page opens on the Immediate tab.

The code compiles (`tsc --noEmit -p tsconfig.app.json` passes). I did **not open any of these screens in the browser**, so the dropdowns, the new tab, the Store finished glass box and the report column were not checked live.

**A finished glass location column in the reports is not included on purpose** - see "Decisions made" below. It was built and then removed at the requester's request. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/inventory/locations-tab.tsx` (new) | The Storage Locations tab: Total, Active and Inactive cards, Active, Inactive and All filters, search, and a table with Code, Location Name, Type, a status switch and Edit and Deactivate actions. Built from the old commented out code. |
| `src/routes/inventory.tsx` | Adds the Storage Locations tab after Movements and an Add Location button in the header when it is open. The tab is kept in the URL. The old commented out block (about 330 lines) is removed because the tab replaces it. |
| `src/features/inventory/components/inventory-location-dialog.tsx` | Add and Edit form gets a Location Type field (default Raw Glass). |
| `src/features/inventory/types.ts`, `api.ts`, `queries.ts` | Add the location type and its labels, a `type` filter on the location list call, and an optional type on `useInventoryLocations`. |
| `src/features/inventory/purchase-types.ts` | A purchase line carries `locationCode` and `locationName`. |
| `src/features/inventory/components/purchase-dialog.tsx` | New Storage Location column after Finish on each row, listing active Raw Glass locations. Saved with the line and filled back in when a purchase is edited. The table minimum width grew for the extra column. |
| `src/features/inventory/components/receive-stock-dialog.tsx` | The Storage Location field is switched back on (it was commented out) and lists active Raw Glass locations. |
| `src/features/production-scan/components/finished-location-dialog.tsx` (new) | The "Store finished glass" box with one required dropdown of active Finished Goods locations. |
| `src/features/production-scan/scan-detail-page.tsx` | On a piece's last stage, Complete Stage opens the new box first and sends the location with the complete call. The Stages Completed card reads "All stages done, Stored at <location>" afterwards. |
| `src/features/production-scan/api.ts`, `queries.ts`, `types.ts` | Send the finished location with the complete call and add the location to the piece type. |
| `src/features/reports/inventory/api.ts`, `batch-consumption-report.tsx` | New Finished Pieces Produced column in the table, the Excel file and the PDF, with a total. The Total rows in both downloads were one column off to the left and are now lined up. |
| `src/components/ui/dialog.tsx` | The dialog header leaves room on the right for the close button and long titles wrap. This applies to every dialog in the app. |
| `src/features/production-scan/components/cutting-entry-dialog.tsx` | The fields area scrolls on a short screen and has some vertical padding. The Wastage field is still there and still optional. Only a comment changed otherwise. |
| `src/features/sales-visits/sales-visits-page.tsx` | The order type filter now starts on Immediate. All is stored as `ALL` in the URL and sent to the server as no filter. |

## Why

The client wants raw glass and finished glass kept in separate named stores. Raw glass arrives through purchases, so the Purchase row is where its store is chosen, and finished glass needs a store chosen at the end of production. The Storage Locations tab was the only way to add locations, and it was hidden, so the dropdowns would have been empty. The type keeps the two dropdowns from offering each other's stores. The report column comes from the Finished Goods and Raw Glass spec. The close button problem was seen on a Cutting entry title that was long, and the Sales Visits change was asked for so the Immediate orders show first.

## How

**Location type in the dropdowns**: Each screen asks the server for the one type it needs (`useInventoryLocations(true, "RAW_GLASS")` or `"FINISHED_GOODS"`), so nothing is filtered in the browser. The query key includes the type so the two lists are cached separately.

**Last stage check**: The piece page already knows the next stage. If there is none, Complete Stage opens the location box. If the last stage is also Cutting, the Cutting entry comes first and is held while the location is chosen. The stage only completes when the location is saved. The stale stage check still runs before the complete call.

**Add Location button**: It lives in the page header like New Purchase, so the page passes an open flag to the tab, and the tab owns the Add and Edit form.

**Sales Visits tab**: Because "no filter" used to be an empty value, it could not be told apart from the default. All is now its own value and the default is Immediate. Clicking a tab sets that value directly.

## Decisions made

**Old locations stay in the table as inactive.** The Storage Locations tab opens on Active, so the old ones are hidden there, and the Inactive and All filters still show them.

**The finished glass location is required in the box, but the phone app is unchanged.** The phone complete button does not ask for a location, and the server accepts a missing one, so a piece finished on the phone has no location.

**The finished glass location is not a report column.** It was added to Batch Consumption and removed on request. It is only shown on the piece page.

**The dialog close button fix is global.** Changing the shared dialog header was simpler and safer than adding space dialog by dialog. The header just has more room on the right.

**The Wastage field stays in the Cutting entry box.** It was taken out and put back during this work, so the box behaves as before.

## Notes

- Run the backend migrations first (`npm run migrate`, migrations 098, 099 and 100 in the backend), or the location type, the purchase location and the piece location will fail.
- The New Stock Location field was only commented out, so its form data and save code already existed. Only the field and the locations call were switched back on.
- The Stock list still hides the Location column and the All Locations filter. They were not part of this change.
- Local build tools showed "can be written as" class name hints on `purchase-dialog.tsx`. They are warnings that were already there for the existing table, not errors.

## Next steps

- Open the Purchase page, the New Stock form, the Storage Locations tab and a piece's last stage in the browser and check each dropdown lists only its own type.
- Add the finished location question to the phone app complete button.
- Decide whether to show the Location column and filter on the Stock list.

## Test plan

**Automated tests (no new tests added, type check passing):**
- [x] `npx tsc --noEmit -p tsconfig.app.json` in the frontend: passes.
- [ ] No component or unit tests were written for the new tab, the dropdowns or the box. There is no test setup for these screens in this repo to extend.

**Manual checks (local):**
- [ ] Inventory > Storage Locations tab, Add Location with each type, edit, and deactivate: not opened in the browser.
- [ ] Purchase page, Add Row: the Storage Location dropdown should list only Raw Glass Store A and B. Not checked live.
- [ ] New Stock form: the Storage Location dropdown should list only the Raw Glass stores. Not checked live.
- [ ] Complete Stage on the last stage of a piece: the Store finished glass box should list only Finished Goods Store A and B, and Save should stay disabled until one is picked. Not checked live.
- [ ] Reports > Raw Glass > Batch Consumption: the Finished Pieces Produced column and the aligned Total rows in Excel and PDF. The query behind it was run in the backend and returned rows, but the screen and downloads were not opened.
- [ ] Sales Visits page opens on the Immediate tab and the All tab still lists everything. Not checked live.
