# Order Booking: Complete Process & Field Guide

> **Target Audience:** Sales coordinators, order entry operators, estimating staff, plant managers, billing operators, and business owners overseeing custom architectural glass manufacturing.

---

## 1. What is Order Booking?

Think of bespoke tailoring for a luxury three-piece suit. The master tailor records exact chest, shoulder, and sleeve measurements down to the fraction of an inch, specifies buttonhole placements, chooses lapel linings, and notes pressing charges before cutting the expensive fabric. In a glass plant, once raw float glass is cut, drilled, edge-polished, and heated inside the tempering furnace, it can never be recut, drilled, or altered. Any dimension mistake means the entire glass panel becomes scrap.

**Order Booking** is the master manufacturing contract in the ERP. It records verified millimetric glass panel dimensions, calculates chargeable square footage, configures fabrication work (such as cutouts, hinge holes, and edge polishing), assigns machine routing, applies commercial rate cards and extra processing charges, and generates the definitive order that drives downstream Proforma Invoicing and factory Work Orders.

### Key Rules:
* 🏢 **Irreversible Process Accountability:** Glass cannot be altered after tempering. Order Booking requires strict millimetric verification before an order can be confirmed for factory production.
* 💼 **Two Order Origins:** Bookings can be entered directly as fresh sales orders (`DIRECT`) or converted seamlessly from an approved customer enquiry (`ENQUIRY`) with attached slips pre-loaded.
* 📏 **Chargeable Rounding Standards (+20mm / +50mm):** Raw float glass sheets have standard manufacturing cutting tolerances. The system supports calculating billing area based on actual sizes or rounded cutting increments (such as `+20MM`, `+30MM`, or `+50MM`) to cover raw material trimming waste.
* 🛡️ **Submission Lock:** An order in `DRAFT` status can be edited, expanded, or recalculated at will. Once clicked "Submit", the booking transitions to `CONFIRMED`, locking line items from accidental edits and passing the job forward to commercial billing and factory execution.

---

## 2. Key Components

When an Order Booking is saved, it generates a comprehensive production and commercial docket:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                         ORDER BOOKING DOCKET                           │
├────────────────────────────────────────────────────────────────────────┤
│  BOOKING HEADER                                                        │
│  • Booking No:    BK-20261009-001               • Date:   09-10-2026   │
│  • Customer:      Apex Glass & Façades Pvt Ltd  • Status: [ CONFIRMED ]│
│  • Project Site:  Skyline Tower Phase 2         • Type:   ENQUIRY      │
│  • Customer Ref:  PO-9942 / SLIP-405            • Terms:  30% Advance  │
│                                                                        │
│  GLASS SIZING & FABRICATION MATRIX                                     │
│  Line 1: Clear Toughened 12mm                                          │
│  • Sizes:      Width: 1200 mm | Height: 2400 mm | Qty: 4 pcs           │
│  • Charge Per: +50MM (Chargeable: 1250 x 2450 mm)                      │
│  • Area:       131.97 Sq.Ft (12.25 Sq.M)                               │
│  • Holes:      4 Hinge Holes (12mm) | 1 Handle Cutout                  │
│  • Grinding:   Flat Polish on all 4 sides                              │
│  • Base Rate:  ₹110.00 / Sq.Ft                                         │
│                                                                        │
│  COMMERCIAL & CHARGE SUMMARY                                           │
│  • Basic Glass Value:     ₹14,516.70                                   │
│  • Fabrication Charges:   ₹1,200.00 (Holes & Cutouts)                  │
│  • Freight & Packaging:   ₹800.00                                      │
│  • Order Discount (2%):   -₹330.33                                     │
│  • Taxable Total:         ₹16,186.37                                   │
│  • GST (18% IGST):        ₹2,913.55                                    │
│  • Net Payable:           ₹19,100.00                                   │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Order Booking Header:** Captures statutory customer identity, project location, customer purchase order or slip reference, booking date, and delivery deadlines.
2. **Glass Sizing & Dimension Panel:** Captures exact millimeters for standard rectangular panels or irregular shapes, with automatic area math in square meters and square feet.
3. **Fabrication & Processing Configurator:** Captures hole drilling counts, hinge and lock cutouts, safety corner radii, logo stamping, and perimeter edge grinding styles.
4. **Additional Charges & Surcharges:** Configures dynamic line charges such as special machine work, wooden crate packaging, freight, crane loading, and insurance.
5. **Commercial Calculation Engine:** Real-time computation of square footage, base glass rates, fabrication charges, order-level discounts, GST tax math, and final order value.

