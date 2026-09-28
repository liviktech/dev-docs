# E-Way Bill Developer Guide: Required Fields & Live Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** A concise, minimal reference detailing the required fields, their originating modules, and the exact steps to replace the current Mock e-Way Bill provider with a real Government NIC / GSP provider.

---

## 1. Required Fields & Originating Modules

To generate a government-compliant e-Way Bill (`EWB-01`), the ERP aggregates data across four primary modules: **Company Profile (Consignor)**, **Customer (Consignee)**, **Transport & Vehicle Details (Part B)**, and **Invoice Line Items (Part A)**.

### 1.1 Seller / Company Details (Consignor)
*Captured from Company Profile and snapshotted onto the Invoice record.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Consignor Legal Name** | Yes | Admin Profile / Settings | `users.companyName` | Legal business name registered on GST portal |
| **Consignor GSTIN** | Yes | Admin Profile / Settings | `users.gstNumber` | 15-character statutory GSTIN (e.g. `29AAPFU0939F1ZV`) |
| **Dispatch Address** | Yes | Admin Profile / Settings | `users.companyAddress` | Dispatch physical origin location |
| **State Code** | Yes | Auto-derived from GSTIN | Derived (first 2 digits) | 2-digit state code (e.g., `29` for Karnataka) |
| **Origin PIN Code** | Yes | Admin Profile / Settings | Extracted from address | 6-digit postal code used for distance lookup |

---

### 1.2 Customer Details (Consignee)
*Captured from Customer Master or inline Invoice Customer selection.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Consignee Name** | Yes | Customers Module / Form | `customers.name` / `customers.companyName` | Buyer / Recipient legal name |
| **Consignee GSTIN** | Yes (B2B) | Customers Module / Form | `customers.gstin` | 15-character GSTIN (or `URP` for unregistered) |
| **Delivery Address** | Yes | Customers Module / Form | `customers.address1`, `customers.city` | Final destination delivery location |
| **State Code** | Yes | Auto-derived from GSTIN / State | Derived (first 2 digits) | 2-digit POS code (e.g., `27` for Maharashtra) |
| **Destination PIN Code** | Yes | Customers Module / Form | `customers.pinCode` | 6-digit destination postal code |

---

### 1.3 Transport & Vehicle Details (Part B)
*Captured from `GenerateEwayBillModal` or `UpdateVehicleModal`.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Transport Mode** | Yes | e-Way Bill Form Modal | `gst_ewaybill.transportMode` | `Road`, `Rail`, `Air`, or `Ship` |
| **Vehicle Number** | Conditional | e-Way Bill Form Modal | `gst_ewaybill.vehicleNumber` | Mandatory for Road unless Transporter ID is provided |
| **Transporter ID** | Conditional | e-Way Bill Form Modal | `gst_ewaybill.transporterId` | 15-character GSTIN/Transporter ID (optional if Vehicle Number is present) |
| **Transport Doc No** | Optional | e-Way Bill Form Modal | `gst_ewaybill.transportDocNumber` | LR / Bilty (Road), RR (Rail), AWB (Air), BOL (Ship) |
| **Approx Distance** | Yes | e-Way Bill Form Modal | `gst_ewaybill.distance` | Driving distance in km (min `1`, max `4000`) |

---

### 1.4 Invoice Line Items & Summary (Part A)
*Captured from Invoice Form header and line items table.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Invoice Number** | Yes | Invoice Form Header | `invoices.invoiceNumber` | Max 16 alphanumeric chars |
| **Invoice Date** | Yes | Invoice Form Header | `invoices.invoiceDate` | Format `YYYY-MM-DD` |
| **Product HSN Code** | Yes | Line Items Table | `invoice_line_items.hsnCode` | 4, 6, or 8 digit HSN code |
| **Taxable Amount** | Yes | Auto-calculated | `invoice_line_items.taxableAmount` | Net taxable value before tax |
| **CGST Amount** | Yes (Intra) | Auto-calculated | `invoice_line_items.cgstAmount` | Only when Seller State == Buyer State |
| **SGST Amount** | Yes (Intra) | Auto-calculated | `invoice_line_items.sgstAmount` | Only when Seller State == Buyer State |
| **IGST Amount** | Yes (Inter) | Auto-calculated | `invoice_line_items.igstAmount` | Only when Seller State != Buyer State |
| **Total Invoice Value** | Yes | Auto-calculated | `invoices.totalAmount` | Gross value including taxes |

