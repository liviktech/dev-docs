# e-Way Bill Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.
> **Purpose:** A concise reference detailing required fields, their originating modules and database columns, validation rules, and the exact steps to run, extend and wire up the e-Way Bill for Final Invoices in the Rubine Toughened Glass project. The e-Way Bill shares its provider layer, validation, audit log, setup and open UI decisions with e-Invoice. Read the [e-Invoice developer guide](../e-invoice/E_INVOICE_DEVELOPER_GUIDE.md) first, especially Section 5 (what is left to do and the open decisions). This guide covers only what is specific to the e-Way Bill. It is built, checked, and wired into the Dispatch page and the Gate Pass (Section 5).

---

## 1. Required Fields & Originating Modules

To generate an e-Way Bill, the system aggregates data across 4 sources: **Company profile (consignor)**, **Parties (consignee)**, **Final Invoice lines (goods)**, and **transport details** typed at generation (with the **Dispatch** as the default for the vehicle).

### 1.1 Company / Consignor (From)
*Same source as e-Invoice seller. See e-Invoice guide 1.1.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **GSTIN** | Yes | Company profile (`/admin/company`) | `core.company.gst_number` | Also decides which government portal login is used |
| **Trade Name** | Yes | Company profile | `core.company.name` | |
| **Address, Place** | Yes | Company profile | `master.address.address_line_1`, `address_line_2`, `city`, `state` | Address cut to 100 characters |
| **PIN Code** | Yes | Company profile | `master.address.pincode` | 6 digits |
| **State Code** | Yes | Auto-derived | Derived | First 2 digits of the GSTIN |

---

### 1.2 Customer / Consignee (To)
*Same source as e-Invoice buyer, but the address is the Ship To address. See e-Invoice guide 1.2.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **GSTIN** | Yes | Parties, address tab | `master.address.gstin` | The buyer's GSTIN, even when the ship-to address is a different site |
| **Trade Name** | Yes | Parties | `master.party.name` | |
| **Ship To Address, Place** | Yes | Parties, address tab | `master.address.*` where `address_type` is DELIVERY or SHIPPING | Falls back to the billing address |
| **PIN Code** | Yes | Parties, address tab | `master.address.pincode` | 6 digits. The government uses it to check the distance |
| **State Code** | Yes | Auto-derived | Derived | First 2 digits of the GSTIN |

---

### 1.3 Transport Details (Part B)
*Typed in the Generate e-Way Bill dialog. The vehicle defaults to the Dispatch.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Transport Mode** | Yes | Generate dialog | `invoice.gst_ewaybill.transport_mode` | `Road`, `Rail`, `Air` or `Ship` |
| **Distance (km)** | Yes | Generate dialog | `invoice.gst_ewaybill.distance_km` | Whole number 1 to 4000. Decides the validity |
| **Vehicle Number** | Road: this or Transporter ID | Generate dialog, default from Dispatch | `invoice.gst_ewaybill.vehicle_number`, default `invoice.dispatch.vehicle_number` | Upper case, no spaces, pattern `^[A-Z]{2}[0-9]{1,2}[A-Z]{0,3}[0-9]{4}$` |
| **Transporter ID** | Optional | Generate dialog | `invoice.gst_ewaybill.transporter_id` | 15 character GSTIN |
| **Transport Doc No.** | Rail, Air, Ship | Generate dialog | `invoice.gst_ewaybill.transport_doc_number` | The dialog requires it for non-road. The backend does not, the portal decides |

---

### 1.4 Goods and Values (Part A)
*Taken from the same validated snapshot as e-Invoice, so the two documents always agree. See e-Invoice guide 1.3 and 1.4.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Product, HSN, Quantity, Unit** | Yes | Final Invoice items, Product master | `invoice.final_invoice_item.*`, `master.product.hsn_code` | Unit is `SQM`, `PCS` or `OTH` |
| **Taxable Value per line** | Yes | Auto-calculated | Derived | After the share of discount |
| **CGST, SGST, IGST rate per line** | Yes | Auto-derived | Derived | Inter-state: IGST only. Intra-state: half the rate each for CGST and SGST |
| **Document No. and Date** | Yes | Auto | `invoice.final_invoice.final_invoice_code`, `invoice_date` | Document type is a tax invoice, supply type outward |
| **Total Invoice Value** | Yes | Auto | `invoice.final_invoice.total_amount` | |

---

### 1.5 What Gets Stored
*Written by the e-Way Bill module.*

