# Vendor Portal - Specification Document
> Specs & Approach Document — no implementation code

| Tài liệu | Mục đích |
|---|---|
| **README.md** (file này) | Business logic, quy trình nghiệp vụ, và các quyết định thiết kế cấp cao |
| [PROCESS_FLOW.md](PROCESS_FLOW.md) | Sơ đồ luồng chi tiết: RFQ → PO → DO → Giao hàng → Xác nhận, kèm swimlane và state machine |
| [roadmap.md](roadmap.md) | Kế hoạch triển khai theo phase: DB schema, API endpoints, cấu hình hạ tầng, checklist go-live |

---

## Project Summary

An independent bilingual (Vietnamese + English) web portal serving two types of users: **vendors** and **portal admins**. Vendors log in using their **Vendor ID** (`res.partner.id` from Odoo), confirm or reject Sent RFQs, edit Delivery Orders (quantities and delivery date), digitally sign and print DOs, and track delivery status through to store receipt confirmation. The portal also supports returns via Return Purchase Orders (RPO) and Goods Return Notes. Vendors can export data as PDF or CSV for invoicing and reconciliation. Portal admins share the same interface but have additional access to view all vendors, all POs and DOs across the system, trigger the Odoo sync manually, unlock signed DOs, and download any vendor's signed PDF. The portal runs on a separate VM from Odoo and integrates via Odoo's XML-RPC API using a dedicated service account. **Vendors only access the portal; stores only access Odoo.** All email is delivered via AWS SES.

---

## Confirmed Decisions

| Concern | Decision |
|---|---|
| Odoo version | 16 Community Edition |
| Odoo API protocol | XML-RPC (Python `xmlrpc.client`) |
| Vendor login identifier | `res.partner.id` (integer, assigned by Odoo — never changes) |
| Vendor password | Portal-owned, stored in portal PostgreSQL only |
| Profile data source | One-way sync from Odoo `res.partner` for profile fields (name, company, phone, tax ID). Email is copied from Odoo on initial account creation only (used to send the welcome email), then never overwritten by sync |
| Account provisioning | Auto from Odoo partners where `is_vendor = True` **and** `email` is set |
| Portal language | Vietnamese + English (bilingual, user-switchable) |
| Portal PO statuses | Waiting (`sent`), Confirmed (`purchase`), Cancelled (`cancel`) — Draft (`draft`) is not shown. Auto-cancel after 7 days past Expected Arrival if vendor has not confirmed or rejected |
| Portal DO statuses | Draft, Signed, Done, Cancelled |
| Odoo PO states (unchanged) | RFQ / RFQ Sent / Purchase Order / Cancelled — portal does not modify Odoo's base behaviour |
| DO per PO | Odoo auto-creates exactly 1 DO (stock.picking) when PO is confirmed — portal reads and displays it. Vendor cannot create additional DOs |
| DO delivery date | Single date for the entire DO (not per product line) |
| DO editing | Vendor edits delivery date + quantities (qty <= ordered qty from PO) |
| DO locking trigger | Vendor digital signature only — no 23:00 cutoff |
| DO signing | Digital signature on portal locks the DO, pushes delivery date + set quantities to Odoo Receipt |
| DO printing | After signing, vendor can print DO PDF (includes signature) multiple times |
| DO PDF language | Vietnamese only — all printed DO PDFs use Vietnamese labels |
| DO PDF content | Vietnamese PDF — PO barcode, vendor info, store ID, PO confirmation date, delivery date, product table (barcode, name, UoM, delivery qty, blank columns for store to fill) |
| UoM handling | Single UoM per product line, inherited from PO (e.g., Thùng 12 Chai, Kg). Vendor can only adjust quantity, not UoM. If UoM is wrong, PO must be re-created |
| Receipt confirmation | Store confirms Receipt in Odoo — sets final qty_done. DO status becomes Done, showing final received amounts |
| Returns | RPO (Return Purchase Order) + RN (Return Note / Biên Bản Trả Hàng). Vendor can only set pickup date and confirm — cannot change quantities. Must sign and print RN like DO |
| PO auto-cancel | If vendor does not confirm or reject within 7 days past Expected Arrival date, PO is auto-cancelled |
| Vendor PO confirmation | Vendor confirms Sent RFQ via portal → Odoo PO state becomes Confirmed, DO is auto-created (no email sent) |
| Vendor RFQ rejection | Vendor rejects Sent RFQ via portal → Odoo PO state becomes Cancelled + email to PO creator |
| Post-signature locking | DO locked after vendor signs — portal admin (buyer) can unlock directly in the portal (vendor notified by email, no reason required) |
| Data retention | 24 months — POs older than 24 months are permanently deleted from portal DB. Applies to all statuses |
| Data export | Vendors can export as PDF (individual or summary) or CSV. Single or bulk export. Date range filter available |
| Backorder handling | Vendor submits qty only — store validates and decides in Odoo |
| Admin role | Separate admin account, portal-managed password |
| Admin capabilities | View all vendors, trigger sync, view all POs/DOs, unlock signed DOs, download any PDF |
| Admin UI | Same layout as vendor portal with additional admin menu items |
| PO list search | Filter by PO number and date range |
| Vendor dashboard summary | PO counts by status (Waiting, Confirmed, Cancelled) shown above PO list |
| Vendor comment on DO | Free-text note added when vendor signs the DO |
| PDF retention | 24 months — matching PO data retention |
| Responsive design | Works on both desktop and mobile equally |
| Vendor accounts | 1 Odoo partner = 1 portal account. Login by Vendor ID (`res.partner.id`). No multi-user per vendor |
| Profile changes | All profile changes must go through Odoo — admin cannot edit in portal |
| Admin language | Bilingual (Vietnamese + English) — same as vendor portal |
| Audit logging | Key user actions are logged: login, PO confirm/reject, DO update/sign/unlock, receipt validation |
| Email notifications | Invite, password reset, PO rejection (to PO creator), receipt confirmed (to vendor with discrepancy alert if any), DO unlocked (to vendor) |
| Store email recipient | Email sent to the person who created the PO in Odoo (not a generic inbox) |
| Email service | AWS SES |

