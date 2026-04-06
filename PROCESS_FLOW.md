# 3SACH Vendor Portal — Process Flow

---

## Business Overview (Simple View)

```mermaid
flowchart LR
    classDef store    fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold
    classDef vendor   fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef portal   fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef internal fill:#FEF3C7,stroke:#F59E0B,color:#451A03,font-weight:bold
    classDef decision fill:#FFF,stroke:#6B7280,color:#111827

    S1([🏪 Cửa hàng\ntạo RFQ\ntrên Odoo]):::store
    S2([📧 Gửi RFQ\ncho nhà cung cấp]):::store

    V1([✅ Nhà cung cấp\nxác nhận PO\ntrên Portal]):::vendor
    V2([📦 Điền số lượng\nvà ngày giao\ntrên Portal]):::vendor
    V3([🚚 Giao hàng\nthực tế\ncho kho]):::vendor
    V4([✍️ Ký xác nhận\ntrên Portal\nvà tải PDF]):::vendor

    P1{{Portal tự động\nkhoá DO\nlúc 23:00}}:::portal
    P2{{Portal tạo\nPDF có chữ ký\nvà gửi email}}:::portal

    I1([🔍 Nội bộ\nkiểm tra\nsố lượng]):::internal
    I2([✔️ Xác nhận\nnhập kho\ntrên Odoo]):::internal

    S1 --> S2 --> V1
    V1 --> V2
    V2 --> P1
    P1 --> V3
    V3 --> V4
    V4 --> P2
    P2 --> I1
    I1 --> I2
    I2 --> DONE([🎉 Hoàn tất]):::store
```

> **Colours:** Blue = Store | Green = Vendor | Purple = Portal System | Yellow = Internal 3Sach

---

## Full Purchase Workflow: RFQ -> PO Confirm -> DO -> Delivery -> Receipt

```mermaid
flowchart TD
    subgraph ONBOARD["Vendor Onboarding"]
        A1([New vendor added in Odoo\nsupplier_rank > 0]) --> A2[Sync job runs every 6h]
        A2 --> A3{Has email?}
        A3 -- No --> A4[Skip & log for manual follow-up]
        A3 -- Yes --> A5[Create portal account\ninactive, no password]
        A5 --> A6[Send Welcome Email via AWS SES\nVendor ID (integer) + set-password link 24h]
        A6 --> A7[Vendor sets password\nAccount becomes active]
    end

    subgraph RFQ_PO["🟨 RFQ → PO Confirmation  ·  Cửa hàng / Vendor Portal"]
        B1([Cửa hàng tạo RFQ trên Odoo]) --> B2[Sent RFQ\nEmail gửi cho NCC]
        B2 --> B3{Vendor xác nhận PO\ntrước Order Deadline?}
        B3 -- No --> B3X([PO không được xác nhận\nRFQ hết hạn])
        B3 -- Yes --> B4[Vendor clicks 'Confirm PO'\non Vendor Portal]
        B4 --> B5[Portal calls button_confirm\non purchase.order via XML-RPC]
        B5 --> B6[PO Confirmed\nOdoo cập nhật trạng thái: sent → purchase]
    end

    subgraph DO["🟧 Delivery Order  ·  Vendor Portal"]
        C1[DO sinh ra tự động\nIn và sửa được] --> C2[NCC chỉnh DO trên portal\nNgày giao + số lượng\nSL giao ≤ SL đặt]
        C2 --> C3{Trước 23:00\nđêm trước ngày giao?}
        C3 -- Yes --> C4[DO vẫn có thể chỉnh sửa]
        C4 --> C3
        C3 -- No / 23:00 CUTOFF --> C5[Hệ thống xác nhận SL giao\nKhóa chỉnh sửa trên Vendor Portal]
        C5 --> C6[DO bị khóa\nKhông sửa được]
        C5 --> C7[Push vào Odoo\nCập nhật Receipt: ngày + SL dự kiến]
    end

    subgraph DELIVERY["🟩 Physical Delivery  ·  Kho nhận"]
        D1[Kho nhận DO\nXác nhận SL] --> D2[NCC giao hàng\nKèm 2 tờ DO]
        D2 --> D3[Ký xác nhận 2 tờ DO\nmỗi bên giữ 1 tờ]
    end

    subgraph STORE_CONFIRM["Receipt Confirmation — Odoo"]
        E1[Store reviews Receipt in Odoo\nCan adjust set quantities] --> E2[Store confirms Receipt\nqty_done is finalized]
        E2 --> E3[Portal notified\nDO status -> Done]
        E3 --> E4[Email to vendor:\nReceipt confirmed\nAlerts if qty differs]
    end

    subgraph UNLOCK["Exception: Unlock DO"]
        G1([Vendor entered wrong quantities]) --> G2[Portal Admin unlocks DO\nDO status -> Draft]
        G2 --> G3[Email notification sent to vendor]
        G3 --> G4[Vendor updates DO\nand re-signs]
        G4 --> C3
    end

    subgraph RETURNS["Returns Flow"]
        R1([Store creates RPO in Odoo\nEmail sent to vendor]) --> R2[RPO appears on portal\nVendor sees return items]
        R2 --> R3[Vendor sets pickup date\nClicks Confirm & Sign\nCannot reject or change qty]
        R3 --> R4[RN signed and locked\nVendor prints RN PDF]
        R4 --> R5[Vendor goes to store\nto collect returned goods\nBoth sign 2 paper copies]
        R5 --> R6[Store confirms return\nreceipt in Odoo]
        R6 --> R7[RN status -> Done]
    end

    %% Connect major phases
    A7 --> B2
    B6 --> C1
    C6 --> D1
    C7 --> E1
    D3 --> E1
    C4 -.->|"Signed / Locked state"| UNLOCK
```

