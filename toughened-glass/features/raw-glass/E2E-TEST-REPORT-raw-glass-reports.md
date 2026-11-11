# End to end test report: Raw Glass reports (Batch Consumption and Offcut Register)

Date: 2026-10-28. Requirement document: `03_Finished_Goods_Raw_Glass_Module.pdf` (Finished Goods and Raw Glass Module). Company tested: RUBINE GLASS (P) LTD, dev database. No code was changed and no data was written during this testing.

## 1. Result in one minute

The numbers in both reports are correct against the raw data. Every calculation I recomputed from the stock, work order and execution log tables matched the report, with one exception that is a design gap between the two reports (finding 1). The reports follow the requirement document for columns and for all filters except Manufacturing Stage. The main weaknesses are not wrong numbers, they are missing or thin data: most old rows have no piece sizes, almost no Cutting entries exist yet, and most stock has no storage location.

| Area | Result |
|---|---|
| Batch Consumption columns and formulas | Pass |
| Production Date matches the execution log | Pass (43 of 43 rows) |
| Offcut Register columns, Offcut ID, area, status, generated date | Pass |
| Filters in the requirement document | Pass except Manufacturing Stage (not built) |
| Both reports agree with each other | Fail on 1 row (finding 1), 1 rounding difference |
| Storage Location is filled in (requirement says Required) | Weak (16 of 21 offcuts blank) |
| Screens, downloads, permissions, live HTTP | Not tested, see section 6 |

## 2. What the report page is for

The Reports menu has a group called **Raw Glass** with two reports. They answer two different questions about the same glass.

**Raw Glass Batch Consumption and Cutting Utilisation** answers: for each raw glass batch (lot) that a work order cut from, how much glass went in, how much became finished pieces, how much went back to stock as a reusable offcut, and how much was lost as waste. Cutting Utilisation is the finished area divided by the raw area used, so it shows how much of each sheet became real product. It is used to trace a batch to the work orders that used it and to see cutting yield.

**Reusable Offcut Register** answers: which leftover pieces exist, where are they, which batch and work order made them, are they still free, and which work order used them. Each offcut is treated as its own piece of stock with its own ID (for example OFC-000021) so the team can find it, reserve it, and reuse it.

The two reports are linked but separate on purpose, as the requirement says: batch traceability, cutting results and offcut stock are kept connected but are not the same table.

## 3. Where the data comes from

Nothing is typed into the report. Everything is read from records created by the normal work flow.

### 3.1 Batch Consumption (one row per production date, work order, line, stage and raw glass batch)

| Column | Where it comes from |
|---|---|
| Production Date | The day the piece's Cutting stage **Started** in the work order execution log (India time). If it never started, the completed day. If the work order never reached Cutting, the work order approval date. |
| Work Order No | `production.work_order` work order code. |
| Work Order Line / Product | The work order item's product and thickness, product name from the product master. |
| Manufacturing Stage | The work center name of the Cutting stage (always Cutting today, because raw glass is only taken there). |
| Raw Glass Batch No | The stock lot code (`inventory_lot.lot_code`, like LOT-20261026-160). |
| Glass Specification | Product name, thickness and colour of the lot (product and glass colour masters). |
| Raw Glass Area Consumed | The stock movement of type ISSUE for the piece: the area of the new sheet it was cut from, or the area of the offcut it was cut from. For old whole-order records, the stored consumed area. |
| Finished Pieces Produced | The number of piece cuts in the row. One ISSUE movement is one piece. |
| Finished Glass Area | What the operator typed as "Finished glass" in the Cutting entry. If nothing was typed, the area of the cut piece (width x height). |
| Reusable Offcut Area | What the operator typed as offcut length x width in the Cutting entry. If nothing was typed, the leftover size saved on the stock movement. |
| Waste Area | What the operator typed as wastage. If nothing was typed, raw consumed minus finished minus the stock leftover, never below 0. |
| Cutting Utilisation (%) | Finished Glass Area divided by Raw Glass Area Consumed, times 100. |

A cut that was reversed (a RETURN movement points at it) is left out. Cuts made before piece sizes were recorded show only the consumed area, and the finished, offcut, waste, pieces and utilisation cells are blank.

### 3.2 Offcut Register (one row per stock offcut record)

