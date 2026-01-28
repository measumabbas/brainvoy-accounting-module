# Accounting Module (AR/AP) Technical Implementation Plan

## 1. Overview
This module automates Accounts Payable (AP) and Accounts Receivable (AR) for mid-market CPG brands. It integrates with existing agents (Procurement, Inventory) and synchronizes strictly with QuickBooks.
**Tech Stack**: NestJS (Backend), Sequelize (ORM), PostgreSQL (DB), React/Next.js (Frontend), BullMQ (Async Queues).

---

## 2. Database Schema (Sequelize Models)

### Shared Entities
*   **`Companies`** (Existing):
    *   **New Columns**: `qb_realm_id`, `qb_access_token`, `qb_refresh_token`, `qb_token_expires_at`, `qb_sync_status`.
    *   **Why**: These fields are required to maintain a persistent, authenticated OAuth2 connection with QuickBooks Online. This allows the system to synchronize financial data (invoices, payments) bidirectionally without requiring the user to re-login constantly.
*   **`AuditLogs`**: `id`, `user_id`, `company_id`, `entity_type`, `entity_id`, `action`, `changes` (JSON), `created_at`.

### Accounts Payable (AP)
#### 1. `Suppliers` (Modify Existing)
*   **New Columns**:
    *   `qb_vendor_id` (String, Indexed)
    *   `payment_terms` (String)
    *   `preferred_payment_method` (Enum: ACH, Wire, Check, Credit_Card)
    *   `bank_account_info` (Text, Encrypted)
*   **Why**:
    *   `qb_vendor_id`: Establish a strict 1:1 mapping with QuickBooks Vendors to prevent duplicate vendor creation during sync.
    *   `payment_terms`: Needed to automatically calculate `due_date` on new invoices (e.g., "Net 30").
    *   `preferred_payment_method` & `bank_account_info`: Required to automate the payment execution process (e.g., generating ACH files) and ensure the correct payment rail is selected for each vendor.

#### 2. `ApInvoices` (New)
*   **Columns**:
    *   `id` (PK), `company_id` (FK), `supplier_id` (FK), `po_id` (FK, Nullable)
    *   `invoice_number` (String), `qb_bill_id` (String)
    *   `invoice_date` (Date), `due_date` (Date), `received_date` (Date)
    *   `subtotal`, `tax_amount`, `shipping_amount`, `total_amount`, `valid_amount_paid`, `balance` (Decimals)
    *   `status` (Enum: processing, matched, exception, approved, scheduled, paid)
    *   `exception_type` (Enum, Nullable), `exception_notes` (Text)
    *   `document_url` (String), `ocr_data` (JSONB)
*   **Indexes**: `company_id`, `supplier_id`, `status`, `due_date`.

#### 3. `ApInvoiceItems` (New)
*   **Columns**:
    *   `id` (PK), `invoice_id` (FK), `po_line_id` (FK, Nullable)
    *   `description`, `sku`, `quantity`, `unit_price`, `total_price`
    *   `qb_expense_account_id` (String)

#### 4. `ApPayments` (New)
*   **Columns**:
    *   `id` (PK), `company_id` (FK), `invoice_id` (FK), `batch_id` (FK, Nullable)
    *   `qb_payment_id` (String)
    *   `payment_date` (Date), `amount` (Decimal), `payment_method` (Enum), `reference_number` (String)

#### 5. `ApPaymentBatches` (New)
*   **Columns**:
    *   `id` (PK), `company_id` (FK)
    *   `batch_number` (String), `total_amount` (Decimal), `payment_count` (Int)
    *   `status` (Enum: draft, approved, processed)

### Accounts Receivable (AR)
#### 1. `Customers` (New)
*   **Columns**:
    *   `id` (PK), `company_id` (FK), `name`, `email`, `phone`
    *   `qb_customer_id` (String)
    *   `billing_address` (JSONB), `shipping_address` (JSONB)
    *   `credit_limit` (Decimal), `credit_hold` (Boolean)

#### 2. `ArInvoices` (New)
*   **Columns**:
    *   `id` (PK), `company_id` (FK), `customer_id` (FK), `sales_order_id` (FK, Nullable)
    *   `invoice_number` (String, Generated), `qb_invoice_id` (String)
    *   `invoice_date` (Date), `due_date` (Date)
    *   `total_amount`, `amount_paid`, `balance` (Decimals)
    *   `status` (Enum: draft, sent, partial, paid, overdue, void)
    *   `document_url` (String)