---

## 3. Step-by-Step Process

```text
  [1. Create Booking Header] ──► [2. Add Glass Products & Sizes] ──► [3. Specify Holes & Edge Work]
                                                                                   │
                                                                                   ▼
  [6. Advance to PI / Work Order] ◄── [5. Confirm Booking] ◄── [4. Configure Routing & Charges] ◄─┘
```

### Step 1: Create Booking Header
The operator clicks "New Booking" (or converts an approved enquiry). They select the customer, site project, booking date, expected delivery date, and enter the customer purchase order or slip reference.

### Step 2: Add Glass Products & Line Items
For each glass specification, the operator selects the glass product (for example: Clear Toughened, Tinted, Frosted, Low-E), variety tint, and thickness in millimeters. Operators can also use the built-in AI tool to read handwritten WhatsApp slips directly into line item drafts.

### Step 3: Enter Millimeter Dimensions & Fabrication
Operators enter panel dimensions (Width and Height in millimeters, or inches/feet with automatic millimetric conversion). They specify the `Charge Per` rounding rule (`ACTUAL`, `+20MM`, `+30MM`, or `+50MM`). If the glass requires hardware fabrication, they enter hole counts, center holes, handle cutouts, and edge polishing requirements.

### Step 4: Configure Additional Charges & Routing
The operator reviews machine routing (such as Cutting, Flat Polishing, Drilling, Washing, Tempering) and adds extra commercial charges (such as delivery freight, wooden box packing, or special logo charges).

### Step 5: Review & Submit Booking
The operator reviews totals on the Review tab: total piece count, total square feet, basic glass amount, additional charges, and GST. Clicking "Submit" confirms the booking, transitioning its status from `DRAFT` to `CONFIRMED`.

### Step 6: Hand-Off to Billing and Factory Floor
Once confirmed, the booking is locked from editing. It becomes immediately available in the Proforma Invoice (PI) Status module for customer advance billing and in the Work Order module for factory production scheduling.

---

## 4. Field-by-Field Explanation

Here is what every single field in the Order Booking module means:

---

### 4.1 Header Details
* **Booking Number:** Unique sequential identifier generated by the ERP (for example: `BK-20261009-001`).
* **Booking Type:** Origin of the order:
  * `DIRECT`: Entered directly by sales operators.
  * `ENQUIRY`: Automatically converted from a customer inquiry ticket.
* **Customer (Party):** The registered dealership, contractor, or builder purchasing the glass.
* **Project Name:** The specific construction project, building site, or interior floor where the glass panels will be delivered.
* **Booking Date:** Date when the order was booked.
* **Required Delivery Date:** Committed delivery date when the glass must reach the customer site.
* **Customer Reference / PO No:** Customer purchase order number, slip number, or delivery slip code.
* **Currency Code:** Transaction currency (defaults to `INR`).
* **Remarks:** Delivery instructions, vehicle restrictions, or packing notes.

---

### 4.2 Sizing & Dimension Panel
* **Unit of Measurement:** Input measurement unit: `MM` (standard industry default), `INCH`, or `FEET`. Non-metric units are automatically converted into exact millimeters.
* **Actual Width & Actual Height:** Physical manufactured dimensions of the finished glass panel in millimeters.
* **Top Width, Bottom Width, Left Length, Right Length:** Used for non-rectangular or trapezoidal glass panels (such as staircase glass or sloped roof lights).
* **Quantity:** Number of identical panels for this line item.
* **Charge Per (Rounding Rule):** Commercial billing rounding option:
  * `ACTUAL`: Exact square footage calculated from physical dimensions.
  * `+20MM`, `+30MM`, `+50MM`: Rounds physical dimensions up to the nearest cutting increment before calculating square footage to account for raw float sheet trim margins.