| Field | Database Column | Notes |
|---|---|---|
| **EWB Number** | `invoice.gst_ewaybill.ewb_number` | 12 digits |
| **EWB Date / Valid Upto** | `invoice.gst_ewaybill.ewb_date`, `valid_upto` | Valid upto is set by the portal. The mock uses 1 day per 200 km or part of it |
| **Status** | `invoice.gst_ewaybill.status` | `Generated`, `Cancelled` or `Failed` |
| **Provider** | `invoice.gst_ewaybill.provider` | `MOCK` or `SANDBOX` |
| **Audit log** | `invoice.gst_api_log` | Operations `EWAYBILL_GENERATE`, `EWAYBILL_CANCEL`, `EWAYBILL_UPDATE_VEHICLE` |

---

## 2. Eligibility & Validation Rules Checklist

Before running e-Way Bill, the system verifies:
1. **Invoice data:** All the e-Invoice checks in the e-Invoice guide, Section 2, rules 1 to 8 (the invoice may be DRAFT or CONFIRMED, but a Dispatch only exists once it is CONFIRMED). The e-Way Bill uses the same snapshot, so a missing HSN, pincode or GSTIN blocks it with the same plain message. An existing e-Invoice is **not** required.
2. **No duplicate active bill:** The service returns `409 DUPLICATE_ACTIVE_EWAYBILL`, and the partial unique index `uq_gst_ewaybill_active_per_invoice` guarantees it.
3. **Distance:** A whole number from 1 to 4000 (Zod schema in `e-way-bill.schema.ts`).
4. **Vehicle format:** If given, it must match the road pattern above after upper-casing and removing spaces.
5. **Road transport:** A vehicle number (typed or from the Dispatch) or a transporter ID is required. Otherwise `MISSING_TRANSPORT_DETAILS`.
6. **Transporter ID:** 15 characters if given.
7. **Cancel and update vehicle:** Only an active (`Generated`) bill with a number can be cancelled or updated, else `NO_ACTIVE_EWAYBILL`. The cancel reason is 3 to 100 characters. An update needs at least one of vehicle number, transport doc number or mode.
8. **Value limit:** The government does not require a bill below about ₹50,000. This is **not enforced**. Accounts decide.

---

## 3. Implementation Guide: Switching from MOCK to the real Government service

The e-Way Bill uses the same provider layer, the same `.env` keys and the same factory as e-Invoice, so switching the provider is done once for both. Follow Section 3 of the [e-Invoice guide](../e-invoice/E_INVOICE_DEVELOPER_GUIDE.md). The only e-Way Bill specific points are:

* **Credentials:** e-Way Bill uses its own portal login, `NIC_EWB_USERNAME` and `NIC_EWB_PASSWORD`, registered on ewaybillgst.gov.in. If the e-Invoice login keys are empty, the Sandbox adapter reuses the e-Way Bill login for e-Invoice, not the other way round.
* **Methods a new adapter must implement:** `generateEwayBill`, `cancelEwayBill`, `updateVehicle`, `getEwayBill` on the `GstProvider` interface. The input type `GenerateEwayBillInput` is already validated and complete. Map it to the vendor's request, and throw `GstProviderError` for any rejection.
* **Working example:** `generateEwayBill`, `cancelEwayBill`, `updateVehicle` and `getEwayBill` in `src/shared/gst/sandbox-gst.provider.ts`.

```typescript
// Shape of the call a new adapter receives (already validated). Map it, call the vendor, return the result.
async generateEwayBill(input: GenerateEwayBillInput): Promise<EwayBillResult> {
  const data = await vendorCall("/ewaybill", { /* map input.from*, input.to*, input.lineItems, input.transport* */ });
  if (!data.ewayBillNo) throw new GstProviderError(data.message ?? "Portal rejected the bill", data.errorCode ?? "EWB_ERROR", data);
  return { ewbNumber: String(data.ewayBillNo), ewbDate: new Date(data.ewayBillDate), validUpto: new Date(data.validUpto), raw: data };
}
```

---

## 4. What Already Works (Zero Changes Needed)

When the environment is set (or a new adapter is added), the following work automatically without modification:

