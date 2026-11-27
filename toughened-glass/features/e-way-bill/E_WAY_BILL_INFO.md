# e-Way Bill: Complete Process & Field Guide

> **Target Audience:** Business owners, accounts team, dispatch staff, billing operators, and anyone at Rubine Glass who wants to understand how an e-Way Bill works for a Final Invoice without government or technical jargon.

---

## 1. What is an e-Way Bill?

It is like the permit a truck needs to carry goods on the road: the invoice says what was sold, the e-Way Bill says that this particular load is allowed to travel from here to there in this vehicle.

An **e-Way Bill** is an electronic permit generated on the government's e-Way Bill portal for a consignment of goods. It has a 12-digit number and a validity date. A road inspector can check it on the spot.

### Key Rules:
* 🚚 **Needed when goods move:** An e-Way Bill is needed when goods above the government value limit (generally ₹50,000) are moved by road, rail, air or ship. Rubine does not block smaller invoices, so the decision stays with accounts.
* 💼 **Made from the Dispatch page:** In Dispatch, open the actions menu of an approved dispatch and choose **Generate e-Way Bill**. An e-Invoice is not required first.
* ⏱️ **Validity depends on distance:** One day of validity for every 200 km, or part of it, with a minimum of one day. Glass is fragile and trips are short, so most bills are valid for one day.
* 🛡️ **One live e-Way Bill per invoice:** A second one cannot be made while one is active.
* 🧪 **Test mode is not real:** In MOCK mode the number looks real but is not registered anywhere, and the printed slip says so in red.

---

## 2. The Two Parts of an e-Way Bill

```text
┌────────────────────────────────────────────────────────┐
│                   e-WAY BILL 2418 9032 1104            │
├────────────────────────────────────────────────────────┤
│  PART A: WHAT AND WHERE (from the invoice)             │
│  • From: Rubine Glass, Trichy        GSTIN, pincode    │
│  • To:   Customer ship-to address    GSTIN, pincode    │
│  • Goods: product, HSN, quantity, taxable value, GST   │
│  • Document: invoice number and date                   │
│                                                        │
│  PART B: HOW (entered when generating)                 │
│  • Mode: Road       • Distance: 120 km                 │
│  • Vehicle: TN45AE8219                                 │
│  • Transporter ID (only if a transporter is used)      │
└────────────────────────────────────────────────────────┘
```

1. **Part A:** Comes from the Final Invoice and the master data. Nobody types it again.
2. **Part B:** The transport details. The vehicle is filled in from the invoice's Dispatch when one is assigned. The distance has to be entered.

---

## 3. Step-by-Step Process

```text
  [1. Invoice          ──►  [2. Dispatch assigns  ──►  [3. Enter distance
      CONFIRMED]                the vehicle]               and generate]
                                                               │
                                                               ▼
  [6. Cancel if the     ◄──  [5. Goods travel     ◄──  [4. EWB number comes
      load is cancelled]        (update vehicle           back, print the slip]
                                 if needed)]
```

### Step 1: The Final Invoice is CONFIRMED
A dispatch exists only after the Final Invoice is confirmed.

### Step 2: The vehicle is assigned
When Dispatch is approved, the vehicle type, vehicle number and driver are saved. The e-Way Bill uses that vehicle automatically.

### Step 3: Enter the distance and generate
The user picks the transport mode, enters the distance in km, and confirms the vehicle. The system checks everything first, then sends it to the government.

### Step 4: The number comes back
The 12-digit e-Way Bill number, the date and the valid-upto date are saved. The number is printed on the Gate Pass. In the same actions menu, **Download e-Way Bill** opens the slip, which can be printed or saved as a PDF, with a barcode of the number.

### Step 5: Goods travel (update the vehicle if needed)
If the truck breaks down and another one takes the load, the vehicle number is updated on the same e-Way Bill. No new bill is needed.

### Step 6: Cancel if the load is cancelled
Allowed only while the goods have not started moving.

---

## 4. Field-by-Field Explanation

---

### 4.1 Addresses and Pincodes
* **From:** The company address and pincode from the Company profile, and the company GSTIN.
* **To:** The party's Delivery (or Shipping) address, or its billing address if there is no delivery address. The party GSTIN is used.
* **Pincodes:** Both must be valid 6-digit pincodes. The government uses them to check the distance.

---

### 4.2 Distance
* **Distance in km:** Entered by the user, from 1 to 4000. It decides the validity of the bill. A wrong distance is a common reason for a bill to expire before the goods arrive, so enter the real road distance.

---

### 4.3 Transport Mode
* **Road, Rail, Air or Ship:** Road needs a vehicle number or a transporter ID. Rail, Air and Ship need the transport document number (RR number, airway bill or bill of lading).

---

### 4.4 Vehicle Number
* **Format:** An Indian road number without spaces, for example TN45AE8219. Spaces and lower case are fixed automatically.
* **Default:** The vehicle assigned on the Dispatch. It can be changed for this e-Way Bill.

---

### 4.5 Transporter ID
* **What it is:** The 15-character GSTIN of the transport company, if one is hired. Not needed when Rubine uses its own vehicle.

---

### 4.6 Key Output Fields (After Generation)
* **e-Way Bill number:** 12 digits.
* **Generated date:** When the government issued it.
* **Valid upto:** The last date and time the goods may be on the road with this bill.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Can I generate an e-Way Bill without a vehicle number?
For road transport you need either a vehicle number or a transporter ID. If the Dispatch has no vehicle yet, assign it first or type the number in the dialog.

### Q2: What if the truck breaks down midway?
Use Update vehicle on the same e-Way Bill and enter the new vehicle number. A new bill is not needed.

### Q3: Can I cancel an e-Way Bill?
Yes, while the goods have not started moving. After that it cannot be cancelled.

### Q4: What happens if the bill expires before the truck arrives?
The goods are then travelling without a valid bill. The government allows extending the validity in some cases, but extension is not built into this system yet, so it has to be done on the government portal.

### Q5: Do I need an e-Invoice before an e-Way Bill?
No. They are separate. Both use the same customer, address and HSN data.

### Q6: Where do I find the e-Way Bill after it is made?
Three places: the Gate Pass of that dispatch shows its number, the Dispatch actions menu now says **Download e-Way Bill**, and the printed tax invoice shows it in the transport block.