| Column | Where it comes from |
|---|---|
| Offcut ID | System generated code OFC-nnnnnn on `inventory_lot_offcut`, unique. |
| Generated Date | The Cutting Started day of the piece that made the offcut (execution log), else the day the offcut was saved. |
| Source Work Order No | The work order whose cut left this offcut. |
| Raw Glass Batch No, Glass Specification | From the lot the offcut belongs to. |
| Length, Width, Offcut Area | Stored size in mm, area is length x width divided by 1,000,000. |
| Storage Location | The location of the lot (`inventory_location`). Blank if the lot has none. |
| Status | Worked out when the report runs, not stored: **Available** if a free piece remains, **Reserved** if an approved work order has taken it but its Cutting stage is not complete, **Consumed** otherwise. |
| Consuming Work Order | Every work order with a non-reversed stock movement against this offcut. |

## 4. The 100% intended work flow

This is the full path the requirement describes, in order, with what each step writes and what the reports show.

1. **Set up masters.** Products, glass types, colours and storage locations exist. Raw glass stores (Raw Glass Store A and B) are used for stock and purchases.
2. **Raw glass comes in.** Either a Purchase is created and approved (Inventory > Purchase), or stock is added by hand (Inventory > Stock > New Stock). Each approval creates a stock lot with a lot code, a sheet size, a quantity and now a storage location, and writes a RECEIPT movement. Nothing appears in the reports yet.
3. **Order and work order.** An order booking with a product and piece sizes becomes a work order. The work order is approved by the admin.
4. **Stock is taken at approval, not at cutting.** For every piece, biggest first, the system uses the smallest offcut the piece fits in, and if none fits it takes a new full sheet. It writes one ISSUE movement per piece with the piece size and the leftover size, and saves one leftover rectangle (the bigger of the two strips left) as a new offcut with an Offcut ID. The smaller strip is not saved and ends up as waste. At this point the Offcut Register already lists the new offcut and the Batch Consumption report already has a row (from the movement sizes).
5. **Production on the floor.** The operator starts the Cutting stage of the piece. The start time becomes the Production Date and the Generated Date.
6. **Cutting entry.** The operator completes Cutting. A box asks for finished glass, offcut length and width (required, 0 means none) and wastage (optional). The stage cannot complete without it. These values are saved on the Cutting stage and replace the calculated finished, offcut and waste numbers in Batch Consumption.
7. **Later stages.** The piece moves through the other stages. On its last stage the operator chooses where the finished glass is stored.
8. **Offcut life.** A free offcut shows Available. When a later work order is approved and takes it, it shows Reserved with that work order, and it shows Consumed once that piece's Cutting stage is complete. A second work order that used the same offcut row is listed next to the first.
9. **Corrections.** If a work order is rejected or cancelled before its cut, the stock movement is returned (RETURN), the offcut is Available again and the consumption row disappears. If one piece breaks after cutting, a replacement piece (barcode ending -R1) takes new stock, and the broken piece's glass stays consumed.
10. **Reading the reports.** Reports > Raw Glass > Batch Consumption and Offcut Register, filtered by date, work order, batch, specification, location, status or source work order, with Excel and PDF downloads of every matching row.

Open items in the requirement document that the intended flow does not settle: a minimum reusable offcut size, the final list of offcut statuses (the document mentions Scrapped), and whether the entered offcut should be posted into stock.

## 5. How I tested

I ran checks against the dev database for RUBINE GLASS using the real report code (the service that the screens call) and read only SQL on the raw tables. For each check I recomputed the expected value independently from the raw stock, work order and execution log tables and compared it with what the report returned. The database held 43 Batch Consumption rows (9 with piece sizes, 34 without), 21 offcuts, 25 stock lots, 58 ISSUE movements and 121 completed Cutting stages. I did not create, change or delete any data.

### 5.1 Batch Consumption results

