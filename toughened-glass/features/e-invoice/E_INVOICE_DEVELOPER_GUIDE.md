# e-Invoice Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.
> **Purpose:** A concise reference detailing required fields, their originating modules and database columns, validation rules, and the exact steps to run, extend and wire up e-Invoice (IRN) for Final Invoices in the Rubine Toughened Glass project. The backend and the frontend are built, checked and wired into the Final Invoice and Dispatch screens. Section 5 says exactly where, and what still needs a decision.

---

## 1. Required Fields & Originating Modules

To generate an e-Invoice, the system aggregates data across 4 sources: **Company profile (seller)**, **Parties (buyer)**, **Products and Final Invoice items (lines)**, and the **Final Invoice header**. Nothing is typed again for e-Invoice. All of it already exists in the app, except the HSN code (see 1.3).

### 1.1 Company / Seller Details
*Captured on the Company profile page (`/admin/company`) and stored on `core.company` and its linked `master.address`.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Legal Name** | Yes | Company profile | `core.company.name` | Printed and sent as the seller legal name |
| **Trade / Brand Name** | Optional | Company profile | `core.company.brand_name` | Sent as trade name. Falls back to the legal name |
| **GSTIN** | Yes | Company profile | `core.company.gst_number` | 15 characters. Falls back to `master.address.gstin` of the company address if empty |
| **Address** | Yes | Company profile | `master.address.address_line_1`, `address_line_2`, `city`, `state` | Joined with commas. The IRP allows 100 characters, the rest is cut |
| **City (place)** | Yes | Company profile | `master.address.city` | At least 3 characters. Falls back to state |
| **PIN Code** | Yes | Company profile | `master.address.pincode` | Exactly 6 digits, no leading 0 |
| **State Code** | Yes | Auto-derived | Derived | First 2 digits of the GSTIN (33 for Tamil Nadu) |

---

