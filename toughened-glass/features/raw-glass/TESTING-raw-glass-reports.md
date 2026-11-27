# Testing flow: Raw Glass report module

This guide tests the two Raw Glass reports (Batch Consumption and Offcut Register). Run the flow in `TESTING-inventory-offcut-work-order-flow.md` first, so the reports have real data. The numbers below match the test data in that guide.

## 0. Before you start

- [ ] The backend is restarted after running migrations 094 and 096.
- [ ] WO-A and WO-B from the other guide are approved, and their Cutting stages are started and completed with the Cutting entry. Note the Cutting **Started** date of each piece: WO-A ______, WO-B ______.
- [ ] You are logged in with the Reports View right.
- [ ] Open **Reports** in the sidebar. The menu opens and shows the groups: Sales Rep, Production Plan, Sales Income, FIR and **Raw Glass**.

## 1. Menu and layout

- [ ] Click **Raw Glass**. Two reports show: **Batch Consumption** and **Offcut Register**.
- [ ] Each report opens with the page title "Reports · Raw Glass · <report name>".
- [ ] The filters sit on the right of the filter bar, the Export XLSX and Download buttons sit on the right of the title bar.
- [ ] Table headings that show an area read "(sqm)", not "(sq m)".
- [ ] Switching between the two reports starts each with empty filters.

## 2. Report 1: Batch Consumption

### 2.1 Columns

- [ ] Columns in order: Production Date, Work Order No, Work Order Line / Product, Manufacturing Stage, Raw Glass Batch No, Glass Specification, Raw Glass Area Consumed (sqm), Finished Glass Area (sqm), Reusable Offcut Area (sqm), Waste Area (sqm), Cutting Utilisation (%).
- [ ] There is no Source, Finished Pieces or Cutting Completed Date column.
- [ ] Work Order Line / Product, Manufacturing Stage and Glass Specification are in Pascal case (for example "Annealed Glass 8 Mm" style, not all capitals).

### 2.2 Expected rows for the test data

| Row | Raw glass consumed | Finished | Offcut | Waste | Utilisation |
|---|---|---|---|---|---|
| WO-A (cut from a new sheet) | 8.930 | 0.8 (entered) | 3660 x 1640 = 6.002 | 2.128 (entered) | 8.96 % |
| WO-B (cut from the offcut) | 6.002 | 0.8 (entered) | 840 x 3660 = 3.074 | wastage left empty: 6.002 - 0.8 - 3.074 = 2.128 | 13.33 % |

- [ ] WO-A row matches the numbers above.
- [ ] WO-B row matches the numbers above (waste worked out because wastage was left empty).
- [ ] If you entered a different wastage on WO-A, the Waste Area shows your entered value, not the calculated one.
- [ ] If you entered a different offcut length and width, Reusable Offcut Area shows length x width / 1,000,000.

### 2.3 Production Date and Manufacturing Stage

This is the most important check. They must match the Work Order execution log.

- [ ] Open the piece of WO-A. The Execution Log shows Cutting **Started** on a date. The report **Production Date** for WO-A is exactly that date (day, month and year).
- [ ] The same for WO-B.
- [ ] Manufacturing Stage shows "Cutting", the same stage name as in the Execution Log.
- [ ] A work order with several pieces started on different days: the row for an old whole-order stock record shows the **earliest** Cutting start of that work order (for example WO-20261015-152 shows 20/10/2026). A newer work order with one stock cut per piece shows that piece's own start date.
- [ ] A work order approved but never started in Cutting shows its approval date.

### 2.4 Filters

- [ ] **Start Date and End Date** keep only rows whose Production Date is inside the range, both ends included. Picking a date in the future is not allowed.
- [ ] Choosing an end date before the start date is refused with a message.
- [ ] **Work Order** dropdown: pick WO-A, only the WO-A row stays.
- [ ] **Batch** dropdown: pick the test lot code, only rows of that batch stay.
- [ ] **Specification** dropdown: pick "Annealed Glass 8 Mm" style entry, only rows of that glass stay. The names are in Pascal case.
- [ ] Combine two filters (work order and batch). The result is the rows matching both.
- [ ] A filter with no match shows "No raw glass consumption for the selected filters."
- [ ] **Reset** is disabled when no filter is set. After setting a filter it is enabled, and pressing it clears all filters and goes back to page 1.

### 2.5 Paging

- [ ] The footer shows "Showing 1 - N of N batch consumption rows" and the rows per page choice.
- [ ] With more rows than the page size, Next and Previous work and the numbers update.
- [ ] Changing a filter goes back to page 1.
- [ ] Column headings are not sortable (clicking a heading does nothing).