1. **Database Tables:** `invoice.gst_ewaybill` stores every attempt with its status, vehicle and distance, and `invoice.gst_api_log` stores every call.
2. **Pre-flight Validation:** Shared snapshot validation, plus the Zod checks on distance, vehicle and transporter ID.
3. **Vehicle default:** If no vehicle is typed, the Dispatch vehicle (`invoice.dispatch.vehicle_number`) is used automatically.
4. **Duplicate Prevention:** One live bill per invoice, enforced in the service and by a partial unique index.
5. **Failure handling:** A failed attempt is saved as a `Failed` row with its error, and the dialog shows the message so the user can fix it and retry without retyping.
6. **Rights:** `e_way_bill_view`, `e_way_bill_add`, `e_way_bill_edit` (update vehicle) and `e_way_bill_status_update` (cancel), granted to ACCOUNTS and SALES_MANAGER by seed `019_gst_compliance_rbac.sql`.
7. **Frontend:** The generate dialog, the slip with a Code 128 barcode (preview, print, PDF), and the number on the Gate Pass. Update vehicle, cancel and the audit log are in the optional `GstCompliancePanel` and are not in the Dispatch menu yet.

---

## 5. How it is wired into the app

### 5.1 The behaviour that was decided
1. **Generate from the Dispatch actions menu.** The kebab on each Dispatch row has **Generate e-Way Bill**. It is disabled until the dispatch is approved (same rule as Create Gate Pass).
2. **It shows on the Gate Pass.** The Gate Pass prints "e-Way Bill No" and the valid-upto date beside the driver.
3. **It downloads separately.** Once a bill exists, the same menu item becomes **Download e-Way Bill**, which opens the slip (preview, print, PDF).

### 5.2 Where each piece lives
| Behaviour | File | What it does |
|---|---|---|
| Menu items | `dispatch/components/dispatch-table.tsx` | Adds Generate or Download in the kebab, driven by the `ewayBillNumbers` map and two callbacks. An item shows only if its callback is passed |
| Page wiring and rights | `dispatch/dispatch-page.tsx` | Calls `GET /e-way-bills?finalInvoiceIds=...` for the rows on screen, builds the map, passes the callbacks only to roles with `e_way_bill_add` and `e_way_bill_view` |
| Generate dialog | `e-way-bill/components/generate-ewaybill-dialog.tsx` | Wraps `GenerateEwayBillModal` with its own mutation. Defaults the vehicle to the Dispatch vehicle, keeps the dialog open with the portal's message on failure, toasts on success |
| Slip | `e-way-bill/components/ewaybill-slip-modal.tsx` | Preview, Print and Download PDF |
| Gate Pass | `dispatch/components/gate-pass-sheet.tsx`, `gate-pass-modal.tsx` | The modal reads the invoice's live bill with `ewayBillQueryOptions` and passes it to the sheet. Read at view time, so a bill generated after the Gate Pass was created still appears |
| List endpoint | `modules/e-way-bill/e-way-bill.routes.ts` | `GET /e-way-bills?finalInvoiceIds=a,b,c` (max 100) returns the live bill of each invoice. It is a separate call so the Dispatch list query is untouched, and a missing migration cannot break the Dispatch page |
| The tax invoice | `final-invoices/components/final-invoice-preview-modal.tsx` | The printed invoice shows the e-Way Bill number, vehicle and removal date in its transport block when a bill exists |

### 5.3 Still open
| # | Decision | Notes |
|---|---|---|
| 1 | Update vehicle and Cancel from the Dispatch menu | Built (`UpdateEwayBillVehicleModal`, cancel mutation) and available in `GstCompliancePanel`, but not added to the kebab. Add them if accounts want them there |
| 2 | Generate for small invoices | The ₹50,000 limit is not enforced. Accounts decide |
| 3 | Distance | Typed by hand. Calculating it needs a maps or pincode service |

### 5.4 One-time setup and tests
Setup is the same as the e-Invoice guide, Section 5.5, including the migration and seed. The e-Way Bill needs the same company, party and HSN data.

1. Open Dispatch, find an **approved** dispatch. Open its kebab. Expect **Generate e-Way Bill** enabled. For a pending dispatch it is disabled with "Approve the dispatch first".
2. Click it, enter a distance, keep or change the vehicle, Generate. Expect a toast, and the kebab item now reads **Download e-Way Bill**.
3. Click Download e-Way Bill. Expect the slip with a barcode and a red "SIMULATED (MOCK)" note. Print and Download PDF work.
4. Open the Gate Pass of that dispatch. Expect "e-Way Bill No" printed next to the driver.
5. Try again with a missing vehicle and a Road mode. Expect the dialog to stay open with a message.
6. API checks: `POST /api/v1/final-invoices/<id>/e-way-bill` twice gives 409 `DUPLICATE_ACTIVE_EWAYBILL`. `POST .../update-vehicle` and `POST .../cancel` work as in Section 6.