| Check | Result |
|---|---|
| Columns in the requirement are all present, plus Finished Pieces Produced | Pass |
| Cutting Utilisation = finished / consumed x 100 on every sized row | Pass, 0 mismatches in 9 rows |
| Finished area never larger than consumed area | Pass |
| No row with zero consumed area | Pass |
| Production Date is a date in the work order's Cutting log or approval date | Pass, 43 of 43 |
| Manufacturing Stage is Cutting on every row | Pass |
| Finished Pieces total equals live piece cuts with sizes | Pass, 22 = 58 live ISSUE cuts minus 36 without sizes |
| Reversed cuts are left out | Pass (no reversed ISSUE exists in the data, so this was checked by rule only) |
| A work order using several batches and a batch used by several work orders | Pass, 7 work orders use more than one batch and 7 batches feed more than one work order, each shown as its own row |
| Work Order filter, Batch filter | Pass |
| Date range filter (same day, both ends included) | Pass |
| Specification filter | Pass, the 11 specifications add up to all 43 rows |
| No match shows an empty result | Pass |
| Paging (page 1 and 2, 10 per page, total 43, no row repeated) | Pass |
| Rows add up: consumed = finished + offcut + waste | 8 of 9 sized rows pass, 1 fails (finding 1) |
| Manufacturing Stage filter in the requirement | **Not built** (finding 5) |

### 5.2 Offcut Register results

| Check | Result |
|---|---|
| Columns in the requirement are all present | Pass |
| Offcut ID is unique and in the format OFC-nnnnnn | Pass, 21 unique, none blank |
| Offcut Area = length x width / 1,000,000 | Pass, 0 mismatches |
| Source work order, batch and specification filled on every row | Pass |
| Status recomputed from raw movements and Cutting completion | Pass, 0 mismatches on 21 offcuts (6 Available, 12 Reserved, 3 Consumed) |
| Reserved and Consumed rows always name a consuming work order, Available rows never do | Pass |
| Status filter gives Available 6, Reserved 12, Consumed 3, total 21 | Pass |
| Generated Date equals the Cutting start of the source piece | Pass, 0 mismatches |
| Storage Location filled in (requirement says Required) | **Weak**, 16 of 21 blank (finding 4) |
| Scrapped status (requirement mentions it) | **Not built** |

### 5.3 Both reports against each other

For each work order and batch with piece sizes, the offcut area in Batch Consumption was compared with the offcut area in the Offcut Register. 6 of 9 rows match exactly, 1 row has no offcut in either report (0 in Batch Consumption, no offcut in the register), 1 differs by 0.001 from rounding, and 1 row differs a lot (finding 1).

## 6. What I did not test

- **The screens in the browser.** Layout, Pascal case text, empty messages, the Reset button, paging footer, the "Could not load the report" panel and the new Finished Pieces Produced column were not opened.
- **Excel and PDF downloads.** Not generated. The Total row alignment fix was read in the code only.
- **The live HTTP API.** I tried to log in to the running backend to call the report endpoints, and the login attempt was blocked, so I did not work around it. The report code was run directly instead.
- **Permissions.** No check with a user without the Reports View right.
- **Writing the flow.** I did not create a purchase, a work order or a Cutting entry, so the transitions Available to Reserved to Consumed, the reject and return flow and the replacement piece were checked by recomputing the rules on existing data, not by performing them. The manual steps in `TESTING-inventory-offcut-work-order-flow.md` and `TESTING-raw-glass-reports.md` still need to be run once to confirm them.
- **Phone app.** Completing stages from the phone was not tested.

## 7. Findings

### 1. The two reports disagree on offcut for the same cut, and one row does not add up (Medium)

For WO-20261028-165 (batch LOT-20261026-160): raw consumed 15.918, finished 0.148, offcut 0.02, waste 7.302. The row is short by 8.448. The Offcut Register shows the offcut from this same cut as OFC-000021, 3611 x 2345 mm = 8.468 sqm.

The cause: the operator's Cutting entry says offcut 100 x 200 mm (0.02 sqm), but the stock cut saved a leftover of 3611 x 2345 mm. Batch Consumption shows the typed offcut (0.02), but when wastage is left empty it works the waste out from the stock leftover (8.468), so the row does not balance. Wastage was empty here, so this is **not** caused by a typed wastage. This corrects what I said earlier when I explained that row from the screenshot: I suggested a typed wastage or a partly cut piece, and the data shows it is this mismatch between the typed offcut and the stock leftover.

This is the "entered offcut is recorded only, it is not posted into stock" design decision showing up in the report. The same pattern is in the other sized cutting entries (100 x 200 mm and 10 x 10 mm look like test values). Decide which offcut is the truth, the typed one or the stock one, and use the same one in waste and in both reports.

### 2. Almost no Cutting entries exist yet (Medium)

