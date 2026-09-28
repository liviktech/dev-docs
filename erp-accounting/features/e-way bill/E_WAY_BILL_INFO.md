# e-Way Bill: Complete Process & Field Guide

> **Target Audience:** Business owners, accounts team, delivery staff, and anyone who wants to understand how e-Way Bills work without complicated government jargon.

---

## 1. What is an e-Way Bill?

Imagine you are sending a valuable gift box to your friend in another city using a delivery truck. Before the truck leaves, the government wants to check **what is inside the truck**, **who is sending it**, **who is receiving it**, and **which vehicle is carrying it**.

An **e-Way Bill (Electronic Way Bill)** is simply a **Digital Delivery Passport** for moving goods across India. 

### Key Rules:
* 💰 **Money Limit:** It is mandatory whenever you transport goods worth **more than ₹50,000** in a single vehicle.
* 🚚 **Transport Modes:** Applies to goods moving by **Road (Trucks/Vans)**, **Rail (Trains)**, **Air (Planes)**, or **Ship (Boats)**.
* 🛡️ **Why it exists:** To prevent illegal tax evasion and ensure that goods being transported match the tax invoice.

---

## 2. The Two Parts of an e-Way Bill

An e-Way Bill has two sections: **Part A** and **Part B**.

```text
┌─────────────────────────────────────────────────────────┐
│                    OFFICIAL e-WAY BILL                  │
├─────────────────────────────────────────────────────────┤
│  PART A (Invoice & Tax Details)                         │
│  • Seller GSTIN & Buyer GSTIN                           │
│  • Invoice Number & Date                                │
│  • HSN Code, Item Value & GST Tax                       │
│  • From PIN Code & To PIN Code                          │
├─────────────────────────────────────────────────────────┤
│  PART B (Transportation Details)                        │
│  • Transport Mode (Road / Rail / Air / Ship)            │
│  • Vehicle Number (e.g. KA01AB1234)                     │
│  • Transporter ID (Courier Company ID)                  │
│  • Transport Document Number (LR / RR / AWB / BOL)      │
└─────────────────────────────────────────────────────────┘
```

1. **Part A (Goods & Tax Information):** Contains the invoice number, item description, total money value, seller/buyer GSTINs, and PIN codes.
2. **Part B (Vehicle & Driver Information):** Contains the vehicle registration number or courier receipt number. **An e-Way Bill is NOT valid for transport until Part B is filled!**

---

## 3. Step-by-Step e-Way Bill Process

```text
  [1. Create Invoice]  ──►  [2. Fill Transport Info]  ──►  [3. Generate 12-Digit EWB]
                                                                  │
                                                                  ▼
  [6. Delivery Completed] ◄──  [5. Update Vehicle (If needed)] ◄── [4. Goods in Transit]
```

### Step 1: Create Tax Invoice
The seller creates a regular tax invoice in the ERP system with items, quantities, and GST rates.

### Step 2: Fill Transport Details
The user selects the **Transport Mode** (Road, Rail, Air, Ship) and enters the **Vehicle Number** or **Transporter ID** and **Transport Document Number**.

### Step 3: Generate Government EWB
The ERP contacts the Government GST Portal. The government issues a unique **12-digit e-Way Bill Number** (e.g., `1210 0987 6543`) along with a QR code and an expiry date.

### Step 4: Goods in Transit
The driver carries a printed copy of the e-Way Bill or shows the digital QR code on a mobile phone to GST highway inspectors if stopped.

### Step 5: Update Vehicle (If needed during journey)
If the truck breaks down, or goods are shifted to another truck, train, or plane midway, the user or transporter **must update Part B** with the new vehicle/doc number.

### Step 6: Completion
Once the goods reach the customer safely, the journey is complete.

---

## 4. Field-by-Field Explanation

Here is what every single field in the e-Way Bill form means:

---

### 4.1 Addresses & PIN Codes
* **Consignor (Seller / From):** The company sending the goods. Must include legal name, GSTIN, street address, state code, and **PIN Code**.
* **Consignee (Buyer / To):** The customer receiving the goods. Must include legal name, GSTIN, delivery address, state code, and **PIN Code**.
* **PIN Code (Origin & Destination):** **Very Important!** The Government GST Portal automatically calculates the driving distance between the Seller's PIN Code and Buyer's PIN Code.

