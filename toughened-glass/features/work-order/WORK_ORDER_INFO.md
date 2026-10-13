# Work Order & Production Tracking: Complete Process & Field Guide

> **Target Audience:** Plant managers, production engineers, shift supervisors, machine operators (cutting, edging, CNC drilling, washing, furnace), quality inspectors, dispatch coordinators, and business owners.

---

## 1. What is a Work Order?

Think of a detailed flight plan and passenger boarding pass system at an international airport. The flight plan outlines the exact route, fuel load, and safety checkpoints, while each passenger carries an individual barcoded boarding pass scanned at security, the departure gate, and the aircraft door. In a toughened glass factory, raw float glass cannot simply be passed around by memory. Every single glass panel receives its own physical job traveler and unique barcode that must be scanned as it passes through the cutting table, the edge polisher, the CNC drill station, the industrial washing machine, and the high-temperature tempering furnace.

A **Work Order** is the definitive shop floor manufacturing passport in the ERP. It translates confirmed customer specifications into executable factory job cards, schedules machine routing, prints piece-level tracking barcodes, coordinates multi-level supervisory approvals, tracks real-time machine scanning, flags breakage or defects, and ensures that only 100% verified toughened panels reach the dispatch staging dock.

### Key Rules:
* 🏢 **Commercial Clearance Requirement:** Work Orders are released to the factory floor only after the linked order has an approved Proforma Invoice or verified credit clearance.
* 🏷️ **Piece-by-Piece Barcode Traceability:** Every single glass panel on an order receives a unique piece number and scannable barcode label, enabling full tracking of individual panels across machine stations.
* 🛡️ **Two-Tier Production Approval:** Draft work orders must be reviewed and approved by authorized plant management before job sheets and barcode labels can be printed for machine operators.
* ⚙️ **Strict Linear Machine Sequence:** Glass must follow a rigorous physical route: Cutting -> Edging -> Drilling -> Washing -> Tempering -> Inspection. Once glass leaves the tempering furnace, it can never be cut or drilled again.

---

## 2. Key Components

When a Work Order is released to the factory floor, it generates a comprehensive job sheet and barcode traveler:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        FACTORY WORK ORDER TRAVELER                     │
├────────────────────────────────────────────────────────────────────────┤
│  WORK ORDER IDENTIFICATION                                             │
│  • Work Order No: WO-20261009-0012              • Date:    09-10-2026  │
│  • Booking Ref:   BK-20261009-001               • Status:  IN PROGRESS │
│  • PI Reference:  PI-2026-0082                  • Priority: HIGH       │
│  • Customer:      Apex Glass & Façades Pvt Ltd  • Delivery: 12-10-2026 │
│  • Project Site:  Skyline Tower Phase 2                                │
│                                                                        │
│  GLASS PANEL SPECIFICATION                                             │
│  • Product:       Clear Toughened 12mm                                 │
│  • Panel Size:    1200 mm × 2400 mm | Thickness: 12.0 mm               │
│  • Quantity:      4 Pieces | Total Area: 124.00 Sq.Ft (11.52 Sq.M)     │
│  • Fabrication:   4 Hinge Holes (Ø 12mm) | 1 Corner Lock Cutout        │
│  • Edge Polish:   Flat Polish on all 4 sides                           │
│                                                                        │
│  MACHINE ROUTING & WORK CENTER STATIONS                                │
│  [1. Cutting] ──► [2. Edging] ──► [3. CNC Drilling] ──► [4. Washing]   │
│                                                                │       │
│  [7. Dispatch Ready] ◄── [6. Quality QC] ◄── [5. Furnace/Heat] ◄───────┘
│                                                                        │
│  INDIVIDUAL PIECE BARCODES & STATION TRACKING                          │
│  • Piece #1: [|||||||||||||||||||||] WO-0012-P1 -> Status: QC PASSED   │
│  • Piece #2: [|||||||||||||||||||||] WO-0012-P2 -> Status: FURNACE     │
│  • Piece #3: [|||||||||||||||||||||] WO-0012-P3 -> Status: CNC DRILL   │
│  • Piece #4: [|||||||||||||||||||||] WO-0012-P4 -> Status: EDGING      │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Work Order Header:** Official manufacturing job number (`WO-YYYYMMDD-XXXX`) linked directly to the commercial booking and proforma invoice.
2. **Glass Technical Specifications:** Detailed millimetric panel dimensions, glass variety, thickness, hole coordinates, cutout profiles, and edge polishing instructions.
3. **Machine Routing Schedule:** Sequential list of physical workstations (such as Cutting, Grinding, Drilling, Washing, Tempering Furnace, and Quality Control).
4. **Piece Barcode Traveler:** Individual scannable barcode assigned to each physical glass panel to track machine-level progress.
5. **Quality & Breakage Logger:** Recording station allowing operators to log broken glass or furnace blowouts, triggering automated remanufacture requests.

---

## 3. Step-by-Step Process

```text
  [1. Generate Work Order from Booking] ──► [2. Review Machine Routing] ──► [3. Supervisor Approval]
                                                                                            │
                                                                                            ▼
  [6. QC Inspection & Dispatch Staging] ◄── [5. Shop Floor Barcode Scanning] ◄── [4. Print Job Travelers]
```

### Step 1: Generate Work Order from Booking
Once the Order Booking is confirmed and its Proforma Invoice is approved, the system generates a draft Work Order capturing panel sizes, hole details, and glass specs.

### Step 2: Review Machine Routing & Glass Sizing
The production planner verifies furnace bed limits, confirms that sheet cutting plans optimize raw float inventory, and ensures all required machine operations (such as CNC cutouts or double-edging) are included in the routing.

