# E-Invoice Developer Guide: Required Fields & Live Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** A concise, minimal reference detailing the required fields, their originating modules, and the exact steps to replace the current Mock provider with a real Government IRP / GSP provider.

---

## 1. Required Fields & Originating Modules

To generate a government-compliant e-invoice (`INV-01`), the ERP aggregates data across four primary modules: **Company Profile (Seller)**, **Customer (Buyer)**, **Item Catalog (Line Items)**, and **Invoice Header**.

### 1.1 Seller / Company Details (Supplier)
*Captured from Company Profile and snapshotted onto the Invoice record.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Legal Name** | Yes | Bottom-Left Admin Profile Modal / Settings | `users.companyName` | Legal business name registered on GST portal |
| **GSTIN** | Yes | Bottom-Left Admin Profile Modal / Settings | `users.gstNumber` | 15-character statutory GSTIN (e.g. `29AAPFU0939F1ZV`) |
| **Address** | Yes | Bottom-Left Admin Profile Modal / Settings | `users.companyAddress` | Building, street, locality |
| **State Code** | Yes | Auto-derived from GSTIN | Derived (first 2 digits) | 2-digit state code (e.g., `29` for Karnataka) |
| **PIN Code** | Yes | Bottom-Left Admin Profile Modal / Settings | Extracted from address | 6-digit postal code |

---

### 1.2 Customer Details (Buyer)
*Captured from Customer Master or inline Invoice Customer selection.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Customer Name** | Yes | Customers Module / New Customer Modal | `customers.name` / `customers.companyName` | Buyer legal or trade name |
| **Customer GSTIN** | Yes (B2B) | Customers Module / New Customer Modal | `customers.gstin` | 15-character GSTIN. Mandatory for B2B e-invoices |
| **Billing Address** | Yes | Customers Module / Customer Form | `customers.address1`, `customers.city` | Dispatch/Billing physical location |
| **State** | Yes | Customers Module / Customer Form | `customers.state` / `customers.placeOfSupply` | Customer state name |
| **State Code** | Yes | Auto-derived from GSTIN / State | Derived (first 2 digits) | 2-digit POS code (e.g., `27` for Maharashtra) |
| **PIN Code** | Yes | Customers Module / Customer Form | `customers.pinCode` | 6-digit destination postal code |

---

### 1.3 Invoice Line Items
*Captured in Invoice Creation / Line Items Table.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Product Name** | Yes | Invoice Line Item / Items Catalog | `invoice_line_items.productName` | Description of goods or services |
| **HSN Code** | Yes | Invoice Line Item / Items Catalog | `invoice_line_items.hsnCode` | 4, 6, or 8 digit HSN/SAC code |
| **Quantity** | Yes | Invoice Line Items Table | `invoice_line_items.quantity` | Positive number (`> 0`) |
| **Unit (UoM)** | Yes | Invoice Line Items Table | `invoice_line_items.unit` | Standard GST unit code (`NOS`, `KGS`, `BOX`, `PCS`, etc.) |
| **Unit Price** | Yes | Invoice Line Items Table | `invoice_line_items.unitPrice` | Price per unit before tax |
| **Discount** | Optional | Invoice Line Items Table | `invoice_line_items.discount` | Line discount (default `0`) |
| **Taxable Amount** | Yes | Auto-calculated | `invoice_line_items.taxableAmount` | `(Quantity × UnitPrice) - Discount` |
| **CGST Rate & Amount** | Yes (Intra) | Auto-calculated | `invoice_line_items.cgstRate`, `cgstAmount` | Only when Seller State == Buyer State |
| **SGST Rate & Amount** | Yes (Intra) | Auto-calculated | `invoice_line_items.sgstRate`, `sgstAmount` | Only when Seller State == Buyer State |
| **IGST Rate & Amount** | Yes (Inter) | Auto-calculated | `invoice_line_items.igstRate`, `igstAmount` | Only when Seller State != Buyer State |
| **Total Line Amount** | Yes | Auto-calculated | `invoice_line_items.totalAmount` | `TaxableAmount + Tax` |

