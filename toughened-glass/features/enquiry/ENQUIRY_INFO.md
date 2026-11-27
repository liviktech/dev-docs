# Enquiry Management: Complete Process & Field Guide

> **Target Audience:** Sales executives, front-desk coordinators, estimating teams, factory managers, and business owners managing incoming customer requests for architectural and toughened glass.

---

## 1. What is Enquiry Management?

Imagine a builder or glass dealer walking into your sales counter with a rough handwritten paper slip, or sending photos of carpenter measurements over WhatsApp. Before drafting a formal commercial contract or cutting raw glass sheets, the sales team needs a fast, simple place to log the customer details, upload the photos, record requested glass sizes, and calculate an initial price quote.

**Enquiry Management** is the front-line sales intake module where customer glass inquiries are captured, paper slips and WhatsApp photos are attached, dimensions and approximate square footage are calculated, and pricing follow-ups are tracked until the customer approves the quote. Once approved, the enquiry can be converted into an official **Order Booking** with a single click.

### Key Rules:
* 🏢 **Zero-Item Inception Allowed:** Sales operators can save an enquiry without entering exact glass measurements as long as at least one customer slip or photo is attached.
* 💼 **Instant One-Click Conversion:** When a customer approves an estimate, clicking "Convert" automatically copies customer details, site info, and attached photo slips straight into an Order Booking draft.
* 🛡️ **Immutable Conversion Lock:** Once an enquiry is converted or closed, it becomes locked from casual editing to protect the sales audit trail and prevent discrepancies with booked orders.
* 🔄 **Reopen Flexibility:** If a client cancels or goes quiet, staff can mark the enquiry as Closed. If the client calls back weeks later, the enquiry can be reopened to Open without losing historical notes or uploaded photos.

---

## 2. Key Components

When an enquiry is recorded in the system, it produces a structured sales record containing header details, attached order slips, and optional glass dimensions:

```text
┌─────────────────────────────────────────────────────────┐
│                 CUSTOMER ENQUIRY RECORD                 │
├─────────────────────────────────────────────────────────┤
│  ENQUIRY IDENTIFICATION                                 │
│  • Enquiry Code:  ENQ-20261008-001                      │
│  • Enquiry Date:  08-10-2026                            │
│  • Status:        [ OPEN ]                              │
│                                                         │
│  CUSTOMER & SALES INFORMATION                           │
│  • Customer:      Apex Glass & Façades Pvt Ltd          │
│  • Project Site:  Skyline Tower Phase 2                 │
│  • Customer Ref:  SLIP-405 (WhatsApp from Ramesh)       │
│  • Sales Rep:     Rajesh Kumar                          │
│  • Source:        WHATSAPP                              │
│                                                         │
│  ATTACHED PHOTO SLIPS                                   │
│  • 📎 2 Photos Attached (Handwritten slip, Site sketch)  │
│                                                         │
│  SPECIFICATION ESTIMATE                                 │
│  • Clear Toughened 12mm | 1200 x 2400 mm | Qty: 4 pcs   │
│  • Estimated Total Area: 124.00 Sq.Ft                   │
│  • Remarks: Urgently required for terrace railing       │
└─────────────────────────────────────────────────────────┘
```

1. **Enquiry Identification Code:** A unique, date-based tracking code (such as `ENQ-YYYYMMDD-XXX`) generated automatically so sales staff and customers can reference the request easily.
2. **Customer & Project Association:** Direct link to customer master records and site projects, allowing sales staff to add new parties or site locations immediately without leaving the form.
3. **Attached Customer Slips & Photos:** High-resolution photo upload drawer storing customer slips, hand sketches, and WhatsApp screenshots that persist throughout the entire order lifecycle.
4. **Sales Rep Assignment:** Assigns lead ownership to specific sales representatives to track follow-ups, conversion performance, and customer accountability.
5. **Specification & Area Estimator:** Real-time square footage and square meter area calculations based on entered millimeter dimensions.

---

## 3. Step-by-Step Process

```text
  [1. Receive Request via WhatsApp/Call] ──► [2. Log Enquiry & Upload Slips] ──► [3. Review & Quote Estimate]
                                                                                               │
                                                                                               ▼
  [6. Order Booking Created] ◄── [5. One-Click Convert] ◄── [4. Manager Approval] ◄────────────┘
```

### Step 1: Receive Customer Request
The customer calls, sends WhatsApp photos of paper slips, emails drawings, or visits the factory office with glass specifications.

### Step 2: Log Enquiry & Upload Photos
The sales executive opens the Enquiry form, selects or creates the customer, enters the customer slip number, sets the source (such as WhatsApp or Walk-in), and uploads photos of the paper slips into the photo drawer.

### Step 3: Review Slips & Estimate Dimensions
If sizes are already finalized, the estimator enters glass types, thicknesses, and width/height dimensions. The ERP automatically computes total square feet and panel count. If sizes are still rough, staff can save the enquiry with zero line items and attach the photos for later processing.