---

## Swimlane View (4 Actors)

```mermaid
sequenceDiagram
    actor Vendor
    participant Portal as Vendor Portal
    participant Odoo as Odoo (XML-RPC)
    participant Store as Store (Odoo only)

    Note over Vendor,Team: ── Phase 1: RFQ & PO Confirmation ──
    Odoo->>Vendor: Sent RFQ email (NCC)
    Vendor->>Portal: Confirm PO (trước Order Deadline)
    Portal->>Odoo: button_confirm on purchase.order
    Odoo-->>Portal: PO state: sent → purchase
    Odoo-->>Portal: DO / Receipt created (assigned)

    Note over Vendor,Team: ── Phase 2: Delivery Order Edit ──
    Portal->>Vendor: DO available — editable
    Vendor->>Portal: Chỉnh DO: ngày giao + SL giao (≤ SL đặt)
    Note over Portal: 23:00 CUTOFF — đêm trước ngày giao
    Portal->>Portal: Lock DO, confirm SL giao
    Portal->>Odoo: Push Receipt: ngày + SL dự kiến

    Note over Vendor,Team: ── Phase 3: Physical Delivery ──
    Portal->>Kho: Gửi DO (2 bản)
    Vendor->>Kho: Giao hàng kèm 2 tờ DO
    Kho->>Vendor: Ký xác nhận — mỗi bên 1 tờ

    Note over Vendor,Store: ── Phase 4: Store Receipt Confirmation (Odoo) ──
    Store->>Odoo: Review Receipt, adjust quantities if needed
    Store->>Odoo: Confirm Receipt (qty_done finalized)
    Odoo->>Portal: Receipt validated (sync mechanism TBD)
    Portal->>Portal: DO status -> Done, final received qty stored
    Portal->>Vendor: Email: receipt confirmed (alerts if any qty differs)

    Note over Vendor,Store: ── Phase 5: Returns ──
    Store->>Odoo: Create RPO
    Odoo->>Vendor: Email: return order notification
    Vendor->>Portal: View RPO, set pickup date
    Vendor->>Portal: Confirm & Sign Return Note (one step)
    Portal->>Vendor: RN PDF available for printing
    Vendor->>Store: Collect returned goods with printed RN (2 copies)
    Store->>Odoo: Confirm return receipt
    Odoo->>Portal: RN status -> Done

    Note over Vendor,Store: ── Phase 6: Export ──
    Vendor->>Portal: Export PDF (individual or summary) or CSV

    Note over Vendor,Store: ── Exception: Unlock DO ──
    Portal->>Vendor: Admin unlocks DO (status -> Draft) + email notification
    Vendor->>Portal: Re-edit DO quantities and re-sign
```