### 2.6 Old and special data

- [ ] An older work order with no piece sizes shows only Raw Glass Area Consumed. Finished, Offcut, Waste and Utilisation show a dash.
- [ ] A cut that was returned (work order rejected or cancelled) does not appear in the report.

### 2.7 Downloads

- [ ] **Export XLSX** downloads a file with the same headings and **every** row that matches the filters (not only the visible page). The last row is a Total row.
- [ ] **Download** (PDF) opens or downloads a PDF titled "Raw Glass Batch Consumption and Cutting Utilisation", with the filters listed under the title and a Total row.
- [ ] The buttons are disabled when the report has no rows.
- [ ] Both files show the "(sqm)" headings.

## 3. Report 2: Offcut Register

### 3.1 Columns

- [ ] Columns in order: Offcut ID, Generated Date, Source Work Order No, Raw Glass Batch No, Glass Specification, Length (mm), Width (mm), Offcut Area (sqm), Storage Location, Status, Consuming Work Order.
- [ ] There are no Available Qty or Reserved Qty columns.
- [ ] Glass Specification is in Pascal case.

### 3.2 Expected rows for the test data

| Offcut | Length x Width | Area | Status after the flow | Consuming work order |
|---|---|---|---|---|
| Made by WO-A | 1640 x 3660 | 6.002 | Consumed (WO-B took it, and its Cutting is done) | WO-B |
| Made by WO-B | 840 x 3660 | 3.074 | Available | blank |

- [ ] The two rows exist with these sizes and areas.
- [ ] Offcut ID looks like `OFC-000123` and is unique for every row.
- [ ] Storage Location shows the location of the lot (blank if the lot has none).

### 3.3 Status (the three states)

Test each state with a fresh offcut.

| State | How to produce it | Expected status |
|---|---|---|
| Available | A leftover is saved and no work order has taken it | Available |
| Reserved | Approve a work order that takes the offcut, but do not complete its Cutting stage | Reserved |
| Consumed | Complete the Cutting stage of that piece | Consumed |

- [ ] A freshly saved offcut shows **Available**.
- [ ] After approving a work order that uses it (stock taken, Cutting not done), the offcut shows **Reserved** and Consuming Work Order shows that work order.
- [ ] After the Cutting stage is completed (with the Cutting entry), it shows **Consumed**.
- [ ] If that work order is rejected before cutting, the offcut goes back to **Available** and Consuming Work Order is empty.
- [ ] Two work orders that used the same offcut row list both codes in Consuming Work Order.

### 3.4 Generated Date

- [ ] For an offcut made by WO-A, Generated Date equals the Cutting **Started** date of the WO-A piece in the Execution Log.
- [ ] An offcut with no Cutting stage (older data) shows the day it was saved.

### 3.5 Filters, paging, Reset, downloads

- [ ] **Start Date and End Date** filter on Generated Date.
- [ ] **Specification**, **Location**, **Status** (Available, Reserved, Consumed) and **Source Work Order** filters each work alone and together.
- [ ] Status filter "Reserved" shows only reserved rows, and the same for Available and Consumed.
- [ ] Reset, paging and the empty message ("No offcuts for the selected filters.") behave as in the first report.
- [ ] Export XLSX and Download (PDF titled "Reusable Offcut Register") contain every matching row and the same columns as the screen.

## 4. Other reports still work (regression)

- [ ] Sales Rep > Sales Visit Register and Sales Rep Performance open and filter as before.
- [ ] Production Plan > Daily Production, Shift-Wise Production and Department-Wise open, filter and download. Area headings read "(sqm)".
- [ ] Sales Income and FIR reports open normally.
- [ ] Filter dropdown text is not cut off on any report (the boxes are wider now).

## 5. Permissions and errors

- [ ] A user without the Reports View right cannot open the Raw Glass reports (menu hidden or access refused).
- [ ] Stop the backend and open a report. A "Could not load the report" panel with **Try again** shows. After restarting the backend, Try again loads the rows.

## 6. Result summary

| Area | Pass / Fail | Notes |
|---|---|---|
| Menu and layout | | |
| Batch Consumption numbers | | |
| Production Date matches execution log | | |
| Batch Consumption filters, paging, Reset | | |
| Batch Consumption downloads | | |
| Offcut Register columns and numbers | | |
| Offcut status (Available, Reserved, Consumed) | | |
| Generated Date matches execution log | | |
| Offcut Register filters and downloads | | |
| Regression of other reports | | |
| Permissions and errors | | |