#### 3. `ArInvoiceItems` (New)
*   **Columns**:
    *   `id` (PK), `ar_invoice_id` (FK)
    *   `description`, `sku`, `quantity`, `unit_price`, `total_price`
    *   `qb_revenue_account_id` (String)

#### 4. `ArPayments` (New)
*   **Columns**:
    *   `id` (PK), `company_id` (FK), `customer_id` (FK)
    *   `qb_payment_id` (String)
    *   `payment_date` (Date), `amount` (Decimal), `method` (Enum), `reference` (String)

#### 5. `ArPaymentApplications` (New - Junction)
*   **Columns**:
    *   `id` (PK), `payment_id` (FK), `invoice_id` (FK)
    *   `amount_applied` (Decimal)

---

## 3. API Endpoints (NestJS)

### AP Module (`/ap`)
*   `POST /ap/invoices/upload` - Upload PDF, trigger OCR (Async).
*   `GET /ap/invoices` - List invoices with filters (status, supplier, date).
*   `GET /ap/invoices/:id` - Get invoice details + line items + match status.
*   `PATCH /ap/invoices/:id/approve` - Approve invoice for payment.
*   `POST /ap/payments/batch` - Create payment batch for selected invoices.
*   `GET /ap/dashboard` - Stats: Total Payables, Overdue, Exceptions.

### AR Module (`/ar`)
*   `POST /ar/invoices` - Create customer invoice (manual or from SO).
*   `POST /ar/invoices/:id/send` - Email invoice to customer.
*   `POST /ar/payments` - Record received payment.
*   `POST /ar/payments/:id/apply` - Apply payment to invoices.
*   `GET /ar/dashboard` - Stats: Total Receivables, DSO, Aging Buckets.

### QuickBooks Integration (`/integrations/quickbooks`)
*   `GET /auth` - OAuth flow start.
*   `GET /callback` - Handle OAuth callback & save tokens.
*   `POST /sync/push` - Trigger manual push to QB.
*   `POST /sync/pull` - Trigger manual pull from QB.

---

## 4. UI/UX Design (Frontend)

### Common Layout
*   **Sidebar**: "Accounting" section with sub-items: Dashboard, Payables (Invoices, Payments), Receivables (Invoices, Customers), Reports.

### 1. Accounting Dashboard
*   **Split View**: Top half AP (Money Out), Bottom half AR (Money In).
*   **Widgets**:
    *   "Cash Flow Forecast": Line chart of AP due vs AR expected.
    *   "Action Required": List of Exception Invoices and Approval Requests.
    *   "Aging Summary": Bar chart (0-30, 31-60, 60+ days).

### 2. AP Invoice Processing (The "Workbench")
*   **Split Screen Interface**:
    *   **Left Pane**: PDF Viewer (Zoom/Pan).
    *   **Right Pane**: Form with extracted data.
    *   **Status Bar**: "Matching: 95% Match". Highlights variances in Red (e.g., Qty 100 vs 90 Received).
*   **Action Buttons**: "Approve", "Reject", "Hold", "Route to Manager".

### 3. Payment Run Flow
*   **Selection Step**: Table of approved invoices due soon. Checkboxes to select for payment.
*   **Review Step**: Summary of total amount, breakdown by method (ACH/Check).
*   **Confirmation**: "Execute Run" -> Generates Batch -> Updates Status -> Syncs to QB.

### 4. AR Invoice Builder
*   **Header**: Customer Selector (Auto-fill address), Dates.
*   **Lines**: Editable table. "Add Item" via SKU lookup.
*   **Footer**: Tax calculations, Notes.
*   **Preview Mode**: See exact PDF before sending.

### 5. Customer Portal (Optional/Future)
*   Secure link for customers to view open invoices and pay via credit card/ACH (Stripe integration).

---

## 5. Implementation Phases
1.  **Foundation**: DB Models, Migrations, Basic CRUD APIs.
2.  **AP Workflow**: OCR integration, 3-way matching logic, Invoice Approval.
3.  **AR Workflow**: Invoice generation, Email delivery, Payment recording.
4.  **QuickBooks Sync**: OAuth setup, Bi-directional sync jobs (BullMQ).
5.  **Intelligence**: Dashboards, Cash flow forecasting, Automated reminders.