---

## Auth Flow

### Vendor Authentication
1. Vendor enters their **Vendor ID** (integer — the `res.partner.id` from Odoo) and password on the login page
2. Backend looks up `vendor_users` by `odoo_partner_id`; bcrypt verification always runs (even for unknown IDs — timing-safe)
3. Same generic error message for wrong ID or wrong password — never reveals whether the ID exists
4. On success: issues a **JWT access token** (30 min) and **refresh token** (7 days), both carrying `role: vendor`
5. All subsequent requests carry the access token in the `Authorization` header
6. On 401: frontend silently calls `/api/auth/refresh` → retries once with new token
7. On refresh failure: clears storage → redirects to `/login`

### Admin Authentication
- Admin logs in at `/admin/login` with a **username** (not a Vendor ID) and password
- JWT carries `role: admin` — admin tokens cannot access vendor routes and vice versa
- First admin account is seeded from `ADMIN_INITIAL_PASSWORD` env variable on startup

### Token Blacklist
- On logout, refresh token is added to Redis with TTL matching remaining lifetime

---

## Email Notifications

| Event | Recipient | Language | Content |
|---|---|---|---|
| New vendor account created (sync job) | Vendor | Vietnamese (default) | Vendor ID (integer) + set-password link (24h expiry) |
| Password reset requested | Vendor | Vendor's preferred language | Reset link (24h expiry) |
| RFQ rejected by vendor | PO creator (internal) | English | RFQ reference, vendor name |
| DO signed by vendor | — | — | No email sent on signing |
| Receipt confirmed by store (qty matches) | Vendor | Vendor's preferred language | Confirmation with PO number, receipt reference |
| Receipt confirmed by store (qty differs) | Vendor | Vendor's preferred language | Alert with difference details |
| DO unlocked by admin | Vendor | Vendor's preferred language | Notification that DO is available to re-edit |

> All vendor-facing emails follow `vendor_users.preferred_language` (`vi` or `en`). Internal emails are always in English. Email delivery is via **AWS SES** with a dedicated IAM user (`ses:SendEmail` only).

---

## Admin Capabilities

Portal admins share the same UI layout as vendors with additional menu items: **Vendors**, **Sync Status**, **Audit Log**.

| Capability | Endpoint |
|---|---|
| View all vendor accounts (status, last login, receipt counts) | `GET /api/admin/vendors` |
| Drill into a specific vendor's POs and receipts | `GET /api/admin/vendors/{partner_id}` |
| Deactivate / reactivate a vendor account | `PATCH /api/admin/vendors/{partner_id}/deactivate|reactivate` |
| Download any vendor's signed PDF | `GET /api/admin/receipts/{picking_id}/pdf` |
| Unlock a signed DO (vendor can then re-edit and re-sign) | `POST /api/admin/receipts/{picking_id}/unlock` |
| Manually trigger the Odoo partner sync job | `POST /api/admin/sync` |
| View sync status (last run, vendors synced, skipped) | `GET /api/admin/sync/status` |
| View paginated audit log | `GET /api/admin/audit-log` |

**Audit log** records four action types: `login`, `qty_update`, `sign`, `unlock`. It is append-only — never editable or deletable through the portal.

**Profile edits are not possible in the portal.** All vendor profile changes (name, email, phone, company) must be made in Odoo and will sync on the next 6-hour cycle. Admin cannot modify vendor profiles directly.

---

## PO Status Mapping: Portal vs Odoo

Portal and Odoo maintain **different status labels**. Odoo's base behaviour is never modified.

> **Implementation scope (README v4):** The portal API returns only POs in `purchase` (Confirmed) and `done` (Done) states for the vendor's main browsing view. The `sent` (Waiting/RFQ) state is surfaced separately for the confirmation/rejection action — vendors must act before a PO moves to `purchase`.