### Step 4: Management Review & Approval
Sales coordinators follow up with the customer. Once the customer agrees to the estimate, an authorized manager or administrator approves the enquiry, changing its status to `APPROVED`.

### Step 5: One-Click Convert to Order Booking
Clicking the "Convert" button automatically opens the Order Booking screen. Customer details, project name, customer reference, and all attached photo slips carry over automatically with zero duplicate data entry.

### Step 6: Enquiry Conversion Lock
The enquiry status updates to `CONVERTED`. The record is preserved permanently in historical archives for sales analytics and audit tracking.

---

## 4. Field-by-Field Explanation

Here is what each field in the Enquiry module means:

---

### 4.1 Header & Customer Information
* **Customer:** The name of the client, architect, contractor, or glass dealership placing the request. A "+ Add New Party" button allows on-the-fly customer creation.
* **Project Name:** The specific construction site, tower, or interior project where the glass will be installed (for example: "Villa 42", "Skyline Tower"). Can be added directly during entry.
* **Enquiry Date:** The calendar date when the inquiry was received. Constrained to today or earlier dates to prevent future-dated entries.
* **Customer Reference:** The customer paper slip number, purchase order reference, or WhatsApp note (for example: "SLIP-405" or "WA-Ramesh"). Helps match physical paper slips with ERP records.
* **Assigned Sales Rep:** The internal sales representative responsible for following up on the lead, providing quotes, and closing the order.
* **Enquiry Source:** The channel through which the lead arrived (such as `PHONE`, `EMAIL`, `WHATSAPP`, `Walk-in`, `Instagram`, or custom marketing sources). Custom sources can be added on the fly.
* **Remarks:** Internal instructions, delivery notes, special edge polishing remarks, or site conditions.

---

### 4.2 Photo Slips & Visual Evidence
* **Attached Photos:** Customer slips, site sketches, or architectural blueprints uploaded via mobile camera or desktop file chooser.
* **Zoom / Fit Preview:** A built-in photo preview viewer that allows estimators to zoom in on handwritten measurements and examine tiny carpenter pencil markings directly within the ERP.

---

### 4.3 Glass Items & Dimension Estimates (Optional in Enquiry)
* **Product:** Base glass category (for example: Clear Toughened, Frosted Toughened, Tinted Glass, Tinted Toughened).
* **Sub-Product / Variety:** Specific tint, shade, or coated variety (for example: Extra Clear, Euro Grey, Bronze, Dark Green).
* **Thickness (mm):** Glass thickness in millimeters (such as 5mm, 6mm, 8mm, 10mm, 12mm, 15mm, 19mm).
* **Width (mm) & Height (mm):** Exact dimensions in millimeters.
* **Quantity:** Number of identical panels required.
* **Calculated Square Footage:** Automatically computed area formula: `(Width in mm × Height in mm) / 92903.04 × Quantity`.
* **Process Notes:** Special fabrication notes for the estimating team, such as hinge cutouts, holes, beveling, or safety corner requirements.

---

### 4.4 Status & Lifecycle Flags
* **Status Badge:**
  * `OPEN`: Active lead under review or awaiting price confirmation. Fully editable.
  * `APPROVED`: Verified by sales management and ready for order conversion.
  * `CONVERTED`: Successfully converted into an active Order Booking. Read-only.
  * `CLOSED`: Dropped, lost, or cancelled lead. Read-only, but can be reopened to `OPEN` if the customer reconnects.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Can I save an enquiry if the customer only sent a photo slip without exact measurements?
**Yes.** In the toughened glass industry, customers frequently drop off paper slips with dozens of handwritten sizes. You can create an enquiry, upload the photo slip, and save it with zero line items. When you later convert the enquiry to Order Booking, our built-in AI tool or order operators can read the photo and extract the exact sizes.

### Q2: What happens when I click the "Convert" button on an approved enquiry?
Clicking "Convert" immediately initiates a new Order Booking draft pre-filled with the customer name, project, reference slip number, and all attached photo slips. Once the order booking is saved, the enquiry status updates automatically to `CONVERTED` and becomes read-only.

### Q3: Can an enquiry be edited after it is approved or converted?
**No.** Once an enquiry is marked as `APPROVED` or `CONVERTED`, the edit button is disabled to preserve commercial integrity and prevent accidental changes to terms already handed off to production. If changes are needed on an approved enquiry before conversion, an administrator must reopen the status.

### Q4: Can a closed enquiry be reopened if a customer calls back weeks later?
**Yes.** If a customer originally declined a quote and the enquiry was closed, staff with control permissions can click the "Reopen" action icon. The enquiry returns to `OPEN` status, preserving all previous notes and photos so sales staff can resume follow-up without creating a duplicate record.

