# Testing flow: Inventory, Offcut, Work Order and Cutting entry (wastage and offcut)

This guide tests the full path of raw glass: buying stock, creating and approving a work order, the system using a sheet or an offcut, and the operator entering finished glass, wastage and offcut on the Cutting stage. Follow the steps in order. Write the real result next to each check box.

## 0. Before you start

- [ ] Backend migrations are run (`npm run migrate`), including `094_inventory_offcut_code.sql` and `096_work_order_process_cutting_entry.sql`. Restart the backend after that.
- [ ] Log in with a user who has View and Control on Inventory, Work Order, Order Booking and Reports (an admin user is fine).
- [ ] A product exists for the test glass (for example Annealed Glass 8 MM), and a storage location exists in Inventory locations.
- [ ] A party (customer) exists.
- [ ] Use a fresh test glass type if you can, so old stock does not mix with the test numbers below.

## 1. How the system picks glass (read this once)

- Sizes are in mm and areas in sqm. A 3660 x 2440 sheet is 8.930 sqm.
- Pieces are cut one at a time, biggest first.
- A piece uses the smallest leftover (offcut) it fits in. If no offcut fits, it takes a new full sheet.
- After a cut, one leftover rectangle is saved: the bigger of the two strips left. The smaller strip is not saved and counts as waste.
- Stock is taken when the work order is **approved**, not when the glass is cut.

## 2. Test data used in this guide

| Item | Value |
|---|---|
| Glass | Annealed Glass 8 MM, clear |
| Sheet size | 3660 x 2440 mm |
| Stock quantity | 1 sheet (8.930 sqm) |
| Test piece | 1000 x 800 mm (0.800 sqm), quantity 1 on each work order |

Expected cuts with these numbers:

| Work order | Piece comes from | Leftover saved | Leftover area | Strip not saved (waste) |
|---|---|---|---|---|
| WO-A | New sheet 3660 x 2440 | 3660 x 1640 | 6.002 sqm | 2.128 sqm |
| WO-B | The offcut 3660 x 1640 | 3660 x 840 | 3.074 sqm | 2.128 sqm |

## 3. Step by step

### Step 1. Add stock

1. Open **Inventory**, then the **Stock** tab, then **New Stock**.
2. Add the glass: Annealed Glass 8 MM, clear, sheet size 3660 x 2440, quantity 1 sheet, with a storage location.
3. Save.

Check:
- [ ] The lot appears in Stock with a lot code like `LOT-YYYYMMDD-NNN`.
- [ ] Available sheets show 1 and available area shows 8.930 sqm.
- [ ] No offcuts yet (cutoff count 0).
- [ ] **Movements** tab shows a RECEIPT for the lot.

### Step 2. Create and approve WO-A

1. Create an order booking for the party with one item: Annealed Glass 8 MM, 1000 x 800 mm, quantity 1. Create the work order from it.
2. On the **Work Order** page, open the row menu and choose **Approve**, then press **Approve** in the dialog.

Check:
- [ ] The work order status becomes In Progress.
- [ ] **Inventory > Stock**: available sheets 0 for the lot, and the cutoff count is 1.
- [ ] **Inventory > Movements**: an ISSUE row for the lot, linked to WO-A, labelled "New sheet".
- [ ] The lot now holds one offcut of 3660 x 1640 mm (6.002 sqm).
- [ ] **Reports > Raw Glass > Offcut Register** (after step 7) lists this offcut with status **Available** and source work order WO-A.

### Step 3. Create and approve WO-B (uses the offcut)

1. Create a second order booking and work order with the same piece (1000 x 800 mm, quantity 1).
2. Approve it.

Check:
- [ ] **Movements**: an ISSUE linked to WO-B, labelled "Offcut 3660 x 1640" (not "New sheet").
- [ ] The 3660 x 1640 offcut quantity goes to 0 and it is marked CONSUMED in stock.
- [ ] A new offcut 3660 x 840 mm (3.074 sqm) is saved, source work order WO-B.
- [ ] Available sheets are still 0 (a new sheet was not used).

### Step 4. Insufficient stock (negative test)

1. Create a third work order that needs more glass than the remaining offcut can give (for example a 3000 x 2000 mm piece when no sheet is left).
2. Try to approve it.

Check:
- [ ] Approval is refused with an insufficient stock message.
- [ ] No stock movement is saved and the work order stays unapproved.

### Step 5. Run the Cutting stage on WO-A and enter the Cutting entry

1. Open the piece of WO-A from the Work Order page (the piece page with the stage steps and Execution Log).
2. On the **Cutting** stage press **Approve / Start Stage**.

