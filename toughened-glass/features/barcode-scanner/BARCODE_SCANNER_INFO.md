# Barcodes & Production Line: Complete Process & Field Guide

> **Target Audience:** Factory managers, production supervisors, machine operators, office staff, and anyone who wants to understand how glass pieces are labelled and tracked on the floor, without technical words.

---

## 1. What is the Barcode and Production Line Feature?

Think of it like a luggage tag at an airport: every bag gets its own tag, and every time the bag passes a check point the tag is scanned, so you always know where each bag is and which ones went missing.

**The barcode feature** gives every single piece of glass in a work order its own barcode label. Operators scan the label to open that piece in the app and record each production stage as it starts and finishes. If a piece breaks, it is marked rejected with a reason, and the app creates a replacement piece with a new label so the order can still be completed.

### Key Rules:
* 🏷️ **One barcode per piece:** A product with quantity 10 becomes 10 pieces, each with its own barcode.
* ✅ **Two approvals:** Labels can be printed after **Factory** approval. Scanning and stage work only start after **both Party and Factory** approval.
* 🔁 **Broken glass is replaced, not lost:** A rejected piece keeps its history, and a replacement with the same size and details gets a new barcode ending in -R1 (then -R2 if that breaks too).
* 🏁 **Done means every live piece is done:** A work order is Completed only when every piece still in production has finished its last stage. Rejected pieces and products that were denied do not hold it back.
* 📷 **Two ways to scan:** the device camera (phone, tablet or laptop) or a handheld scanner gun.

---

## 2. Key Components

```text
┌──────────────────────────────────────────────────┐
│  ||| |||| | ||| || |||| | || ||| | |||| ||| ||   │
│  RB-WO-0042-01-007                               │
│                                                  │
│  WO: WO-0042            Party: ABC Builders      │
│  Product: TUFF GLASS - Clear       12 mm         │
│  Size: 1200 x 900 mm    Holes: 2   Cutouts: 0    │
│  Route: CUT / GRN / HOLE / TMP / STR             │
│  Piece 7 / 10                  Date: 06/10/2026  │
└──────────────────────────────────────────────────┘
```

1. **The barcode:** A real Code 128 barcode that any camera or scanner gun can read. The text under it is the same value.
2. **Barcode value:** `RB-<work order>-<product number>-<piece number>`, for example `RB-WO-0042-01-007` is piece 7 of the first product in work order WO-0042. A replacement adds `-R1`, `-R2` and so on.
3. **Piece details:** The work order, party, product, thickness, this piece's own size, holes and cutouts, the route of stages, and "Piece 7 / 10" (plus the replacement number when it is one).
4. **Label formats:** **A4 sheet** (10 labels per sheet, the default) or **Thermal label** (100 x 50 mm, one label per page, for a label printer). The choice is remembered on each device.

---

## 3. Step-by-Step Process

```text
  [1. Work order made]  ──►  [2. Factory approves]  ──►  [3. Print labels]
                                                               │
                                                               ▼
  [6. Work order done]  ◄──  [5. Scan and work stages]  ◄──  [4. Party approves]
```

### Step 1: Work order is made
When a work order is created from an order booking, the app makes one item per piece of glass, each with its own barcode, its own size and its own list of stages.

### Step 2: Factory approval
Each product in the work order is reviewed (Approve or Deny). The work order then needs Factory approval.

### Step 3: Print the labels
In the work order details window, each product card has a **Barcode** button (shown only after Factory approval). Choose the label format and whether to print the **whole product** (every live piece) or **select pieces** (to reprint a damaged label). The labels open as a PDF with the print window. There is also **Print Scan Code** in the work order's Print menu, which prints the labels of every live piece in the whole work order. Stick each label on its piece of glass.

### Step 4: Party approval releases the work order
Once both Party and Factory have approved, the work order moves to **In Progress** and its pieces can be scanned.