---

### 1.4 Invoice Header & Totals
*Captured from Invoice Form header and calculated summary.*

| Field | Required | Where to Fill in ERP | Database Column | Notes |
|---|---|---|---|---|
| **Invoice Number** | Yes | Invoice Form Header | `invoices.invoiceNumber` | Max 16 alphanumeric chars with `/` or `-` |
| **Invoice Date** | Yes | Invoice Form Header | `invoices.invoiceDate` | Format `YYYY-MM-DD` |
| **Document Type** | Yes | Invoice Form | `invoices.invoiceType` | `INV` (Tax Invoice), `CRN` (Credit Note), `DBN` (Debit Note) |
| **Supply Type** | Yes | Auto-classified | Derived (`B2B`, `SEZWP`, `EXPWP`) | Default `B2B` when buyer GSTIN is present |
| **Total Taxable Value** | Yes | Auto-calculated | `invoices.subTotal` | Sum of all line item taxable values |
| **Total Tax** | Yes | Auto-calculated | `invoices.taxAmount` | Sum of CGST+SGST or IGST |
| **Total Invoice Value** | Yes | Auto-calculated | `invoices.totalAmount` | `Total Taxable Value + Total Tax ± Round Off` |

---

## 2. Eligibility & Statutory Rules Checklist

Before generating an e-invoice, the system verifies:
1. **Document Status:** Cannot be `Draft` or `Void`.
2. **B2B Requirement:** Both Seller GSTIN and Buyer GSTIN must be valid 15-character strings.
3. **State Consistency:** First 2 digits of GSTIN must match the party's 2-digit state code.
4. **Tax Split Validation:**
   - **Intra-State:** If Seller State == Buyer State $\rightarrow$ CGST and SGST must be $> 0$; IGST must be $0$.
   - **Inter-State:** If Seller State $\neq$ Buyer State $\rightarrow$ IGST must be $> 0$; CGST and SGST must be $0$.
5. **Line Item Integrity:** Every line item must have a valid HSN, positive quantity, valid UoM, and non-negative taxable value.
6. **No Duplicate Active IRN:** If the invoice already has an active IRN in `gst_einvoice`, duplicate generation is blocked.

---

## 3. Switching from Mock to Real Government / GSP Implementation

The ERP architecture uses the **Provider Pattern**. The database layer, validation rules, payload builders, logging, and frontend UI are **already fully built and decoupled**.

To switch to a live government/GSP provider, **only 3 areas need modification**:

```text
┌────────────────────────────────────────────────────────┐
│               ERP Invoicing / UI Layer                 │
│         (NO CHANGES NEEDED - 100% COMPLETE)            │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│            gstEinvoiceService.ts (Backend)             │
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
      ┌─────────────────┐       ┌────────────────────────┐
      │ mockGstProvider │       │ 2. realGspProvider.ts  │
      │   (Simulated)   │       │  (EDIT: Add API Calls) │
      └─────────────────┘       └───────────┬────────────┘
                                            │
                                            ▼
                                ┌────────────────────────┐
                                │ 3. .env (EDIT: Keys)   │
                                └────────────────────────┘
```

---