Check:
- [ ] The Execution Log shows Cutting with a **Started** time and the operator name. Note the Started date and time: ______

3. Press **Complete Stage**.

Check:
- [ ] The **Cutting entry** box opens. The stage is not completed yet.
- [ ] The box has: Finished glass (sqm), Wastage (sqm), Offcut length (mm), Offcut width (mm).
- [ ] Wastage has no red star. The other three have a red star.
- [ ] **Save and complete stage** is disabled while any required field is empty.
- [ ] Typing a negative number does not enable Save.

4. Enter Finished glass `0.8`, Wastage `2.128`, Offcut length `1640`, Offcut width `3660`, and press **Save and complete stage**.

Check:
- [ ] The stage completes and the next stage (Grinding or similar) is open.
- [ ] The Execution Log shows Cutting as Completed with a Completed time.
- [ ] Click the completed Cutting stage in the stepper: the four values show above the Execution Log (Finished glass 0.8, Wastage 2.128, Offcut length 1640, Offcut width 3660).

### Step 6. Wastage is optional, and other stages are not affected

1. On WO-B, start and complete the Cutting stage again. This time leave **Wastage** empty and enter Finished glass `0.8`, Offcut length `840`, Offcut width `3660`.

Check:
- [ ] Save becomes enabled with wastage empty and the stage completes.
- [ ] The completed Cutting stage shows Wastage as `-`.
2. Complete the next stage (for example Grinding) on any piece.
- [ ] No Cutting entry box appears for a stage that is not Cutting.

### Step 7. Server rule (optional, API check)

1. Call the complete endpoint for a Cutting stage with no entry (for example from a REST tool: `POST /api/v1/production-scan/<item id>/complete` with an empty body).

Check:
- [ ] The server refuses with the message "Enter the finished glass and the offcut before completing Cutting" (code `SCAN_CUTTING_ENTRY_REQUIRED`).
- [ ] The stage stays In Progress.

### Step 8. Return and reject cases

1. Reject (cancel) WO-B before its piece is cut, if your role allows it, and check the stock.

Check:
- [ ] The offcut 3660 x 1640 is back to quantity 1 and Available.
- [ ] The 3660 x 840 offcut made by WO-B is removed.
- [ ] A RETURN row appears in Movements.
- [ ] The Offcut Register no longer lists WO-B as a consuming work order of the 3660 x 1640 offcut.
2. Reject a single piece for breakage on a work order whose piece was already cut.
- [ ] A replacement piece with barcode ending `-R1` is created and new stock is taken for it.
- [ ] The broken piece's glass stays consumed (it is not returned to stock).

## 4. Database checks (optional, read only)

Replace the work order code with your own.

```sql
-- the cutting entry saved on the Cutting stage of each piece
SELECT woi.barcode_value, wop.status, wop.started_at, wop.completed_at,
       wop.finished_sqm, wop.wastage_sqm, wop.offcut_length, wop.offcut_width
FROM production.work_order_process wop
JOIN production.work_order_item woi ON woi.id = wop.work_order_item_id
JOIN production.work_order wo ON wo.id = woi.work_order_id
WHERE wo.work_order_code = 'WO-A-CODE' AND wop.work_center_code = 'CUT';

-- stock cuts of a work order (one ISSUE per piece)
SELECT m.movement_type, m.piece_width, m.piece_height, m.leftover_width, m.leftover_height, m.offcut_id
FROM public.inventory_movement m
JOIN production.work_order wo ON wo.id = m.work_order_id
WHERE wo.work_order_code = 'WO-A-CODE' ORDER BY m.performed_at;

-- offcuts and their readable codes
SELECT offcut_code, width, height, quantity, status FROM public.inventory_lot_offcut ORDER BY created_at;
```

## 5. Known behaviour to expect (not bugs of this change)

- Older work orders (approved before per piece cutting) have one stock record for the whole order, with no piece sizes.
- The stored offcut status becomes CONSUMED at work order approval, not after cutting. The Offcut Register works out Reserved and Consumed from the Cutting stage instead.
- A chain of cuts from one big sheet saves every leftover as an offcut row, and all but the last are marked CONSUMED straight away.
- The entered offcut in the Cutting entry box is recorded and reported only. It does not add stock.

## 6. Result summary

| Step | Pass / Fail | Notes |
|---|---|---|
| 1. Add stock | | |
| 2. WO-A approve | | |
| 3. WO-B approve (uses offcut) | | |
| 4. Insufficient stock | | |
| 5. Cutting entry box | | |
| 6. Wastage optional | | |
| 7. Server rule | | |
| 8. Return and reject | | |
