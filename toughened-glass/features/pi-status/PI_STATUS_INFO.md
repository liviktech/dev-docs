# Proforma Invoice (PI) Status: Complete Process & Field Guide

> **Target Audience:** Accounts executives, billing operators, finance managers, sales coordinators, dispatch heads, and business owners managing commercial verification and advance payments.

---

## 1. What is Proforma Invoice (PI) Status?

Imagine ordering a set of custom granite countertops or custom modular kitchen cabinets. The fabricator will never slice expensive stone slabs to your unique kitchen layout without first issuing a formal quotation and collecting a 50% advance deposit. Because toughened glass is cut, drilled, and heat-tempered to exact millimetric specifications, it cannot be returned, resized, or repurposed for another building if a customer changes their mind. If an unverified order goes into the furnace, the factory bears 100% of the financial loss.

**Proforma Invoice (PI) Status** is the commercial control checkpoint of the ERP. It manages formal proforma invoices created from confirmed order bookings, verifies customer approval, records multi-channel advance payments (via NEFT, RTGS, UPI, Cheques, or Cash), enforces commercial locks, and serves as the financial green light that authorizes factory Work Orders for shop floor manufacturing.

### Key Rules:
* 🏢 **Production Gatekeeper:** Factory machine lines should not begin processing custom glass until the linked Proforma Invoice is approved through advance payment or formal management credit clearance.
* 💼 **Multi-Mode Advance Receipts:** Customers can pay advances in multiple installments across diverse channels (such as bank transfers, UPI apps, cheques, or cash); each receipt is recorded in an audit-ready payment ledger.
* 🛡️ **Commercial Lock Protection:** Once a Proforma Invoice is approved or locked, core pricing, tax rates, and line items cannot be modified, preventing invoice tampering and billing discrepancies.
* 🚚 **Dispatch Synchronization:** The PI Status dashboard stays connected to factory operations throughout the order journey, automatically reflecting when goods are completed and fully dispatched (`FULLY_DISPATCHED`).

---

## 2. Key Components

When a Proforma Invoice is tracked in the system, it provides a comprehensive commercial and payment status record:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        PROFORMA INVOICE STATUS                         │
├────────────────────────────────────────────────────────────────────────┤
│  DOCUMENT IDENTIFICATION                                               │
│  • PI Number:     PI-2026-0082                  • Date:    09-10-2026  │
│  • Booking Ref:   BK-20261009-001               • Status:  [ APPROVED ]│
│  • Customer:      Apex Glass & Façades Pvt Ltd  • Balance: ₹0.00       │
│  • Project Site:  Skyline Tower Phase 2         • Locked:  YES         │
│                                                                        │
│  COMMERCIAL & TAX BREAKDOWN                                            │
│  • Total Quantity: 4 Pieces                     • Taxable: ₹16,186.37  │
│  • Total Area:     131.97 Sq.Ft (12.25 Sq.M)    • 18% GST: ₹2,913.55   │
│  • Basic Glass:    ₹14,516.70                   • Total:   ₹19,100.00  │
│                                                                        │
│  ADVANCE PAYMENT LEDGER                                                │
│  ┌────────────┬──────────┬──────────────┬──────────────┬────────────┐  │
│  │ Date       │ Mode     │ Reference    │ Bank / Note  │ Amount     │  │
│  ├────────────┼──────────┼──────────────┼──────────────┼────────────┤  │
│  │ 09-10-2026 │ NEFT     │ HDFC99382104 │ Axis Bank    │ ₹10,000.00 │  │
│  │ 09-10-2026 │ UPI      │ UPI/38201/01 │ GPay QR      │ ₹9,100.00  │  │
│  └────────────┴──────────┴──────────────┴──────────────┴────────────┘  │
│  • Total Advance Received: ₹19,100.00 (100% Settled)                   │
│                                                                        │
│  DOWNSTREAM FACTORY STATUS                                             │
│  • Work Orders Generated: WO-20261009-001 (Released to Cutting floor)  │
└────────────────────────────────────────────────────────────────────────┘
```

1. **PI Document Header:** Standardized commercial identification code (such as `PI-YYYY-XXXX`) tied directly to the originating Order Booking number and customer master record.
2. **Commercial & Tax Summary:** Detailed calculation of glass basic value, fabrication surcharges, freight, GST split (CGST+SGST or IGST), and rounded net invoice value.
3. **Advance Payment Ledger:** Real-time register tracking every advance installment, payment timestamp, payment instrument, and bank reference code.
4. **Commercial Status Indicators:** Visual color-coded badges indicating current workflow stage (`PENDING`, `APPROVED`, `LOCKED`, `FULLY_DISPATCHED`, `DECLINED`, `CANCELLED`).
5. **Shop Floor Authorization Link:** Direct pipeline triggering and releasing associated Work Orders to the factory floor upon commercial confirmation.

---

## 3. Step-by-Step Process

```text
  [1. Generate PI from Booking] ──► [2. Share Formal PI PDF] ──► [3. Collect Advance Payment]
                                                                                │
                                                                                ▼
  [6. Production & Dispatch Tracking] ◄── [5. Unlock Work Orders] ◄── [4. Record Advance & Approve PI]