### Step 1: Configure Environment Variables
Edit [`ERP-tool-backend/.env`](file:///D:/Livik-projects/ERP/ERP-tool-backend/.env):

```env
# Set Provider mode to LIVE or GSP
GST_PROVIDER=DIRECT
GST_ENVIRONMENT=PRODUCTION

# Official GSP / IRP Endpoints & Credentials (e.g. ClearTax, Cygnet, Masters India, NIC)
GSP_API_URL=https://api.gsp-provider.com/einvoice/v1
GSP_CLIENT_ID=your_gsp_client_id
GSP_CLIENT_SECRET=your_gsp_client_secret
NIC_EINVOICE_USERNAME=your_irp_username
NIC_EINVOICE_PASSWORD=your_irp_password
```

---

### Step 2: Implement the Real Provider
Create or edit: [`ERP-tool-backend/src/services/gst/sandboxProvider.ts`](file:///D:/Livik-projects/ERP/ERP-tool-backend/src/services/gst/sandboxProvider.ts) (or create `realGspProvider.ts`).

Implement the standard `GstProvider` interface:

```typescript
import type {
  GstProvider,
  GenerateEinvoiceInput,
  EinvoiceResult,
  CancelEinvoiceProviderInput,
  CancelEinvoiceResult,
  EinvoiceDetailsResult,
} from './gstProvider.js';
import { GstProviderError } from './gstProvider.js';
import { env } from '../../config/env.js';

export class RealGspProvider implements GstProvider {
  readonly name = 'RealGspProvider';

  /**
   * 1. Generates IRN with the Government IRP
   */
  async generateEinvoice(input: GenerateEinvoiceInput): Promise<EinvoiceResult> {
    // A. Map input to Government INV-01 JSON Schema
    const payload = {
      Version: '1.1',
      TranDtls: {
        TaxSch: 'GST',
        SupTyp: input.supplyType,
        RegRev: 'N',
      },
      DocDtls: {
        Typ: input.documentType,
        No: input.invoiceNumber,
        Dt: input.invoiceDate, // DD/MM/YYYY
      },
      SellerDtls: {
        Gstin: input.sellerGstin,
        LglNm: input.sellerLegalName,
        Addr1: input.sellerAddress,
        Loc: input.sellerPlace,
        Pin: Number(input.sellerPincode),
        Stcd: input.sellerStateCode,
      },
      BuyerDtls: {
        Gstin: input.buyerGstin,
        LglNm: input.buyerLegalName,
        Pos: input.placeOfSupply || input.buyerStateCode,
        Addr1: input.buyerAddress,
        Loc: input.buyerPlace,
        Pin: Number(input.buyerPincode),
        Stcd: input.buyerStateCode,
      },
      ItemList: input.lineItems.map((item, idx) => ({
        SlNo: String(idx + 1),
        PrdDesc: item.productName,
        IsServc: 'N',
        HsnCd: item.hsnCode,
        Qty: item.quantity,
        Unit: item.unit,
        UnitPrice: item.unitPrice,
        TotAmt: item.grossAmount,
        Discount: item.discount || 0,
        AssAmt: item.taxableAmount,
        GstRt: (item.cgstRate || 0) + (item.sgstRate || 0) + (item.igstRate || 0),
        CgstAmt: item.cgstAmount || 0,
        SgstAmt: item.sgstAmount || 0,
        IgstAmt: item.igstAmount || 0,
        TotItemVal: item.totalItemAmount,
      })),
      ValDtls: {
        AssVal: input.totalTaxableAmount,
        CgstVal: input.totalCgst || 0,
        SgstVal: input.totalSgst || 0,
        IgstVal: input.totalIgst || 0,
        Discount: input.totalDiscount || 0,
        RndOffAmt: input.roundOffAmount || 0,
        TotInvVal: input.totalInvoiceValue,
      },
    };

    // B. Send HTTP request to GSP/IRP API
    const response = await fetch(`${env.GSP_API_URL}/generate`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client-id': env.GSP_CLIENT_ID || '',
        'client-secret': env.GSP_CLIENT_SECRET || '',
        'user_name': env.NIC_EINVOICE_USERNAME || '',
      },
      body: JSON.stringify(payload),
    });

    const data = await response.json();

    if (!response.ok || data.status === 'ERROR') {
      throw new GstProviderError(
        data.message || 'Failed to generate IRN from IRP',
        data.errorCode || 'IRP_GENERATE_ERROR',
        data,
      );
    }

    // C. Return the exact response fields required by ERP
    return {
      irn: data.Irn,
      ackNumber: String(data.AckNo),
      ackDate: new Date(data.AckDt),
      signedQrCode: data.SignedQRCode,
      signedInvoice: data.SignedInvoice,
      status: 'Generated',
      raw: data,
    };
  }

  /**
   * 2. Cancels an existing IRN on the IRP
   */
  async cancelEinvoice(input: CancelEinvoiceProviderInput): Promise<CancelEinvoiceResult> {
    const response = await fetch(`${env.GSP_API_URL}/cancel`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'client-id': env.GSP_CLIENT_ID || '',
        'client-secret': env.GSP_CLIENT_SECRET || '',
      },
      body: JSON.stringify({
        Irn: input.irn,
        CnlRsn: input.cancelReason, // '1': Duplicate, '2': Data entry mistake, etc.
        CnlRem: input.cancelRemarks || '',
      }),
    });

    const data = await response.json();
    if (!response.ok || data.status === 'ERROR') {
      throw new GstProviderError(data.message || 'IRP cancellation failed', data.errorCode || 'IRP_CANCEL_ERROR', data);
    }

    return {
      irn: data.Irn,
      cancelledDate: new Date(data.CancelDate || Date.now()),
      raw: data,
    };
  }

  /**
   * 3. Fetches IRN details from IRP
   */
  async getEinvoice(irn: string, gstin?: string): Promise<EinvoiceDetailsResult> {
    const response = await fetch(`${env.GSP_API_URL}/irn/${irn}`, {
      method: 'GET',
      headers: {
        'client-id': env.GSP_CLIENT_ID || '',
        'client-secret': env.GSP_CLIENT_SECRET || '',
      },
    });

    const data = await response.json();
    return {
      irn: data.Irn,
      ackNumber: String(data.AckNo),
      ackDate: new Date(data.AckDt),
      signedQrCode: data.SignedQRCode,
      signedInvoice: data.SignedInvoice,
      status: data.Status === 'ACT' ? 'Generated' : 'Cancelled',
      raw: data,
    };
  }
}

export const realGspProvider = new RealGspProvider();
```

---

### Step 3: Wire the Real Provider in Factory
Edit [`ERP-tool-backend/src/services/gst/gstProviderFactory.ts`](file:///D:/Livik-projects/ERP/ERP-tool-backend/src/services/gst/gstProviderFactory.ts):

```typescript
import { env } from '../../config/env.js';
import type { GstProvider } from './gstProvider.js';
import { mockGstProvider } from './mockGstProvider.js';
import { sandboxProvider } from './sandboxProvider.js';
// 1. Import your real provider
import { realGspProvider } from './realGspProvider.js';

export function getGstProvider(): GstProvider {
  switch (env.GST_PROVIDER) {
    case 'DIRECT':
      return realGspProvider; // <-- Returns the real GSP provider
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

When you make the above 3 edits, the entire e-invoicing pipeline immediately works without touching any of the following:

1. **Database Tables:** `gst_einvoice` and `gst_api_logs` automatically persist the live IRN, Ack No, Ack Date, Signed QR code, and provider logs.
2. **Pre-flight Validation:** [`einvoiceValidation.ts`](file:///D:/Livik-projects/ERP/ERP-tool-backend/src/validations/einvoiceValidation.ts) automatically blocks invalid requests before contacting the provider.
3. **Audit Logging:** Every live API request, response, and error is automatically recorded in `gst_api_logs` with latency tracking and masked credentials.
4. **Duplicate Prevention:** The backend automatically blocks submitting the same invoice twice if an active IRN exists.
5. **Invoice Status Lifecycle:** The invoice `gstStatus` automatically updates from `Pending` $\rightarrow$ `EinvoiceGenerated` $\rightarrow$ `EinvoiceCancelled`.
6. **Frontend Display:**
   - The invoice details page displays the live IRN, Ack No, Ack Date, and Cancellation actions.
   - The print/template view ([`InvoiceQRCode.tsx`](file:///D:/Livik-projects/ERP/ERP-tool-frontend/src/features/invoice/template/components/preview/InvoiceQRCode.tsx)) automatically renders the live high-density signed QR Code on all invoice printouts.