### Step 3: Production Supervisor Approval
The plant supervisor reviews the job order. Clicking "Approve" transitions the status from `PENDING_APPROVAL` to `IN_PROGRESS` and authorizes the print room to issue production paperwork.

### Step 4: Print Job Travelers & Barcode Stickers
The office prints the official Work Order Job Sheet along with adhesive barcode labels for each panel. The paperwork is delivered to the cutting table supervisor.

### Step 5: Shop Floor Barcode Scanning
As raw float glass is cut, the operator sticks the unique barcode label onto the corner of each glass panel. At every subsequent machine station (Edging, CNC Drilling, Washing, and Furnace entrance), operators scan the barcode using shop floor scanners or tablets to log completion.

### Step 6: Final Quality Inspection & Dispatch Staging
After emerging from the tempering cooling section, each panel undergoes optical and dimensional inspection (checking for roller waves, bow, edge chips, or surface inclusions). Passed panels are scanned as `COMPLETED` and transferred to wooden A-frames for customer dispatch.

---

## 4. Field-by-Field Explanation

Here is what every field in the Work Order module means:

---

### 4.1 Header & Job Scheduling Information
* **Work Order No:** Unique sequential shop floor job number (for example: `WO-20261009-0012`).
* **Booking No:** Originating Order Booking reference code. Clicking it opens the full commercial booking summary.
* **PI No:** Linked Proforma Invoice number confirming commercial approval.
* **Customer Name:** The purchasing client or dealership name.
* **Project Name:** The architectural site or delivery destination.
* **Application / Order Date:** The date when the work order was created.
* **Committed Delivery Date:** Deadline when finished glass must be ready for vehicle loading.
* **Priority:** Job urgency indicator (`NORMAL`, `HIGH`, or `URGENT`) used by supervisors to prioritize furnace queues.

---

### 4.2 Panel Specifications & Fabrication Details
* **Glass Product:** Base glass description (for example: Clear Toughened, Euro Grey, Tinted Bronze, Frosted).
* **Thickness (mm):** Glass thickness in millimeters (such as 8mm, 10mm, 12mm).
* **Width (mm) & Height (mm):** Exact manufactured panel dimensions.
* **Quantity:** Number of identical panels in this job line.
* **Total Area (Sq.M & Sq.Ft):** Total physical surface area calculated for furnace bed loading.
* **Edge Work Finishing:** Perimeter polishing instructions (such as `Flat Polish`, `Pencil Edge`, `Rough Arris`, or `Beveling`).
* **Holes & Cutout Specifications:** Diameter, quantity, and position of drilled holes and edge cutouts for door fittings.

---

### 4.3 Machine Routing & Work Center Sequence
* **Sequence Number:** Step-by-step physical processing order (such as 10, 20, 30, 40, 50).
* **Work Center Name:** Designated factory machine or processing zone:
  * `Cutting Table`: Raw glass sheet scoring, optimization, and breakout.
  * `Edge Polishing / Grinding`: Perimeter arrissing, flat edge grinding, and polishing wheels.
  * `CNC / Drilling`: Waterjet cutout fabrication and diamond core hole drilling.
  * `Washing Machine`: High-pressure demineralized water washing and hot air drying before heat treatment.
  * `Tempering Furnace`: Heating glass to ~620°C followed by rapid quenching to induce compressive surface stress.
  * `Quality Inspection`: Visual clarity, bow/warp measurement, and safety fragment verification.
  * `Dispatch Packing`: Final staging, crating, and vehicle loading.

---

### 4.4 Barcode & Piece Tracking Fields
* **Piece Number:** Identifier for an individual physical panel (for example: `Piece #1 of 4`).
* **Piece Barcode:** Scannable Code128 barcode or QR code containing the unique piece token.
* **Current Machine Station:** The last recorded work center where the panel was successfully scanned.
* **Operator Name:** Staff member who performed the operation or scan.
* **Scan Timestamp:** Exact date and time when the piece completed the station.
* **Pass / Reject Flag:** Quality indicator verifying that the piece passed dimensional and optical checks.

---

### 4.5 Status & Control Flags
* **DRAFT:** Newly created job ticket undergoing engineering and machine verification.
* **PENDING_APPROVAL:** Awaiting supervisor or plant manager sign-off.
* **IN_PROGRESS:** Active manufacturing on the factory floor with live machine scanning.
* **COMPLETED:** All panels have successfully completed tempering, passed quality control, and are staged for dispatch.
* **REJECTED:** Returned by supervisor for correction before production.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Can a glass panel be recut or drilled if a mistake is discovered after tempering?
**No.** Tempering creates intense internal tensile stress balanced by compressive surface stress. Any attempt to cut, drill, or trim tempered glass will cause it to shatter instantly into thousands of small blunt fragments. All cuts, holes, and edge work must be completed and inspected before the glass enters the washing line and tempering furnace.

### Q2: What happens if a glass panel breaks or cracks inside the tempering furnace?
When a blowout or thermal breakage occurs, the furnace operator logs a breakage event against that specific piece barcode in the system. The ERP automatically generates a replacement piece ticket linked to the same Work Order so the cutting line can immediately cut a replacement panel without delaying the entire project.

### Q3: Why is barcode scanning required at each workstation?
Barcode scanning provides live visibility into where every piece of glass is physically located inside the factory. If a customer calls asking when their 10 shower cubicle panels will be ready, customer service can instantly see that 8 pieces have passed the furnace and 2 pieces are at the drilling machine, eliminating manual phone calls to the factory floor.

### Q4: Can a Work Order be modified once it is IN PROGRESS?
**No.** Once glass is being processed on cutting tables or polishing lines, modifying dimensions or quantities in the ERP can cause dangerous confusion on the shop floor. If an architect changes sizes mid-production, the supervisor must pause or cancel the active work order and reissue a new job card for the altered pieces.