```

### Step 1: Generate Proforma Invoice
Once an Order Booking is submitted as `CONFIRMED`, the billing operator opens the PI module and creates the Proforma Invoice with one click. Glass dimensions, quantities, and pricing carry over automatically.

### Step 2: Share Formal PI Document with Customer
The operator downloads or prints the official Proforma Invoice PDF containing your company brand, banking details, statutory numbers (GSTIN, PAN, CIN, MSME), UPI payment QR code, and customer site terms. The PDF is shared with the client via WhatsApp or email.

### Step 3: Customer Confirms & Transfers Advance
The customer checks measurements, approves rates and delivery schedules, and initiates an advance deposit (such as 30%, 50%, or 100% of order value) into your company bank account or via UPI.

### Step 4: Record Advance in Payment Ledger
The accounts operator opens the PI drawer, enters the received amount, selects the payment mode (such as `NEFT`, `RTGS`, `UPI`, `Cheque`, or `Cash`), enters the bank reference UTR number, and saves the entry.

### Step 5: Approve PI & Authorize Production
Once required advance terms or credit limits are satisfied, the finance manager marks the PI as `APPROVED`. Approving the PI immediately unlocks the linked factory Work Orders for production scheduling.

### Step 6: Monitor Execution & Complete Dispatch
As the factory cuts, polishes, drills, washes, and tempers the glass, the PI dashboard reflects manufacturing progress. Once all panels pass quality inspection and are loaded onto the delivery truck, the record advances to `FULLY_DISPATCHED`.

---

## 4. Field-by-Field Explanation

Here is what every single field in the PI Status module means:

---

### 4.1 Header & Customer Information
* **PI Number:** Unique sequential identifier for the Proforma Invoice (for example: `PI-2026-0082`).
* **PI Date:** The date when the proforma invoice was issued.
* **Order Booking Reference:** The originating booking code (such as `BK-20261009-001`). Clicking it opens the full booking summary modal.
* **Enquiry Reference:** Original customer enquiry number if the job began as an inquiry.
* **Customer / Party Name:** The registered client, fabricator, or dealer billed on the invoice.
* **Project Name:** The specific building site or architectural project location.

---

### 4.2 Financial & Commercial Summary
* **Total Pieces:** Total count of individual glass panels on the order.
* **Total Area (Sq.M & Sq.Ft):** Aggregated manufacturing and billable area across all panels.
* **Basic Glass Amount:** Base material and tempering charge before extra processing.
* **Fabrication & Processing Charges:** Surcharges for holes, cutouts, edge polishing, and special CNC cuts.
* **Freight & Handling:** Packaging, transport, or site crane delivery charges.
* **Taxable Amount:** Subtotal subject to statutory goods and services tax.
* **CGST & SGST:** Central and State GST (9% each for local sales inside your state).
* **IGST:** Integrated GST (18% for inter-state deliveries to other states).
* **Grand Total (₹):** Final net commercial invoice amount.

---

### 4.3 Advance Payment Ledger Fields
* **Payment Date:** Date when the payment landed in your company account or cash counter.
* **Amount Received (₹):** Actual funds received for this specific transaction.
* **Payment Mode:** Payment channel used by the customer:
  * `NEFT` / `RTGS`: Direct inter-bank electronic transfer.
  * `UPI`: Unified Payments Interface or QR code scan transfer.
  * `CHEQUE`: Bank cheque (subject to clearing).
  * `CASH`: Physical currency receipt.
* **Reference / UTR Number:** Bank transaction reference, cheque number, or UPI transaction ID.
* **Payment Notes:** Internal notes or bank account name where funds were credited.
* **Total Advance Received:** Aggregated sum of all recorded advance entries.
* **Balance Amount Due:** Outstanding balance calculated as `Grand Total - Total Advance Received`.

---

### 4.4 Status & Workflow Indicators
* **Status Badge:**
  * `PENDING`: Proforma issued, but awaiting customer confirmation or advance payment.
  * `APPROVED`: Commercial terms confirmed; advance received or credit approved. Work orders released.
  * `LOCKED`: Commercial terms locked against editing.
  * `FULLY_DISPATCHED`: Production complete and 100% of panels shipped from the plant.
  * `DECLINED`: Rejected by the customer or credit committee.
  * `CANCELLED`: Withdrawn or cancelled before factory production.
* **Is Locked Flag:** Internal security switch preventing updates to prices, taxes, or line items.
* **Remarks:** Notes on commercial negotiations, credit terms, or delivery coordination.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Why do we use a Proforma Invoice instead of issuing a final Tax Invoice immediately?
Under Indian GST law, a final Tax Invoice cannot be issued until goods are physically manufactured and ready for removal/dispatch. A Proforma Invoice serves as an official commercial quotation and advance demand note. It allows you to legally collect advance payments, confirm customer agreement on measurements, and lock pricing before incurring raw material and furnace costs.

### Q2: Can a customer pay the advance across multiple transactions?
**Yes.** The advance payment drawer allows recording multiple payment entries. For example, a customer may send ₹10,000 via NEFT on Monday and ₹9,100 via UPI on Tuesday. The ledger tracks each transaction separately and automatically recalculates the remaining balance.

### Q3: What happens to factory Work Orders if a PI remains in PENDING status?
Work Orders linked to a pending PI cannot be released for active factory floor processing without explicit management override. This safeguards the company against cutting expensive glass for customers who have not committed financially.

### Q4: Can a Proforma Invoice be modified after receiving an advance payment?
**No.** Once advance funds are recorded and the PI is approved, the record is locked to maintain financial compliance and prevent disputes. If dimensions or specifications change, the existing PI must be amended or cancelled with formal management approval and an adjusted proforma issued.