### Step 5: Scan and work the stages
Click **Scan Barcode** (on the Work Orders page, on a work order's own page, or on the Production Line page). Pick **Camera** or **Handheld scanner**:
* **Camera:** hold the label inside the frame on screen; it reads by itself.
* **Handheld scanner:** pull the trigger; the code is typed in and opened.

The piece's page opens. It shows the party, product, stages done, the barcode and "Piece n / N", a stepper of all stages, and an execution log. The operator clicks **Approve / Start Stage** to start the current stage, does the work, then clicks **Complete Stage**. The next stage then waits to be started.

If a piece breaks or fails a check, the operator clicks **Reject piece** and types a reason. The app creates the replacement straight away and offers to **Print label** for it and **Open replacement**. The new glass is cut, labelled, and goes through the stages from the first one.

### Step 6: The work order completes
When every live piece has finished its last stage, the work order becomes **Completed** on its own, and the app prepares its Final Invoice.

---

## 4. Field-by-Field Explanation

---

### 4.1 Production Line list (one row per released work order)
* **Work Order:** the work order number.
* **Party / Project:** who it is for.
* **Products:** the product names and how many products.
* **Pieces:** how many pieces are done out of the total, a progress bar, and how many were rejected.
* **Delivery:** the delivery date.
* **Stages:** a chip for each stage (CUT, GRN, HOLE...) coloured by how far it has got. Hover to see "x/y pieces".
* **Status:** **Pending** (released, nothing started), **In Progress** (any stage started on any piece), **Completed** (all counted pieces done).
* The cards at the top count released, pending, in progress and completed work orders. There are tabs to filter by status and a search box.

---

### 4.2 The piece page
* **Party, Product, Stages Completed, Barcode:** summary cards. The barcode card shows "Piece 7 / 10", plus "Replacement 1" when it is one.
* **Stepper:** every stage in order. Click a finished stage to see what was done there.
* **Stage panel:** what the current stage asks for (for example the edge type for grinding, or tempering limits) and the Start or Complete button.
* **Execution log:** when each stage started and finished, and who did it.
* **Print label:** reprint this piece's label.
* **Reject piece:** only for people with control rights, only on a live piece.

---

### 4.3 Rejected and replacement pieces
* **A rejected piece** opens read-only, with a red banner showing the reason and an **Open replacement** button.
* **A replacement piece** shows an amber note "Replaces rejected piece ..." with a link back to the old one, and starts again from the first stage.
* In the pieces list, a rejected piece is shown struck through with its reason, just above its replacement.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Why can't I scan a piece yet?
Scanning only works after **both** Party and Factory approval. Until then the app says the work order needs both approvals.

### Q2: A label got damaged. How do I get a new one?
Open the work order details window, click **Barcode** on that product, choose **Select pieces**, tick the piece, and print. Or open the piece's page and click **Print label**.

### Q3: A piece broke at grinding. What happens to the order?
Click **Reject piece** on that piece and enter the reason. A replacement with a new barcode (ending in -R1) is created, starts from the first stage, and the broken piece is kept as history. The work order will not complete until the replacement is done. Raw glass stock is taken for the replacement and the breakage is logged.

### Q4: A piece broke after the work order was Completed (for example in the store). What now?
Rejecting it still works. A replacement is created and the work order goes back to **In Progress** until the replacement is finished.

### Q5: The scanner says "Barcode was not recognised". What should I do?
Check the label is clean and inside the frame, then scan again. You can also switch to the handheld scanner or type the code by hand. The scan box also accepts a work order number, which opens that work order.

### Q6: The camera does not open.
The browser needs permission to use the camera, and cameras only work on a secure (https) address. Allow the camera for the site in the browser settings and reopen the scan window, or use **Handheld scanner** instead.

### Q7: Where is Production Line in the menu?
It was taken out of the side menu. Use **Scan Barcode** on the Work Orders page, or the **Production Line** link inside a work order's details window.