---

### 4.2 Distance (in Kilometers)
* **What it is:** The total road distance between the seller and buyer locations (e.g., `250 km`).
* **Why it matters:** The validity period (expiry date) of the e-Way Bill is decided based on this distance!
  * **Standard Goods:** 1 Day of validity for every **100 km** (or part thereof).
  * **Over Dimensional Cargo (Huge Machines/Pipes):** 1 Day of validity for every **20 km**.

---

### 4.3 Transport Mode
You can choose one of 4 options:

| Mode | Transport Vehicle Type | Required Identification |
| :--- | :--- | :--- |
| 🚚 **Road** | Trucks, Lorries, Vans, Tempo | Vehicle Number OR Transporter ID |
| 🚂 **Rail** | Goods Train (Indian Railways) | Railway Receipt (RR) Number |
| ✈️ **Air** | Cargo Airplane | Airway Bill (AWB) Tracking Number |
| 🚢 **Ship** | Cargo Ship / Boat / Vessel | Bill of Lading (BOL) Number |

---

### 4.4 Vehicle Number
* **What it is:** The official number plate on the truck or van carrying the goods (e.g., `KA01AB1234` or `MH12C5678`).
* **Format Rules:**
  * Starts with 2 state letters (e.g., `KA` for Karnataka, `MH` for Maharashtra, `DL` for Delhi).
  * Followed by RTO code digits (e.g., `01`, `12`).
  * Followed by optional series letters (e.g., `AB`, `C`).
  * Ends with 1 to 4 registration numbers (e.g., `1234`, `5678`).
* **Temporary/Defense Vehicles:** Temporary registration numbers or defense numbers (4 to 15 alphanumeric characters) are also accepted.

---

### 4.5 Transporter ID (Courier Company ID)
* **What it is:** Think of this as the **Courier Company's Badge Number**.
* **When to use it:** If you hire a logistics company (like Blue Dart or VRL Logistics) and you **don't know which exact truck number** will pick up your goods tomorrow morning.
* **How it works:** You enter the 15-digit GSTIN of the transport company. The transport company can then log into the GST portal and type in their truck number when loading the goods.

---

### 4.6 Transport Document Number & Date
* **What it is:** The **Tracking Receipt / Parcel Slip Number** given to you by the driver or transport company when you hand over your goods.
* **Different Names by Transport Mode:**
  * 🚚 **Road:** **LR Number** (Lorry Receipt) or **Bilty Number**.
  * 🚂 **Rail:** **RR Number** (Railway Receipt).
  * ✈️ **Air:** **AWB Number** (Airway Bill Tracking Number).
  * 🚢 **Ship:** **BOL Number** (Bill of Lading / Vessel IMO Code).

---

### 4.7 Key Output Fields (After Generation)

* **e-Way Bill Number (EWB No):** A unique 12-digit number generated by the Government portal (e.g., `1410 9876 5432`).
* **e-Way Bill Date:** The exact date and timestamp when the bill was created.
* **Valid Until:** The expiry date and time. Goods must reach their destination before this timestamp.
* **QR Code:** A scannable barcode containing verified details for highway officers.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Can I generate an e-Way Bill without a vehicle number?
**Yes!** If you fill in the **Transporter ID** (Courier Company ID), you can generate Part A. The courier company will then add the vehicle number in Part B before starting the journey.

### Q2: What if the truck breaks down midway?
You can open the active e-Way Bill in ERP or on the GST portal, click **"Update Vehicle"**, and enter the new truck's number.

### Q3: Can I cancel an e-Way Bill?
**Yes**, but only within **24 hours** of generation, and ONLY if the goods have not already been checked by a GST officer on the highway.

### Q4: What happens if the e-Way Bill expires before the truck reaches?
You can apply for a **Validity Extension** within 8 hours before or 8 hours after the expiry time, providing a reason such as traffic block, severe weather, or vehicle breakdown.