Only 3 of 121 completed Cutting stages have an entry. The other 118 were completed before the entry existed (migration 096 was applied on 2026-10-27 15:32 UTC), so they use the calculated numbers. One more stage (WO-20261022-161, piece RB-WO-20261022-161-01-001) completed at 15:51 UTC, 19 minutes after the migration, with no entry. It was most likely completed by a backend that had not been restarted with the new rule, but I could not prove that from the data. Worth a quick check that a restarted backend refuses to complete Cutting without the entry (the manual test in the testing guide, step 7).

### 3. Most rows have no piece sizes, so most cells are blank (Low to Medium)

34 of 43 rows (26 work orders approved between 5 and 21 October) are older whole-order stock records. They show only Raw Glass Area Consumed, and Finished Pieces, Finished Glass Area, Offcut, Waste and Utilisation are blank. Only 9 rows, from 23 to 28 October, are fully filled in. This is known and expected, and it means the report will look mostly empty on this database until more work orders are cut piece by piece.

### 4. Storage Location is blank on most stock (Medium)

The requirement says Storage Location is Required for an offcut. 16 of 21 offcuts and 19 of 25 stock lots have no location, because the field was hidden until recently and purchases could not set one. Nothing enforces it. The new Purchase row location and the Raw Glass location dropdown now give a way to fill it in, so new stock should have it from now on. Old lots would still be blank.

### 5. Manufacturing Stage filter is missing (Low)

The requirement lists Manufacturing Stage as a filter. The API accepts a `stage` value but the query ignores it, and the screen has no stage dropdown. Today every row is Cutting, so nobody notices. It becomes a gap when another stage can consume raw glass.

### 6. Specification dropdown lists glass that was never cut (Low)

2 of the 11 specifications (Annealed Glass, Annealed Glass 10 mm) return no rows, because the dropdown is built from all stock lots and not only the ones that were consumed. Picking one shows the empty message.

### 7. Rounding and old comments (Low)

- WO-20261010-130 shows 9.608 in Batch Consumption and 9.607 in the Offcut Register, because one rounds each row and the other adds unrounded pieces.
- A comment in `inventory-report.schema.ts` still says the date filter uses the work order approval date. It actually uses the Production Date (Cutting start).
- 15 of 21 offcuts have no Cutting log (older data), so their Generated Date is the day they were saved. That is the intended fallback.

## 8. Requirement document, point by point

| Requirement | Status |
|---|---|
| Batch Consumption has all 11 report columns | Done |
| Filters: date range, work order, batch, specification | Done |
| Filter: manufacturing stage | Not done (finding 5) |
| A work order can use many batches and a batch many work orders, recorded at batch level | Done |
| Utilisation is finished area / raw area x 100, and is a yield measure only | Done |
| Units are consistent (sqm), and rows are not assumed to sum exactly | Done, 1 row does not sum (finding 1) |
| Cutting pattern optimisation is out of scope | Respected, none built |
| Offcut Register has all 11 columns, unique system ID | Done |
| Filters: generated date, specification, storage location, status, source work order | Done |
| Statuses Available, Reserved, Consumed, Scrapped (final list to be confirmed) | Three done, Scrapped not built |
| A reused offcut is linked to its consuming work order and not counted as available again | Done (verified on existing data) |
| Minimum reusable offcut size or quality criteria | Open, not defined by the client, so every leftover is listed |
| Storage Location required | Column exists, not enforced (finding 4) |
| Use existing manufacturing and cutting entry flows, no duplicate forms | Done, the entry is a small box on the existing Complete Stage step |

## 9. To finish the end to end test

Run these once, with the backend restarted and migrations 098, 099 and 100 applied:

1. Follow `TESTING-inventory-offcut-work-order-flow.md` steps 1 to 8 (add stock, approve two work orders, cut, enter the Cutting entry, reject one). Use the same offcut sizes in the Cutting entry as the stock cut, so finding 1 does not hide other problems.
2. Follow `TESTING-raw-glass-reports.md` sections 1 to 5 in the browser, and download the Excel and PDF files.
3. Create a Purchase row with a Raw Glass location, approve it, and confirm the lot and its later offcuts show that location in the Offcut Register.
4. Complete the last stage of a piece, choose a Finished Goods location, and confirm it shows on the piece page.
5. Open the reports with a user who has no Reports right.