---

## Architecture Overview

```mermaid
flowchart TD
    classDef client   fill:#FEF3C7,stroke:#F59E0B,color:#451A03,font-weight:bold
    classDef infra    fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold
    classDef app      fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef store    fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef external fill:#FEE2E2,stroke:#EF4444,color:#7F1D1D,font-weight:bold

    Browser(["Vendor / Admin Browser"]):::client

    subgraph VM_PORTAL["Portal VM — Docker Compose"]
        Nginx["Nginx\nReverse Proxy + TLS termination"]:::infra
        React["React + Vite\nFrontend container\nserves /"]:::app
        FastAPI["FastAPI\nBackend container\nserves /api/"]:::app
        PG["PostgreSQL\nVendor accounts, tokens\nDO records, audit log"]:::app
        Redis["Redis\nRate limiting\nToken blacklist"]:::app
    end

    subgraph VM_ODOO["Odoo VM"]
        Odoo["Odoo 16 CE\nPurchase, Inventory\nPartner data"]:::store
    end

    SES["AWS SES\nOutbound email"]:::external

    Browser -->|HTTPS| Nginx
    Nginx -->|"/"| React
    Nginx -->|"/api/"| FastAPI
    FastAPI --- PG
    FastAPI --- Redis
    FastAPI -->|"XML-RPC — read/write\nbutton_confirm, button_cancel\nqty_done, partner sync"| Odoo
    FastAPI -->|"HTTP session\nPDF download"| Odoo
    Odoo -.->|"Webhook / polling TBD\nreceipt validated"| FastAPI
    FastAPI -->|"Send email\nboto3"| SES
```

**Key principle:** The React frontend never contacts Odoo. All Odoo communication is proxied through the FastAPI backend using a single service account. Vendor credentials never leave the portal's own database.

---