| Portal PO Status | Odoo State | Trigger | Vendor can do |
|---|---|---|---|
| **Waiting** | `sent` | Store sends RFQ | Confirm or Reject |
| **Confirmed** | `purchase` | Vendor confirms PO on portal | View DO, export data |
| **Cancelled** | `cancel` | Vendor rejects, store cancels, or auto-cancel (7 days past Expected Arrival) | Read-only |

---

## DO State Machine

```mermaid
stateDiagram-v2
    [*] --> Ready : PO confirmed,\nDO locked after cutoff

    Draft --> Signed : Vendor edits qty + date,\nsigns digitally

    Signed --> Draft : Portal Admin unlocks\n(vendor notified by email)

    Signed --> Done : Store confirms receipt\nin Odoo

    Done --> [*]

    Draft --> Cancelled : PO cancelled\n(before receipt confirmation)

    Cancelled --> [*]

    note right of Draft
        Vendor: can edit single delivery date + quantities
        Quantities must be <= ordered qty
        UoM inherited from PO — cannot be changed
        Portal: editable
    end note

    note right of Signed
        Vendor: read-only, can print DO PDF
        Portal: locked, pushed to Odoo
        Odoo Receipt: set quantities populated
        No email sent
    end note

    note right of Done
        Vendor: read-only, sees final received qty
        Can export PDF/CSV
        Email sent if qty differs
    end note

    note right of Cancelled
        Vendor: read-only
        Only possible if Receipt not yet confirmed
    end note
```

---

## RN (Return Note) State Machine

```mermaid
stateDiagram-v2
    [*] --> Draft : RPO created by store,\nRN auto-created

    Draft --> Signed : Vendor sets pickup date,\nConfirm & Sign (one step)

    Signed --> Done : Store confirms return\nreceipt in Odoo

    Done --> [*]

    note right of Draft
        Vendor: can set pickup date only
        Cannot change quantities
        Cannot reject
    end note

    note right of Signed
        Vendor: read-only, can print RN PDF
        Portal: locked
    end note

    note right of Done
        Return completed
        Can export PDF/CSV
    end note
```

---

## Returns: RPO & Return Note (Bien Ban Tra Hang)

| Concept | Purchase Flow | Returns Flow |
|---|---|---|
| Order | PO (Purchase Order) | RPO (Return Purchase Order) |
| Delivery document | DO (Delivery Order) | RN (Return Note / Bien Ban Tra Hang) |
| Vendor can edit | Delivery date + quantities (not UoM) | Pickup date only (no qty or UoM change) |
| Vendor can reject | Yes | No |
| Signature | Required (DO) | Required (RN) |
| Printable PDF | Yes (DO PDF) | Yes (RN PDF, same format) |
| Physical exchange | Vendor delivers to store | Vendor collects from store |

---

## Data Retention

- Vendors can view PO data for **24 months** from PO creation date
- Applies to **all PO statuses** equally: Waiting, Confirmed, Cancelled
- Applies to **all DO statuses**: Draft, Signed, Done, Cancelled
- Applies to returns (RPO/RN) equally
- POs older than 24 months are **permanently deleted** from the portal database
- A scheduled cleanup job runs periodically to enforce this rule

---

## Data Export

- Vendors can export data as **PDF** or **CSV**
- PDF: individual document or summary report of multiple records
- CSV: bulk data for vendor to edit and import into their systems
- UI supports selecting single or multiple records for export
- Date range filter available
- Includes both regular POs/DOs and returns (RPO/RN)

---

**Glossary**

| Term | Meaning |
|---|---|
| NCC | Nhà cung cấp (Vendor) |
| DO | Delivery Order — vendor's planned delivery document, edited and signed on portal |
| RN | Return Note / Bien Ban Tra Hang — return equivalent of DO, signed by vendor |
| Receipt | Phiếu nhập kho — Odoo's incoming shipment record, confirmed by store |
| RPO | Return Purchase Order — return equivalent of PO, created by store |
| SL | Số lượng (Quantity) |
| RFQ | Request for Quotation |
| PO | Purchase Order |
| Receipt | Phiếu nhập kho Odoo |
| Cutoff | 23:00 đêm trước ngày giao — DO bị khóa sau mốc này |