---

## 2. Eligibility & Statutory Rules Checklist

Before generating an e-Way Bill, the system verifies:
1. **Invoice Value Threshold:** Total invoice value exceeds statutory threshold (default ₹50,000).
2. **Document Status:** Cannot be `Draft` or `Void`.
3. **Part B Requirement:** 
   - **Road Transport:** Either `vehicleNumber` OR `transporterId` must be provided.
   - **Rail / Air / Ship:** Either `vehicleNumber` (receipt ref) OR `transportDocNumber` must be provided.
4. **Vehicle Format Validation:**
   - Standard Indian Vehicle: Regex `/^[A-Z]{2}[0-9]{1,2}[A-Z]{0,3}[0-9]{1,4}$/i` (e.g. `KA01AB1234`).
   - Temporary/Defense: Alphanumeric string between 4 to 15 characters.
5. **Transporter ID Validation:** Must be a valid 15-character GSTIN format (`/^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$/i`).
6. **No Duplicate Active EWB:** If the invoice already has an active status (`Generated`) in `gst_ewaybill`, duplicate generation is blocked.

---

## 3. Switching from Mock to Real Government / GSP Implementation

The ERP e-Way Bill architecture uses the **Provider Pattern**. The database access layer, validation rules, UI modal wrappers, dynamic icon renderers, and logging are **already fully built and decoupled**.

To switch to a live government NIC / GSP provider, **only 3 areas need modification**:

```text
┌────────────────────────────────────────────────────────┐
│               ERP Invoicing / UI Layer                 │
│         (NO CHANGES NEEDED - 100% COMPLETE)            │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│             gstEwayBillService.ts (Backend)            │
│         (NO CHANGES NEEDED - 100% COMPLETE)            │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│     1. gstProviderFactory.ts (EDIT: Switch Provider)   │
└──────────────┬──────────────────────────┬──────────────┘
               │                          │
        (GST_PROVIDER=MOCK)        (GST_PROVIDER=LIVE/GSP)
               │                          │
               ▼                          ▼
      ┌─────────────────┐       ┌──────────────────────────┐
      │ mockGstProvider │       │ 2. realGspEwayProvider.ts│
      │   (Simulated)   │       │   (EDIT: Add API Calls)  │
      └─────────────────┘       └───────────┬──────────────┘
                                            │
                                            ▼
                                ┌──────────────────────────┐
                                │ 3. .env (EDIT: Keys)     │
                                └──────────────────────────┘
```

---