> Chi tiết implementation flows (Auth, Sync, DO, Receipt) xem tại [Data Flow Summary](roadmap.md#data-flow-summary) trong roadmap.md.

---

## Business Logic and Flow

This section describes the portal's behaviour in plain business terms, intended for stakeholder communication. It covers the four main scenarios: onboarding a new vendor, day-to-day portal usage, the delivery confirmation workflow, and exception handling.

---

### 1. Vendor Onboarding

**Trigger:** A new vendor is registered in Odoo with `is_vendor = True` **and** a valid email address on their partner record.

**What happens automatically:**
1. The portal sync job runs every 6 hours and detects the new vendor in Odoo
2. A portal account is created, linked to the vendor's Odoo ID
3. The vendor receives a **Welcome Email** (in Vietnamese by default) containing:
   - Their **Vendor ID** (`res.partner.id`) — the integer they will use to log in
   - A **set-password link** valid for 24 hours
4. The vendor clicks the link, sets their own password, and their account becomes active
5. From this point, the vendor can log in at any time using their Vendor ID and password

**If the vendor misses the 24h window:** they use the "Forgot Password" option on the login page, enter their Vendor ID, and receive a new reset link.

**If the vendor has no email in Odoo:** the sync job skips them and logs the case. The internal team must add an email to the Odoo partner record — the account will be created on the next sync cycle.

**Profile updates:** If the vendor's name, phone, tax ID, or company name changes in Odoo, the portal reflects those changes automatically on the next sync. The vendor's **password** is managed on the portal only and is never overwritten by sync. The Vendor ID never changes — it is the permanent `res.partner.id` assigned by Odoo.

---

### 2. Viewing Purchase Orders

**Who can see what:** Each vendor sees only their own Purchase Orders. It is technically impossible for a vendor to view another vendor's data.

**Which POs are visible (portal statuses):**
- **Waiting** — RFQ has been sent to the vendor and is awaiting confirmation. Vendor can confirm or reject it.
- **Confirmed** — PO has been approved, DO has been created. Vendor can edit and sign the DO.
- **Cancelled** — PO has been cancelled (vendor rejected, or store cancelled). Read-only.

Draft RFQs are not shown. Vendors can view PO data for **24 months** from creation date. Older POs are permanently deleted.

**Confirming a Sent RFQ:**
- A "Confirm PO" button is shown on Waiting PO detail pages
- Vendor clicks "Confirm PO" → Odoo PO state becomes Confirmed, Odoo auto-creates the linked DO (stock.picking), portal reads and displays it
- Once confirmed, the button is no longer shown — the PO is read-only

**Rejecting a Sent RFQ:**
- A "Reject" button is shown alongside "Confirm PO" on Waiting PO detail pages
- Vendor clicks "Reject" → Odoo PO state becomes Cancelled
- Portal sends an email notification to the PO creator (store staff who created the RFQ)
- Rejection is final — the store must create a new RFQ if they wish to re-order

**Auto-cancellation:**
- If a vendor does not confirm or reject a Waiting PO within **7 days past the Expected Arrival date**, the portal automatically cancels it in Odoo
- A scheduled job checks daily for overdue Waiting POs
- Email notification sent to **both vendor and PO creator** when a PO is auto-cancelled

**Searching and filtering:**
- Vendor can search by PO number (e.g. typing "PO004" filters the list instantly)
- Vendor can filter by date range (e.g. "POs from January to March")
- Results are paginated — 20 POs per page

**PO detail view:** clicking a PO shows the full list of ordered products with quantities and expected delivery dates, plus the linked DO with its status (Draft / Signed / Done / Cancelled).

---

### 3. Delivery Order & Delivery Workflow

This is the core business process of the portal. It covers the full flow from DO creation to physical delivery at the store.

```mermaid
flowchart TD
    classDef vendor   fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef portal   fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef store    fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold

    S1(["PO confirmed in Odoo"]):::store
    S2["Odoo auto-creates DO (stock.picking)\nPortal reads and displays it\nStatus: Draft"]:::portal

    V1["Vendor edits DO\nSingle delivery date\nQty per line — must not exceed ordered qty\nBase UoM only\nCan save multiple times"]:::vendor
    V2["Vendor clicks Sign DO\nDraws digital signature\nOptional comment"]:::vendor
    V3["Signed DO PDF generated\nVietnamese — PO barcode\nDO locked — no further edits"]:::portal
    V4["Vendor prints PDF\n2 copies"]:::vendor
    V5["Vendor brings goods + 2 paper DOs\nto store"]:::vendor

    D1["Store receives goods\nChecks quantities"]:::store
    D2["Both parties sign physically\n2 paper copies — each keeps 1"]:::store

    E1["Store confirms Receipt in Odoo\nqty_done finalized"]:::store
    E2["DO status: Done\nFinal received qty visible on portal"]:::portal
    E3["Email to vendor\nReceipt confirmed\nAlert if qty differs from DO"]:::portal

    S1 --> S2 --> V1 --> V2 --> V3 --> V4 --> V5
    V5 --> D1 --> D2 --> E1 --> E2 --> E3
```

> 🟢 Xanh lá = Vendor action | 🔵 Xanh dương = Store action | 🟣 Tím = Portal System

**Key points for stakeholders:**
- Vendor ký xác nhận DO trên Portal trước khi giao hàng — PDF có chữ ký là chứng từ chính thức
- Vendor cầm 2 bản DO (PDF đã ký) ra cửa hàng — cửa hàng ký physically cả 2 bản
- Cửa hàng xác nhận trên Odoo sau khi nhận hàng và ký DO
- Không có quy trình tự động khoá DO — vendor chủ động ký khi sẵn sàng giao
- No email is sent when vendor signs — the signed PDF is the vendor's own document to bring to the store

> Sơ đồ toàn bộ flow (bao gồm RFQ → PO → Returns) xem tại [Business Overview](PROCESS_FLOW.md#business-overview-simple-view) trong PROCESS_FLOW.md. Diagram trên chỉ thể hiện phần DO & Delivery (từ sau khi PO đã confirmed).

---

### 4. PO and DO Status Lifecycle

**PO Statuses (portal-managed):**

| Portal PO Status | Odoo PO State | Trigger | Vendor can do |
|---|---|---|---|
| **Waiting** | `sent` | Store sends RFQ | Confirm or Reject |
| **Confirmed** | `purchase` | Vendor confirms PO | View DO, export data |
| **Cancelled** | `cancel` | Vendor rejects or store cancels | Read-only |

**DO Statuses (portal-managed):**

| DO Status | Trigger | Vendor can do |
|---|---|---|
| **Draft** | PO confirmed, DO auto-created | Edit delivery date + quantities, save multiple times |
| **Signed** | Vendor signs DO digitally | Read-only, print DO PDF (multiple times). Data pushed to Odoo Receipt |
| **Done** | Store confirms Receipt in Odoo | Read-only, view final received amounts alongside delivery amounts, export PDF/CSV |
| **Cancelled** | PO cancelled (before receipt confirmation) | Read-only |

**Cancellation rules:**
- A PO can only be cancelled if the linked Receipt has **not** been confirmed by the store (no `qty_done`)
- If the Receipt has already been confirmed in Odoo, cancellation is blocked — the portal shows a warning and the vendor must contact procurement to resolve

---

### 5. Post-Signature Locking and Unlocking

**Why DOs are locked after signing:** Once a vendor confirms delivery quantities with their digital signature, the DO becomes the official delivery document. The signed PDF is what the vendor prints and brings to the store. Allowing edits after signing would undermine the document's integrity.

**What "locked" means in practice:**
- All quantity and date fields on the DO are read-only in the portal
- The signed DO PDF remains downloadable and printable at any time
- The vendor cannot re-sign or submit a new signature
- The delivery date and set quantities have already been pushed to Odoo's Receipt

**Unlocking process:**
If quantities were entered incorrectly and the vendor needs to resubmit, a portal admin can unlock the DO directly from the admin section (`/admin/vendors/:id`). Unlocking:
1. Removes the portal lock on the DO
2. Sends an email notification to the vendor
3. The vendor can then update quantities/date and re-sign
4. The unlock action is logged in the audit log

---

### 6. Email Notifications Summary

| Event | Recipient | Language | Content |
|---|---|---|---|
| New vendor account created | Vendor | Vietnamese (default) | Vendor ID (integer) + set-password link |
| Password reset requested | Vendor | Vendor's preferred language | Reset link (24h expiry) |
| Vendor rejects RFQ | PO creator (store staff) | Vietnamese | PO rejected + cancelled in Odoo |
| PO auto-cancelled (7 days) | Vendor + PO creator | Vendor's pref lang / Vietnamese | PO auto-cancelled due to no response within 7 days past Expected Arrival |
| Store confirms receipt | Vendor | Vendor's preferred language | Receipt confirmed notification. Alerts if any quantities differ between DO and receipt |
| RPO created by store | Vendor | Sent by Odoo (Send by Email) | Return order notification — vendor should log into portal to confirm |
| DO unlocked by admin | Vendor | Vendor's preferred language | Notification that DO has been unlocked for re-editing |

**Note:** No email is sent when a vendor confirms a PO (the confirmation is pushed to Odoo in real-time) or when a vendor signs a DO/RN (the data is pushed to Odoo automatically). The store email recipient is always the specific person who created the PO in Odoo, not a generic team inbox. RPO email is sent by Odoo natively (not by the portal).

---

### 7. What the Portal Does NOT Do

It is equally important for stakeholders to understand the boundaries of the portal:

- **Does not validate stock movements** — all Odoo validation is done by the store in Odoo
- **Does not create Purchase Orders** — vendors can confirm or reject a Sent RFQ but cannot create POs or edit PO lines
- **Does not handle invoicing or payments** — outside the scope of this portal (vendors can export data for their own invoicing)
- **Does not manage backorders** — the store decides on backorders in Odoo after reviewing quantities
- **Does not modify Odoo's base behaviour** — Odoo states and workflows remain unchanged
- **Does not expose any Odoo credentials to vendors** — vendors have no access to Odoo, directly or indirectly
- **Does not allow vendors to see other vendors' data** — enforced at every layer of the system

---

> Chi tiết kỹ thuật về cơ chế sync Odoo ↔ Portal xem tại [Odoo ↔ Portal Sync](roadmap.md#odoo--portal-sync--open-questions--pending-it-confirmation) trong roadmap.md.

---

> For implementation phases, DB schema, API endpoints, and developer gotchas see [roadmap.md](roadmap.md).