---

## 6. API Reference

Base path `/api/v1`. All routes need a signed-in user. `:id` is the `invoice.final_invoice.id` UUID. Responses use the standard `{ success, message, data }` envelope with snake_case row fields.

| Method and path | Right | Purpose |
|---|---|---|
| `POST /final-invoices/:id/e-way-bill` | `e_way_bill_add` | Generate. Body `{ transportMode, distanceKm, vehicleNumber?, transporterId?, transportDocNumber? }`. 201 |
| `GET /e-way-bills?finalInvoiceIds=a,b,c` | `e_way_bill_view` | The live bill of each invoice (up to 100 ids). Used by the Dispatch table |
| `GET /final-invoices/:id/e-way-bill` | `e_way_bill_view` | Latest attempt (Generated, Cancelled or Failed), or null |
| `POST /final-invoices/:id/e-way-bill/cancel` | `e_way_bill_status_update` | Body `{ reason }` |
| `POST /final-invoices/:id/e-way-bill/update-vehicle` | `e_way_bill_edit` | Body `{ vehicleNumber?, transportDocNumber?, transportMode? }`, at least one |
| `GET /final-invoices/:id/e-way-bill/logs` | `e_way_bill_view` | Provider call audit log, newest first |

---

## 7. File Map

**Backend** (`rubine-toughened-glass-backend/src`)

| File | Purpose |
|---|---|
| `modules/e-way-bill/e-way-bill.routes.ts`, `controller.ts`, `schema.ts`, `types.ts` | HTTP layer and Zod validation (new) |
| `modules/e-way-bill/e-way-bill.service.ts` | Validation, provider call, storage, audit log for generate, cancel, update vehicle (new) |
| `modules/e-way-bill/e-way-bill.repository.ts` | SQL for `invoice.gst_ewaybill` (new) |
| `shared/gst/*` | Provider interface, factory, mock, sandbox, snapshot, audit log. Shared with e-Invoice (new) |
| `database/migrations/063_gst_einvoice_eway_bill.sql`, `database/seeds/019_gst_compliance_rbac.sql` | Shared with e-Invoice (new) |
| `api/route-mapper.ts` | Mounts `/final-invoices/:finalInvoiceId/e-way-bill` and `/e-way-bills` (edited) |

**Frontend** (`rubine-toughened-glass-frontend/src/features/e-way-bill`)

| File | Purpose |
|---|---|
| `types.ts`, `api.ts`, `queries.ts`, `utils.ts` | Types, API calls, hooks, and the same transport checks as the backend |
| `components/generate-ewaybill-modal.tsx` | Generate dialog. Form starts clean each time it opens |
| `components/update-ewaybill-vehicle-modal.tsx` | Change the vehicle on a live bill |
| `components/ewaybill-slip.tsx` | The slip layout with barcode, plus `EWAYBILL_SLIP_CSS` |
| `components/ewaybill-slip-modal.tsx` | Preview, Print, Download PDF |
| `components/generate-ewaybill-dialog.tsx` | Generate dialog with its own mutation, used by the Dispatch page |
| `../e-invoice/components/gst-compliance-panel.tsx` | An optional card with every action, not used by the current screens |
| `../dispatch/*` | **Edited.** Menu items, dialogs and the Gate Pass line. See Section 5.2 |

---

## 8. Known Gaps and Things to Know

1. **Data prerequisites are shared with e-Invoice**, including the mistyped seed GSTIN and PAN, and empty HSN for non-toughened products. See e-Invoice guide Section 8.
2. **Extending validity, multi-vehicle and consolidated bills** are not built.
3. **Consignor and consignee state codes** are taken from the GSTIN. The "actual" ship-to state is the ship-to address state. The Sandbox adapter sends the same state code for both because the Ship To state is normally the buyer's state.
4. **Only outward supply of goods** (supply type outward, sub type supply, regular transaction) is sent.
5. **A MOCK number is not registered** and is marked in red on the slip.
6. **Not checked against a real database or the Sandbox service.** The shared checks are listed in the e-Invoice guide, Section 9. For the e-Way Bill specifically, the request mapping was type-checked and the mock provider was exercised only through the shared snapshot. The generate, cancel and update-vehicle endpoints were not called, and the Dispatch menu, the dialog, the slip and the Gate Pass line were not opened in the app. They are verified by type check and lint only.