### Step 1: Configure Environment Variables
Edit [`ERP-tool-backend/.env`](file:///D:/Livik-projects/ERP/ERP-tool-backend/.env):

```env
# Set Provider mode to LIVE or GSP
GST_PROVIDER=DIRECT
GST_ENVIRONMENT=PRODUCTION

# Official GSP / NIC Endpoints & Credentials
GSP_EWAYBILL_API_URL=https://api.gsp-provider.com/ewaybill/v1.1
GSP_CLIENT_ID=your_gsp_client_id
GSP_CLIENT_SECRET=your_gsp_client_secret
NIC_EWAYBILL_USERNAME=your_nic_username
NIC_EWAYBILL_PASSWORD=your_nic_password
```

---

### Step 2: Implement the Real Provider
Create or edit: `ERP-tool-backend/src/services/gst/realGspEwayProvider.ts`.

Implement the e-Way Bill methods of the `GstProvider` interface:

```typescript
import type {
  GstProvider,
  GenerateEwayBillInput,
  EwayBillResult,
  CancelEwayBillProviderInput,
  CancelEwayBillResult,
  UpdateVehicleProviderInput,
  UpdateVehicleResult,
} from './gstProvider.js';
import { GstProviderError } from './gstProvider.js';
import { env } from '../../config/env.js';

export class RealGspEwayProvider implements GstProvider {
  readonly name = 'RealGspEwayProvider';

  /**
   * 1. Generates 12-Digit e-Way Bill with NIC/GSP
   */
  async generateEwayBill(input: GenerateEwayBillInput): Promise<EwayBillResult> {
    const payload = {
      supplyType: 'O', // Outward
      subSupplyType: '1', // Supply
      docType: input.documentType || 'INV',
      docNo: input.invoiceNumber,
      docDate: input.invoiceDate,
      fromGstin: input.sellerGstin,
      fromTrdName: input.sellerLegalName,
      fromAddr1: input.sellerAddress,
      fromPincode: Number(input.sellerPincode),
      fromStateCode: Number(input.sellerStateCode),
      toGstin: input.buyerGstin,
      toTrdName: input.buyerLegalName,
      toAddr1: input.buyerAddress,
      toPincode: Number(input.buyerPincode),
      toStateCode: Number(input.buyerStateCode),
      totalValue: input.totalTaxableAmount,
      cgstValue: input.totalCgst || 0,
      sgstValue: input.totalSgst || 0,
      igstValue: input.totalIgst || 0,
      totInvValue: input.totalInvoiceValue,
      transMode: input.transportMode === 'Road' ? '1' : input.transportMode === 'Rail' ? '2' : input.transportMode === 'Air' ? '3' : '4',
      transDistance: String(input.distance),
      transporterId: input.transporterId || '',
      transDocNo: input.transportDocNumber || '',
      vehicleNo: input.vehicleNumber || '',
      vehicleType: 'R', // Regular
      itemList: input.lineItems.map((item, idx) => ({
        itemNo: idx + 1,
        productName: item.productName,
        hsnCode: Number(item.hsnCode),
        quantity: item.quantity,
        qtyUnit: item.unit,
        taxableAmount: item.taxableAmount,
        cgstRate: item.cgstRate || 0,
        sgstRate: item.sgstRate || 0,
        igstRate: item.igstRate || 0,
      })),
    };

    const response = await fetch(`${env.GSP_EWAYBILL_API_URL}/generate`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client-id': env.GSP_CLIENT_ID || '',
        'client-secret': env.GSP_CLIENT_SECRET || '',
      },
      body: JSON.stringify(payload),
    });

    const data = await response.json();

    if (!response.ok || data.status === 'ERROR') {
      throw new GstProviderError(
        data.message || 'Failed to generate e-Way Bill from NIC',
        data.errorCode || 'NIC_EWB_GENERATE_ERROR',
        data,
      );
    }

    return {
      ewbNumber: String(data.ewayBillNo),
      ewbDate: new Date(data.ewayBillDate),
      validUpto: new Date(data.validUpto),
      status: 'Generated',
      raw: data,
    };
  }

  /**
   * 2. Updates Vehicle Number / Mode in Part B
   */
  async updateVehicle(input: UpdateVehicleProviderInput): Promise<UpdateVehicleResult> {
    const response = await fetch(`${env.GSP_EWAYBILL_API_URL}/updatevehicle`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client-id': env.GSP_CLIENT_ID || '',
        'client-secret': env.GSP_CLIENT_SECRET || '',
      },
      body: JSON.stringify({
        ewbNo: Number(input.ewbNumber),
        vehicleNo: input.vehicleNumber || '',
        transMode: input.transportMode === 'Road' ? '1' : input.transportMode === 'Rail' ? '2' : input.transportMode === 'Air' ? '3' : '4',
        transDocNo: input.transportDocNumber || '',
        reasonCode: '1', // Transporter Change / Breakdown
        reasonRemark: 'Updated via ERP',
      }),
    });

    const data = await response.json();
    if (!response.ok || data.status === 'ERROR') {
      throw new GstProviderError(data.message || 'Vehicle update failed', data.errorCode || 'NIC_VEHICLE_UPDATE_ERROR', data);
    }

    return {
      ewbNumber: String(data.ewayBillNo),
      validUpto: new Date(data.validUpto),
      updatedDate: new Date(data.vehUpdDate || Date.now()),
      raw: data,
    };
  }

  /**
   * 3. Cancels an e-Way Bill within 24 Hours
   */
  async cancelEwayBill(input: CancelEwayBillProviderInput): Promise<CancelEwayBillResult> {
    const response = await fetch(`${env.GSP_EWAYBILL_API_URL}/cancel`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client-id': env.GSP_CLIENT_ID || '',
        'client-secret': env.GSP_CLIENT_SECRET || '',
      },
      body: JSON.stringify({
        ewbNo: Number(input.ewbNumber),
        cancelRsnCode: 2, // Data entry mistake
        cancelRemark: input.cancelReason,
      }),
    });

    const data = await response.json();
    if (!response.ok || data.status === 'ERROR') {
      throw new GstProviderError(data.message || 'e-Way Bill cancellation failed', data.errorCode || 'NIC_CANCEL_ERROR', data);
    }

    return {
      ewbNumber: String(data.ewayBillNo),
      cancelledDate: new Date(data.cancelDate || Date.now()),
      raw: data,
    };
  }
}

export const realGspEwayProvider = new RealGspEwayProvider();
```

---

### Step 3: Wire the Real Provider in Factory
Edit [`ERP-tool-backend/src/services/gst/gstProviderFactory.ts`](file:///D:/Livik-projects/ERP/ERP-tool-backend/src/services/gst/gstProviderFactory.ts):

```typescript
import { env } from '../../config/env.js';
import type { GstProvider } from './gstProvider.js';
import { mockGstProvider } from './mockGstProvider.js';
import { sandboxProvider } from './sandboxProvider.js';
import { realGspEwayProvider } from './realGspEwayProvider.js';

export function getGstProvider(): GstProvider {
  switch (env.GST_PROVIDER) {
    case 'DIRECT':
      return realGspEwayProvider;
    case 'SANDBOX':
      return sandboxProvider;
    case 'MOCK':
    default:
      return mockGstProvider;
  }
}
```

---

## 4. What Already Works (Zero Changes Needed)

When you make the above 3 edits, the entire e-Way Bill pipeline works seamlessly without touching any of the following:

1. **Database Tables:** [`gst_ewaybill`](file:///D:/Livik-projects/ERP/ERP-tool-backend/src/database/gstEwayBill.ts) and `gst_api_logs` automatically persist the live e-Way Bill number, creation timestamp, validity expiry date, transport mode, vehicle number, transporter ID, document number, and audit logs.
2. **Pre-flight Validation:** [`ewayBillValidation.ts`](file:///D:/Livik-projects/ERP/ERP-tool-backend/src/validations/ewayBillValidation.ts) and [`ewayBillUtils.ts`](file:///D:/Livik-projects/ERP/ERP-tool-frontend/src/features/invoice/utils/ewayBillUtils.ts) validate transport modes, vehicle regex, and distances before making API calls.
3. **Audit Logging:** Every API payload, response, latency (ms), and failure traceback is logged in `gst_api_logs`.
4. **Vehicle Updates & Cancelation:** Dedicated backend endpoints (`POST /api/invoices/:id/ewaybill/update-vehicle` and `POST /api/invoices/:id/ewaybill/cancel`) manage the Part B lifecycle.
5. **Frontend UI Components:**
   - [`GenerateEwayBillModal.tsx`](file:///D:/Livik-projects/ERP/ERP-tool-frontend/src/features/invoice/GenerateEwayBillModal.tsx): Reusable modal for initializing Part A & Part B.
   - [`UpdateVehicleModal.tsx`](file:///D:/Livik-projects/ERP/ERP-tool-frontend/src/features/invoice/UpdateVehicleModal.tsx): Reusable modal for updating Part B vehicle/document info.
   - [`EwayBillTransportFields.tsx`](file:///D:/Livik-projects/ERP/ERP-tool-frontend/src/features/invoice/components/ewaybill/EwayBillTransportFields.tsx): Dynamic transport mode field renderer with mode-specific icons (`Truck`, `Train`, `Plane`, `Ship`), labels, and placeholders.
   - Dynamic mode badges on the Invoice Details page and Invoice View Slip automatically change transport icons based on active mode (`Road` $\rightarrow$ Truck, `Rail` $\rightarrow$ Train, `Air` $\rightarrow$ Plane, `Ship` $\rightarrow$ Ship).