* **Square Feet (Sq.Ft) & Square Meters (Sq.M):** Calculated billable area: `(Chargeable Width in mm × Chargeable Height in mm) / 92903.04 × Quantity`.

---

### 4.3 Fabrication & Machine Work Details
* **Glass Shape:** Profile configuration (standard `Blocked` rectangular, `Trapezoid`, `Rake`, `Arch`, or custom CAD template).
* **Hole Qty:** Number of standard holes drilled for spider fittings, patch fittings, or glass standoffs.
* **Centre Hole Qty:** Internal cut holes or sink/outlet holes positioned away from perimeter edges.
* **Cutout Qty:** Standard edge corner cutouts for door hinges, floor springs, or architectural locks.
* **Big Hole Qty & Big Cutout Qty:** Oversized cutouts requiring specialized CNC waterjet or milling operations.
* **Grinding (Top, Bottom, Left, Right):** Edge polishing style applied to each perimeter side (`Flat Polish`, `Pencil Polish`, `Beveling`, `Rough Grind`, or `None`).
* **Logo Qty:** Stamp or acid-etch logo marking indicating toughened glass safety certification (such as ISI mark or plant logo).

---

### 4.4 Commercials & Tax Math
* **Rate Per Sq.Ft / Sq.M:** Base price charged for the raw glass and basic tempering per area unit.
* **Basic Glass Amount:** Net glass cost calculated as `Billable Area × Rate`.
* **Fabrication Charges:** Surcharges for holes, cutouts, CNC milling, and beveling.
* **Other Charges:** Transport freight, wooden crate crating, unloading labor, or transit insurance.
* **Order Discount:** Commercial discount applied either as a percentage (`%`) or flat value (`₹`) across basic and fabrication charges.
* **Taxable Total:** Net taxable amount after subtracting discounts and adding extra charges.
* **GST Rates (CGST/SGST/IGST):** Statutory Indian Goods and Services Tax: 9% CGST + 9% SGST for intra-state sales, or 18% IGST for inter-state deliveries.
* **Grand Total:** Final billable amount payable by the customer.

---

### 4.5 Status & Control Flags
* **DRAFT:** Work in progress. Line items can be added, updated, cloned, or deleted freely.
* **CONFIRMED:** Finalized sales contract. Dimensions and line items are locked to prevent shop floor discrepancies.
* **CANCELLED:** Revoked order. Archived for historical reporting.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Why does Order Booking have Charge Per options like +50MM?
In the architectural glass processing industry, large raw float glass sheets (such as 2440 × 3660 mm) must be scored and snapped on cutting tables. Small cutoffs cannot always be reused. Rounding glass dimensions up to the nearest 50 mm increment is standard industry practice to compensate processors for raw sheet trim waste and cutting blade margins.

### Q2: Can I edit an order booking after clicking "Submit"?
**No.** Once an order is submitted, its status updates to `CONFIRMED`. Because a confirmed booking immediately generates commercial Proforma Invoices and shop floor Work Orders, line items cannot be modified directly. If a customer alters measurements before production starts, authorized administrators must unlock the booking or cancel and replicate it.

### Q3: Can glass items be imported directly from customer WhatsApp photos?
**Yes.** The Products tab includes an integrated "Read from Photo" feature. When an enquiry with attached slips is converted, our built-in AI tool scans the handwritten pencil slips, extracts panel dimensions and quantities, and queues them as draft rows for operator verification.

### Q4: What is the difference between Order Booking and a Work Order?
* 📋 **Order Booking:** The **commercial and contractual record** between your company and the customer, capturing pricing, taxes, payment terms, and overall order dimensions.
* 🏭 **Work Order:** The **shop floor manufacturing traveler** issued to factory machine operators, containing physical barcodes, furnace loading schedules, and piece-by-piece inspection tracking.

