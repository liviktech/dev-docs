# 2026-10-28 - storage-location-flow (backend)

## Summary

Stock lots had a storage location, but purchases had no place to set one, finished glass had no storage location at all, and the location list was one mixed list. This PR lets a purchase line carry a storage location that is copied to the stock lot when the line is approved, saves a finished glass location on the work order piece when its last stage is completed, and gives every storage location a type (Raw Glass or Finished Goods) so each screen can list only its own kind. It also adds a "Finished Pieces Produced" count to the Batch Consumption report.

The code compiles (`tsc --noEmit` passes). Migrations 098, 099 and 100 were run by me against the dev database, and the Batch Consumption query was run through the service and returned 43 rows for RUBINE GLASS. Saving a purchase with a location, approving it, and completing a last stage with a location through the API were **not checked live** here.

**A finished glass location column in the reports is not included on purpose** - see "Decisions made" below. It was built and then removed at the requester's request. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/098_storage_location_purchase_and_finished_glass.sql` (new) | Adds a nullable `location_code` to `public.inventory_purchase_item` and a nullable `finished_location_code` to `production.work_order_item`. Adds only, old rows are untouched. |
| `src/database/migrations/099_rubine_storage_locations.sql` (new) | For RUBINE GLASS (P) LTD only: upserts the four locations (FINISHED-STORE-A, FINISHED-STORE-B, RAW-GLASS-A, RAW-GLASS-B) and deactivates every other location of that company. Safe to run again. |
| `src/database/migrations/100_inventory_location_type.sql` (new) | Adds `location_type` (RAW_GLASS or FINISHED_GOODS, default RAW_GLASS) to `master.inventory_location` with a check. Existing codes that start with FINISHED-STORE are set to FINISHED_GOODS. |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts`, `.types.ts` | A purchase line accepts an optional `locationCode` and returns `locationCode` and `locationName`. |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | Saves `location_code` with the line and joins the location name when lines are read. |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | Returns the location on each line, and gives a newly created stock lot the line's location when the line is approved. |
| `src/modules/master/inventory-location/inventory-location.schema.ts`, `.types.ts`, `.repository.ts`, `.service.ts`, `.controller.ts` | Create and update take `locationType`, rows return it, and the list call accepts a `type` filter (`RAW_GLASS` or `FINISHED_GOODS`) next to `activeOnly`. |
| `src/modules/production-scan/production-scan.schema.ts`, `.controller.ts` | The complete stage body accepts an optional `finishedLocationCode` and the controller passes it on. |
| `src/modules/production-scan/production-scan.service.ts` | On the last stage of a piece, checks the location is an active Finished Goods location and saves it on the piece. Other stages ignore it. The piece detail now returns the saved location code and name. |
| `src/modules/production-scan/production-scan.repository.ts`, `.types.ts` | Reads the piece's finished location with its name, checks a location is active and Finished Goods, and saves it. |
| `src/modules/report/inventory/inventory-report.repository.ts` | Counts the pieces cut per report row (`finished_pieces`). The rest of the diff in this file is lines re-wrapped by the formatter, no real change. |
| `src/modules/report/inventory/inventory-report.service.ts` | Returns `finishedPieces` for each row, empty when the row has cuts made before piece sizes were saved. |

## Why

The client wants stock kept in named raw glass and finished goods stores. Raw glass arrives through purchases, so the purchase line is the natural place to say where it goes, and finished glass needs a place when the piece leaves production. The Finished Goods and Raw Glass report spec also lists "Finished Pieces Produced", which the Batch Consumption report did not have. The real work was the location type, because without it a raw glass dropdown would also offer finished stores.

## How

**Purchase line location**: The location is saved on the purchase line, not on a lot, because the lot does not exist until approval. When approval creates a new lot, the lot gets the line's location through the existing lot create code. If approval adds sheets to a lot that already exists, that lot keeps its own location.

**Finished glass location**: The complete stage call takes the location with the stage. The service works out whether the stage being completed is the piece's last real stage (the same check that already marks the piece produced). Only then does it validate and save the location, in the same transaction as the complete. The location is stored on the work order piece (`work_order_item`), so every piece records its own place.

**Location type**: One column, and the list call filters on it. The web screens ask for the type they need, so raw glass screens never see finished stores and the other way round. The server also refuses a finished location that is not a Finished Goods one.

**Finished Pieces Produced**: One stock cut is one piece, so the count is the number of sized cuts in the row.

## Decisions made

**Old locations are deactivated, not deleted.** The request was to keep only four names. Stock lots still point at the old locations and the database refuses to delete a location a lot uses (restrict key), and the app never hard deletes master rows. They disappear from the dropdowns and the Active list, and lots keep their history.

**A location type column, not a name prefix.** Filtering on codes like RAW-GLASS and FINISHED-STORE would have needed no migration, but a location added later with another code would land in the wrong list. The type is chosen when the location is added.

**The server accepts the finished location as optional, the web screen makes it required.** The phone app also calls complete stage and does not ask for a location. Requiring it on the server would break completing a last stage on the phone. A piece finished on the phone just has no location.

**Migration 099 is tied to the company name RUBINE GLASS (P) LTD.** It changes data, not shape, and the other tenants in the dev database (test and demo companies) were not asked about, so they are untouched. It does nothing if no company has that name.

**The finished glass location is not in the reports.** It was added as a column on Batch Consumption and then removed at the requester's request. The location is kept on the piece and shown on the piece page.

**It does not create finished goods stock.** The location is only recorded on the piece.

## Notes

- Run `npm run migrate` and restart the backend. The piece detail query now reads `finished_location_code`, and the purchase query reads `location_code`, so both fail until 098 is applied. The location type filter needs 100.
- Migrations 098, 099 and 100 were already applied to the shared dev database during this work, so other people on that database get the four new locations and the deactivated old ones straight away.
- The Offcut Register already showed a Storage Location from the lot. That code is unchanged.

## Next steps

- Add the same finished glass location question to the phone app complete button, or make it required on the server once the phone screen asks for it.
- Decide whether finished glass should become real finished goods stock at the chosen location.
- Move or clean up the stock lots that still point at the deactivated locations if the client wants them under the new stores.

## Test plan

**Automated tests (no new tests added, type check passing):**
- [x] `npx tsc --noEmit` in the backend: passes.
- [ ] No unit or API tests were written for the purchase location, the finished location rule or the location type filter. The backend has no test files for these modules to extend.

**Manual checks (local):**
- [x] `npm run migrate` applied 098, 099 and 100 to the dev database. RUBINE GLASS now has four active locations (FINISHED-STORE-A and B as FINISHED_GOODS, RAW-GLASS-A and B as RAW_GLASS), and the five old ones are inactive.
- [x] Ran the Batch Consumption query through the service for RUBINE GLASS: 43 rows, each with `finishedPieces`.
- [ ] Creating a purchase with a line location, approving it, and seeing the new lot take that location was not checked against a database.
- [ ] Completing a last stage through the API with a finished location, with a raw glass location (should be refused), and with none (should still complete) was not checked.
- [ ] The location list call with `type=RAW_GLASS` and `type=FINISHED_GOODS` was not called over HTTP.