### 1.2 Customer / Buyer Details
*Captured on the Parties page (`/admin/parties`), in the party's addresses. The party is found through the Final Invoice's Proforma Invoice (`invoice.proforma_invoice.party_code`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Party Name** | Yes | Parties | `master.party.name` | Sent as buyer legal and trade name |
| **GSTIN** | Yes | Parties, address tab | `master.address.gstin` (via `master.party_address`) | 15 characters. A party with no GSTIN is B2C and cannot have an e-Invoice |
| **Billing Address** | Yes | Parties, address tab | `master.address.*` where `master.party_address.address_type` is BILLING or OFFICE | Falls back to the default address, then the first address |
| **Ship To Address** | Yes | Parties, address tab | `master.address.*` where `address_type` is DELIVERY or SHIPPING | Falls back to the billing address. GSTIN is always the buyer's |
| **PIN Code** | Yes | Parties, address tab | `master.address.pincode` | Exactly 6 digits |
| **City (place)** | Yes | Parties, address tab | `master.address.city` | At least 3 characters. Falls back to state |
| **State Code** | Yes | Auto-derived | Derived | First 2 digits of the GSTIN. Decides CGST+SGST or IGST |

---

### 1.3 Line Items and Charges
*Items come from the Final Invoice. Charges and discount come from the Proforma Invoice and the Order Booking.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Product Name** | Yes | Final Invoice items | `invoice.final_invoice_item.description`, else `master.product.name` | Sent as the item description |
| **HSN Code** | Yes | **No screen yet.** Set with SQL, see Section 5.4 | `master.product.hsn_code` (new, migration 063) | 4 to 8 digits. Back-filled to `70071900` for toughened glass only. Annealed Glass stays empty on purpose |
| **Quantity** | Yes | Final Invoice items | `invoice.final_invoice_item.chargeable_sqm`, else `quantity` | SQM when chargeable SQM is present, otherwise pieces. Pieces and SQM are both kept for printing |
| **Unit (UoM)** | Yes | Auto | Derived | `SQM` for glass priced per sqm, `PCS` for pieces, `OTH` for charge lines |
| **Unit Price** | Yes | Auto-calculated | `invoice.final_invoice_item.amount / quantity` | Rounded to 3 decimals so Qty x Price matches the amount |
| **Gross Amount** | Yes | Final Invoice items | `invoice.final_invoice_item.amount` | Before discount |
| **Charge lines** | Optional | Proforma Invoice | `invoice.proforma_invoice_charge` | Multiplied by this invoice's share of the PI (item value / PI item value). Use the HSN of the first product |
| **Discount** | Optional | Order Booking | `sales.order_booking.discount_type`, `discount_value` | Prorated, then spread over all lines by value so every line adds up |
| **Taxable Amount** | Yes | Auto-calculated | Derived | Gross amount minus its share of the discount |
| **GST Rate** | Yes | Auto-derived | Derived | See 1.4. GST is not stored on a Final Invoice |
| **CGST / SGST / IGST** | Yes | Auto-calculated | Derived | Intra-state: CGST and SGST each half the rate. Inter-state: IGST full rate |

---

### 1.4 Invoice Header and Totals
*Captured when the Final Invoice is created and approved.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Invoice Number** | Yes | Auto-generated | `invoice.final_invoice.final_invoice_code` | Sent as the IRP document number. Max 16 characters of letters, digits, `/` and `-`. Cannot start with `0`, `/` or `-` |
| **Invoice Date** | Yes | Final Invoice | `invoice.final_invoice.invoice_date` | Sent as DD/MM/YYYY |
| **Document / Supply Type** | Yes | Fixed | Fixed | Always `INV` and `B2B`. Credit notes, debit notes, exports and SEZ are not built |
| **Status** | Yes | Final Invoice | `invoice.final_invoice.status` | `DRAFT` or `CONFIRMED`. The e-Invoice is generated when the invoice is created (DRAFT). `CANCELLED` and `DECLINED` are refused |
| **Total Invoice Value** | Yes | Final Invoice | `invoice.final_invoice.total_amount` | Items + charges - discount + GST |
| **GST Rate** | Yes | Auto-derived | Derived | `total_amount` minus taxable value gives the tax. The rate is the standard slab (0, 0.1, 0.25, 1, 1.5, 3, 5, 7.5, 12, 18, 28) that fits within 2 rupees. No slab fits means `TOTALS_MISMATCH` |
| **Round Off** | Yes | Auto-calculated | Derived | `total_amount` minus (taxable + GST). Must be within plus or minus 2 |

---

### 1.5 What Gets Stored
*Written by the e-Invoice module. Nothing here is typed by a user.*

| Field | Database Column | Notes |
|---|---|---|
| **IRN** | `invoice.gst_einvoice.irn` and mirrored on `invoice.final_invoice.einvoice_irn` | Unique across the table. Mirror is cleared when the IRN is cancelled |
| **Ack Number / Date** | `invoice.gst_einvoice.ack_number`, `ack_date` and mirrored on `final_invoice.einvoice_ack_no`, `einvoice_ack_date` | |
| **Signed QR** | `invoice.gst_einvoice.signed_qr_code` | The text drawn as the QR on the printed invoice |
| **Signed Invoice JSON** | `invoice.gst_einvoice.signed_invoice` | Kept as returned by the IRP |
| **Status** | `invoice.gst_einvoice.status` | `Generated`, `Cancelled` or `Failed` |
| **Provider** | `invoice.gst_einvoice.provider` | `MOCK` or `SANDBOX`. A `MOCK` row is simulated |
| **Audit log** | `invoice.gst_api_log` | Every provider call, success or failure, with credentials masked |

---

## 2. Eligibility & Validation Rules Checklist

Before running e-Invoice, the system verifies the following, in `src/shared/gst/gst-invoice-snapshot.ts`. Every failure is a 400 with a plain message that says which master data to fix. Nothing is sent to the provider until all pass.
1. **Status:** The Final Invoice status is `DRAFT` or `CONFIRMED`. A `CANCELLED` or `DECLINED` invoice is refused. Code `INVOICE_NOT_ELIGIBLE`.
2. **Invoice number:** `final_invoice_code` matches `^[A-Za-z1-9][A-Za-z0-9/-]{0,15}$`. Code `INVALID_INVOICE_NUMBER`.
3. **Party and items:** The invoice has a party and at least one item. Codes `MISSING_PARTY`, `EMPTY_LINE_ITEMS`.
4. **Seller:** A valid GSTIN, an address line, a 6-digit pincode, and a city of at least 3 characters. Codes `MISSING_SELLER_GSTIN`, `INVALID_SELLER_GSTIN`, `MISSING_SELLER_ADDRESS`, `INVALID_SELLER_PINCODE`, `MISSING_SELLER_CITY`.
5. **Buyer:** A valid GSTIN, a 6-digit pincode, a city of at least 3 characters, for both the billing and the ship-to address. Codes `MISSING_BUYER_GSTIN`, `INVALID_BUYER_GSTIN`, `INVALID_BUYER_PINCODE`, `MISSING_BUYER_CITY`, `MISSING_BUYER_ADDRESS`.
6. **State consistency:** The state code is read from the GSTIN prefix. Seller state not equal to buyer state means inter-state, so IGST and no CGST or SGST. Equal means CGST and SGST and no IGST.
7. **HSN:** Every product has a 4 to 8 digit `hsn_code`. Codes `MISSING_HSN`, `INVALID_HSN`.
8. **Totals:** Taxable value is above zero, the GST rate is a standard slab that fits within 2 rupees, and round off is within plus or minus 2. Codes `ZERO_TAXABLE_VALUE`, `TOTALS_MISMATCH`.
9. **No duplicate active IRN:** The service returns `409 DUPLICATE_ACTIVE_EINVOICE`, and the partial unique index `uq_gst_einvoice_active_per_invoice` makes it a database guarantee even under a double click.
10. **Cancel:** Only a `Generated` IRN can be cancelled (`NO_ACTIVE_EINVOICE` otherwise). The government allows it only within 24 hours. The provider's error is passed back to the user if the window is over. Reason `4` (Others) needs remarks of at least 3 characters.

---

## 3. Implementation Guide: Switching from MOCK to the real Government service

The implementation uses the **Provider Pattern**. The database layer, validation, payload building, audit logging and the frontend are already complete and independent of the provider. The provider is chosen by `GST_PROVIDER` and only `src/shared/gst/gst-provider.factory.ts` creates one.

Two real paths exist. `SANDBOX` is fully written (`sandbox-gst.provider.ts`, calling Sandbox.co.in) and only needs credentials. `DIRECT` is reserved for a direct IRP/GSP adapter that does not exist yet, so for now it falls back to `SANDBOX`.

```text
┌─────────────────────────────────────────────────────┐
│     Frontend: Final Invoice preview, Dispatch menu  │
│          (NO CHANGES NEEDED - COMPLETE)             │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│  e-invoice.service.ts + gst-invoice-snapshot.ts     │
│          (NO CHANGES NEEDED - COMPLETE)             │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│  1. .env (EDIT: GST_PROVIDER and credentials)       │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│  2. gst-provider.factory.ts (EDIT only for a new    │
│     adapter, not needed for SANDBOX)                │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│  3. <new>.provider.ts (CREATE only for a new        │
│     adapter, not needed for SANDBOX)                │
└─────────────────────────────────────────────────────┘
```

---

### Step 1: Configure Environment Variables
Edit [`rubine-toughened-glass-backend/.env`](file:///D:/Livik-projects/Rubine-glass/rubine-toughened-glass-backend/.env). The keys are already listed in `.env.example` and validated in `src/config/env.ts`.

```env
# MOCK (default) needs nothing. SANDBOX calls Sandbox.co.in.
GST_PROVIDER=SANDBOX
GST_ENVIRONMENT=TEST

# Sandbox account (test-api.sandbox.co.in for testing, api.sandbox.co.in for live)
GSP_API_URL=https://test-api.sandbox.co.in
SANDBOX_API_KEY=your_sandbox_api_key
SANDBOX_API_SECRET=your_sandbox_api_secret

# Logins of the seller GSTIN on the government portals (not Sandbox logins)
NIC_EINVOICE_USERNAME=your_einvoice_portal_username
NIC_EINVOICE_PASSWORD=your_einvoice_portal_password
NIC_EWB_USERNAME=your_ewaybill_portal_username
NIC_EWB_PASSWORD=your_ewaybill_portal_password
```

Restart the backend. With `GST_PROVIDER=SANDBOX` and a missing key the call fails with `GST_PROVIDER_NOT_CONFIGURED`, and the audit log says which key is missing. Change nothing else for Sandbox.

---

### Step 2: Implement a new Provider (only if not using Sandbox)
Create [`rubine-toughened-glass-backend/src/shared/gst/<name>-gst.provider.ts`](file:///D:/Livik-projects/Rubine-glass/rubine-toughened-glass-backend/src/shared/gst).

Implement the `GstProvider` interface from `gst-provider.types.ts`. All 7 methods are required because the e-Way Bill module uses the same provider. `sandbox-gst.provider.ts` is a complete working example to copy from, including token caching and one retry on a stale session.

```typescript
import { env } from "../../config/env.js";
import { toDocDate } from "./gst-codes.util.js";
import { GstProviderError, type GenerateEinvoiceInput, type EinvoiceResult, type GstProvider } from "./gst-provider.types.js";

// Adapter for <vendor>. The input is already validated, so only map fields and call the API.
class VendorGstProvider implements GstProvider {
  readonly name = "VENDOR";

  async generateEinvoice(input: GenerateEinvoiceInput): Promise<EinvoiceResult> {
    // 1. Build the IRP INV-01 (schema 1.1) payload from the input
    const payload = {
      Version: "1.1",
      TranDtls: { TaxSch: "GST", SupTyp: input.supplyType, RegRev: "N" },
      DocDtls: { Typ: input.documentType, No: input.invoiceNumber, Dt: toDocDate(input.invoiceDate) },
      SellerDtls: { Gstin: input.sellerGstin, LglNm: input.sellerLegalName, Addr1: input.sellerAddress.slice(0, 100), Loc: input.sellerPlace, Pin: Number(input.sellerPincode), Stcd: input.sellerStateCode },
      BuyerDtls: { Gstin: input.buyerGstin, LglNm: input.buyerLegalName, Pos: input.placeOfSupply, Addr1: input.buyerAddress.slice(0, 100), Loc: input.buyerPlace, Pin: Number(input.buyerPincode), Stcd: input.buyerStateCode },
      ItemList: input.lineItems.map((i) => ({
        SlNo: String(i.itemNumber), PrdDesc: i.productName, IsServc: i.isService ? "Y" : "N", HsnCd: i.hsnCode,
        Qty: i.quantity, Unit: i.unit, UnitPrice: i.unitPrice, TotAmt: i.grossAmount, Discount: i.discount,
        AssAmt: i.taxableAmount, GstRt: i.gstRate, CgstAmt: i.cgstAmount, SgstAmt: i.sgstAmount, IgstAmt: i.igstAmount, TotItemVal: i.totalItemAmount,
      })),
      ValDtls: { AssVal: input.totalTaxableAmount, CgstVal: input.totalCgst, SgstVal: input.totalSgst, IgstVal: input.totalIgst, Discount: input.totalDiscount, RndOffAmt: input.roundOffAmount, TotInvVal: input.totalInvoiceValue },
    };

    // 2. Call the vendor
    const res = await fetch(`${env.GSP_API_URL}/einvoice`, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) });
    const data = await res.json();

    // 3. A rejection must be a GstProviderError. The service stores it as a Failed row and a log entry
    if (!res.ok || !data.Irn) throw new GstProviderError(data.message ?? "IRP rejected the invoice", data.errorCode ?? "IRP_ERROR", data);

    // 4. Return the normalised result
    return { irn: data.Irn, ackNumber: String(data.AckNo), ackDate: new Date(data.AckDt), signedQrCode: data.SignedQRCode, signedInvoice: data.SignedInvoice, status: "Generated", raw: data };
  }

  // cancelEinvoice, getEinvoice, generateEwayBill, cancelEwayBill, updateVehicle, getEwayBill: same shape.
}

export const vendorGstProvider = new VendorGstProvider();
```

---

### Step 3: Wire the Provider in the Factory (only for a new adapter)
Edit [`src/shared/gst/gst-provider.factory.ts`](file:///D:/Livik-projects/Rubine-glass/rubine-toughened-glass-backend/src/shared/gst/gst-provider.factory.ts) and add the new value to the `GST_PROVIDER` enum in [`src/config/env.ts`](file:///D:/Livik-projects/Rubine-glass/rubine-toughened-glass-backend/src/config/env.ts).

```typescript
import { vendorGstProvider } from "./vendor-gst.provider.js";

export const getGstProvider = (): GstProvider => {
  switch (env.GST_PROVIDER) {
    case "VENDOR":
      return vendorGstProvider; // the new adapter
    case "SANDBOX":
    case "DIRECT":
      return sandboxGstProvider;
    case "MOCK":
    default:
      return mockGstProvider;
  }
};
```

---

## 4. What Already Works (Zero Changes Needed)

When the environment is set (or a new adapter is added), the following work automatically without modification:

1. **Database Tables:** `invoice.gst_einvoice`, `invoice.gst_ewaybill` and `invoice.gst_api_log` persist the IRN, Ack No, Ack date, signed QR, status and every provider call.
2. **Pre-flight Validation:** `gst-invoice-snapshot.ts` blocks bad data before the provider is contacted, with a message that names the master data to fix. The dry run `GET /final-invoices/:id/e-invoice/preview` returns the same result without storing anything.
3. **Duplicate Prevention:** The service checks first, and a partial unique index guarantees one live IRN per invoice.
4. **Audit Logging:** Every call, success or failure, is saved with timing. Passwords, tokens, API keys and authorization headers are masked before saving (`sanitizeForLog` in `gst-api-log.repository.ts`).
5. **Failure handling:** A failed attempt is saved as a `Failed` row with its error and can be retried. It never blocks the invoice.
6. **Invoice mirror:** The live IRN, Ack No and Ack date are copied onto `final_invoice.einvoice_*` in the same transaction, and cleared on cancel.
7. **Rights:** The routes check `e_invoice_view`, `e_invoice_add` and `e_invoice_status_update`. Seed `019_gst_compliance_rbac.sql` grants them to ACCOUNTS and SALES_MANAGER. ADMIN and SUPER_ADMIN bypass.
8. **Frontend:** The tax invoice sheet with IRN and QR (preview, print, PDF), the automatic generate on invoice creation, and the generate and cancel dialogs work against either provider.

---

## 5. How it is wired into the app

### 5.1 The behaviour that was decided
1. **e-Invoice is automatic.** When a user creates a Final Invoice and it is saved, the e-Invoice is generated straight away. The invoice is already saved by then, so a failure never undoes or blocks it.
2. **The Final Invoice preview is the tax invoice layout.** Everywhere the Final Invoice preview opens (Final Invoices table, Dispatch page, Gate Pass "Download Invoice"), it shows the new sheet in the layout of the sample, with the IRN, acknowledgement and QR. Print and Download PDF produce the same sheet.
3. **e-Way Bill is generated from the Dispatch actions menu** (kebab), shown on the Gate Pass, and can be downloaded on its own.

### 5.2 Where each piece lives

| Behaviour | File | What it does |
|---|---|---|
| Auto-generate on create | `final-invoices/components/create-final-invoice-panel.tsx` | In `onSuccess` of the create mutation, calls `useAutoGenerateEinvoice()`. The helper is `e-invoice/auto-generate.ts`. It never throws. On success a toast says "e-Invoice generated". On failure a toast says the invoice was saved and what to fix |
| Preview shows the tax invoice | `final-invoices/components/final-invoice-preview-modal.tsx` | Rewritten. Loads `GET /e-invoice/preview`. If ready, renders `EInvoiceTaxInvoiceSheet`. If not ready, falls back to the earlier `ProformaInvoiceSheet` and shows a yellow note with the reason, so printing is never blocked |
| Retry from the preview | same file | When there is no live IRN, a strip under the header shows the state (not generated, failed with the reason, cancelled) and a **Generate e-Invoice** button for roles with `e_invoice_add` |
| Copy label | same file | A selector in the header: Original for Recipient, Duplicate for Transporter, Triplicate for Assessee |
| Dispatch menu items | `dispatch/components/dispatch-table.tsx` | Kebab shows **Generate e-Way Bill** (disabled until the dispatch is approved), and **Download e-Way Bill** once one exists |
| Which rows have a bill | `dispatch/dispatch-page.tsx` | One call `GET /e-way-bills?finalInvoiceIds=...` for the rows on screen, kept apart from the Dispatch list query |
| Generate dialog | `e-way-bill/components/generate-ewaybill-dialog.tsx` | Opens for the clicked row, defaults the vehicle to the Dispatch vehicle, stays open with the message if the portal rejects it |
| Download separately | `e-way-bill/components/ewaybill-slip-modal.tsx` | Preview, Print and Download PDF of the slip with a barcode |
| Gate Pass | `dispatch/components/gate-pass-sheet.tsx`, `gate-pass-modal.tsx` | Prints "e-Way Bill No" and its valid-upto date next to the driver, read live, so a bill generated after the Gate Pass was created still shows |

Buttons are hidden for a role without the right (`useAuth().canModule`), and the backend enforces the same rights. A role without any e-Way Bill right sees no e-Way Bill items, and the Dispatch page skips the e-Way Bill call.

### 5.3 What the user sees if the data is not ready
The first save of a Final Invoice for a party with no GSTIN, or a product with no HSN, does **not** fail. The invoice is saved, a toast explains the problem, and the preview shows the earlier layout with the same reason in a note. After fixing the party or product, open the preview and click **Generate e-Invoice**.

### 5.4 Still open
| # | Decision | Notes |
|---|---|---|
| 1 | How is HSN entered? | No screen yet. Add an HSN field to the Product form. Until then it is set with SQL (5.5) |
| 2 | What happens to the IRN when a Final Invoice is declined or cancelled? | Today nothing. The IRN stays live at the government. See Section 8, gap 1 |
| 3 | Payment conditions, Policy No, Project Name, P.O. No, LR No, Case Nos | The sample prints them. Rubine does not store them, so they print blank |
| 4 | Credit notes, debit notes, exports | Not built |
| 5 | Should e-Invoice be skipped for a party with no GSTIN, silently? | Today a toast always appears. A B2C sale cannot have an e-Invoice, so this toast will be noise for retail customers |

`GstCompliancePanel` (`e-invoice/components/gst-compliance-panel.tsx`) is a ready made card with every action in one place. It is **not used** by the screens above. It is kept as an option for a future Final Invoice detail page.

### 5.5 One-time setup before the first real test
1. Run the migration and seed: `npm run migrate` then `npm run seed` in the backend. Migration `063_gst_einvoice_eway_bill.sql` adds the three tables and `master.product.hsn_code`. Seed `019_gst_compliance_rbac.sql` adds the `e_invoice` and `e_way_bill` modules and rights. Both are re-runnable. **Do this before opening the new build.** Without the tables, the preview falls back to the earlier layout and the e-Way Bill menu call fails quietly.
2. Check the company data on `/admin/company`: GSTIN, address line, city, 6-digit pincode.
3. Check one party has a GSTIN, city and 6-digit pincode on its billing address.
4. Check every product on the invoice has an HSN. List the ones that do not, and set them:
```sql
SELECT code, name, hsn_code FROM master.product WHERE hsn_code IS NULL;
UPDATE master.product SET hsn_code = '70071900' WHERE code = '<PRODUCT_CODE>';
```
5. Keep `GST_PROVIDER=MOCK` for the first run. Switch to `SANDBOX` (Section 3, Step 1) only when the flow is accepted.

### 5.6 How to test the flow on MOCK
1. Create a Final Invoice for a party that has a GSTIN, from a Work Order whose products have an HSN. Expect the toast "Final invoice created" followed by "e-Invoice generated".
2. Open the invoice preview. Expect the tax invoice layout, a green strip with the IRN start, a QR, and a red "SIMULATED (MOCK)" note under it. Print and Download PDF work.
3. Create one for a party with no GSTIN. Expect the invoice saved, an error toast naming the party, and the preview in the earlier layout with a yellow note. Add the GSTIN, reopen the preview, click **Generate e-Invoice**.
4. Call `POST /api/v1/final-invoices/<id>/e-invoice` again on the first invoice. Expect 409 `DUPLICATE_ACTIVE_EINVOICE`.
5. `POST .../e-invoice/cancel` with `{ "cancelReason": "2" }`. Expect status `Cancelled`, and the preview strip says it was cancelled.

---

## 6. API Reference

Base path `/api/v1`. All routes need a signed-in user. `:id` is the `invoice.final_invoice.id` UUID. Responses use the standard `{ success, message, data }` envelope with snake_case row fields. A provider failure returns 502 with the provider's error code and message.

| Method and path | Right | Purpose |
|---|---|---|
| `GET /final-invoices/:id/e-invoice/preview` | `e_invoice_view` | Dry run. `{ ready: true, snapshot }` or `{ ready: false, code, message }`. Stores nothing, calls no provider |
| `POST /final-invoices/:id/e-invoice` | `e_invoice_add` | Generate the IRN. 201 with the e-Invoice row |
| `GET /final-invoices/:id/e-invoice` | `e_invoice_view` | Latest attempt (Generated, Cancelled or Failed), or null |
| `POST /final-invoices/:id/e-invoice/cancel` | `e_invoice_status_update` | Body `{ cancelReason: "1".."4", cancelRemarks? }` |
| `GET /final-invoices/:id/e-invoice/logs` | `e_invoice_view` | Provider call audit log, newest first, last 100 |

---

## 7. File Map

**Backend** (`rubine-toughened-glass-backend/src`)

| File | Purpose |
|---|---|
| `database/migrations/063_gst_einvoice_eway_bill.sql` | Tables, indexes, `master.product.hsn_code` (new) |
| `database/seeds/019_gst_compliance_rbac.sql` | Modules and rights (new) |
| `config/env.ts`, `../.env.example` | `GST_PROVIDER` and credential keys (edited, additive) |
| `api/route-mapper.ts` | Mounts the two routers under `/final-invoices/:finalInvoiceId/...` (edited, 4 lines) |
| `shared/gst/gst-provider.types.ts` | `GstProvider` interface, input and result types, `GstProviderError` (new) |
| `shared/gst/gst-provider.factory.ts` | The only place a provider is chosen (new) |
| `shared/gst/mock-gst.provider.ts` | Offline simulation (new) |
| `shared/gst/sandbox-gst.provider.ts` | Sandbox.co.in adapter (new) |
| `shared/gst/gst-invoice-source.repository.ts` | Reads everything about one Final Invoice (new) |
| `shared/gst/gst-invoice-snapshot.ts` | Pure validation and GST calculation. Shared by e-Invoice and e-Way Bill (new) |
| `shared/gst/gst-api-log.repository.ts` | Audit log with credential masking (new) |
| `shared/gst/gst-codes.util.ts` | GSTIN, pincode, HSN, state code helpers (new) |
| `modules/e-invoice/*` | routes, controller, service, repository, schema, types, payload mapper (new) |
| `modules/final-invoice/*`, `modules/dispatch/*`, `modules/master/*` | no real change. Only read from |
| `modules/e-way-bill/*` | The e-Way Bill module also uses `shared/gst`. See its own guide |

**Frontend** (`rubine-toughened-glass-frontend/src/features`)

| File | Purpose |
|---|---|
| `e-invoice/index.ts` | Public exports. Import from `@/features/e-invoice` |
| `e-invoice/types.ts`, `api.ts`, `queries.ts` | Types, API calls, TanStack Query hooks |
| `e-invoice/utils.ts` | Amount in words, state names, date formats |
| `e-invoice/components/einvoice-sheet-data.ts` | Builds the print data from the validated snapshot and the IRN |
| `e-invoice/components/einvoice-tax-invoice-sheet.tsx` | The tax invoice layout with IRN and QR, plus `EINVOICE_SHEET_CSS` |
| `e-invoice/components/einvoice-preview-modal.tsx` | Preview, Print, Download PDF |
| `e-invoice/components/cancel-einvoice-modal.tsx` | Cancel with reason and remarks |
| `e-invoice/components/gst-audit-log-modal.tsx` | Provider call log |
| `e-invoice/auto-generate.ts` | `useAutoGenerateEinvoice()`, called after a Final Invoice is created (new) |
| `e-invoice/components/gst-compliance-panel.tsx` | An optional card with every action, not used by the current screens |
| `final-invoices/components/final-invoice-preview-modal.tsx` | **Edited.** Renders the tax invoice sheet, falls back to the earlier sheet |
| `final-invoices/components/create-final-invoice-panel.tsx` | **Edited.** 3 lines to auto-generate after saving |
| `dispatch/components/dispatch-table.tsx`, `dispatch-page.tsx`, `gate-pass-sheet.tsx`, `gate-pass-modal.tsx` | **Edited.** e-Way Bill menu items, dialogs, and the number on the Gate Pass |
| `final-invoices/final-invoice-as-proforma-data.ts`, `pi-status/*`, `lib/pdf/*` | no real change. The earlier sheet is still used as the fallback |

---

## 8. Known Gaps and Things to Know

1. **A declined or cancelled Final Invoice keeps its IRN.** The e-Invoice is generated while the invoice is still DRAFT, before the Party and Admin approvals. If the invoice is then declined or cancelled, the IRN stays live at the government. The user must cancel the e-Invoice by hand (the government allows it only within 24 hours) or a credit note is needed. Nothing cancels it automatically yet (open decision 2).
2. **Auto-generate runs in the browser,** right after the create call returns. If the user closes the tab in those milliseconds, no e-Invoice is made and the preview shows "not generated yet" with the Generate button. A server side hook inside `createFinalInvoice` would remove this gap, but it changes the Final Invoice service, so it was not done.
3. **Company PAN and GSTIN in the repo are mistyped.** Migration `049_company_invoice_fields.sql` and several frontend fallbacks hold PAN `AAAFCR6930C` and GSTIN `33AAAFCR6930C1ZO` (an extra `A`). The sample invoice shows PAN `AAFCR6930C` and GSTIN `33AAFCR6930C1ZO`. The GSTIN check rejects the mistyped one on purpose, with the message "Your company GSTIN is not a valid 15 character GSTIN". Check the value on `/admin/company`. The old migration was not edited because migrations are append-only.
4. **HSN for non-toughened products** is empty until someone enters it. Annealed Glass is not 7007 19 00.
5. **Charge lines use the HSN of the first product.** They are treated as part of the same supply, as on the sample. A different rule needs a change in `gst-invoice-snapshot.ts`.
6. **The GST rate is inferred from the total.** If tax rules for a product differ (mixed rates on one invoice), this cannot represent them. Mixed rates need GST stored per line on the Final Invoice first.
7. **B2C, exports, credit and debit notes** are not supported. The document type is always `INV` and the supply type always `B2B`.
8. **The sample's Policy No, Project Name, Party P.O. No, Party W/o No, L.C. No, Delivery Type, LR No, Case Nos and Payment Conditions** are printed as empty fields, because Rubine does not store them. See decision 6.
9. **"Page 1 of 1" is fixed text.** A very long invoice runs onto a second PDF page, but the label does not count pages.
10. **A MOCK QR cannot be verified** on any government app. It is plain text and is marked in red on the print.
11. **Not checked against a real database or the Sandbox service.** See Section 9.

---

## 9. What Was Checked

* Backend and frontend type checks pass (`npx tsc --noEmit`), and ESLint passes on the new frontend folders.
* A script ran the validation and GST calculation on the figures from the sample invoice (items 13,431.00, logo 40.00, polish 1,110.00, documentation 72.90, energy 1,998.00, handling 499.56). It produced assessable value 17,151.46, SGST 1,543.63, CGST 1,543.63, round off 0.28 and total 20,239.00, the same as the sample. It also produced a 64-character IRN from the MOCK provider and the expected plain error for a draft invoice, a missing buyer GSTIN, a missing HSN, a total that matches no GST slab, and an invalid invoice number. An inter-state buyer passed.
* The tax invoice sheet was rendered to a screenshot with the same data and compared with the sample image. The layout, summary rows and percentages match.
* Backend and frontend type checks pass again after the integration, and ESLint is clean on the changed files except two `set-state-in-effect` errors that were already in `gate-pass-modal.tsx` and `approve-dispatch-modal.tsx`.
* **Not checked:** the migration and seed were not run against a database, the endpoints were not called, the Final Invoice preview, the Dispatch menu and the Gate Pass were not opened in the app, and the Sandbox provider was not called. The auto-generate, the preview switch and the menu items are verified by type check only.
