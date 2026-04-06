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
    V3([✍️ Ký xác nhận DO\ntrên Portal\nvà tải PDF]):::vendor
    V4([🚚 Cầm 2 tờ DO\nra cửa hàng]):::vendor

    P1{{Portal tạo\nPDF có chữ ký}}:::portal

    I1([🏪 Cửa hàng ký\nphysically 2 tờ DO\nmỗi bên giữ 1 tờ]):::store
    I2([✔️ Cửa hàng\nxác nhận\ntrên Odoo]):::store

    S1 --> S2 --> V1
    V1 --> V2
    V2 --> V3
    V3 --> P1
    P1 --> V4
    V4 --> I1
    I1 --> I2
    I2 --> DONE([🎉 Hoàn tất]):::store
```

> **Màu sắc:** 🔵 Cửa hàng &nbsp;|&nbsp; 🟢 Nhà cung cấp &nbsp;|&nbsp; 🟣 Hệ thống Portal &nbsp;|&nbsp; 🟡 Nội bộ 3Sach

---

## Full Purchase Workflow: RFQ → PO Confirm → Delivery → Receipt

```mermaid
flowchart TD
    subgraph ONBOARD["🟦 Vendor Onboarding"]
        A1([New vendor added in Odoo\nsupplier_rank > 0]) --> A2[Sync job runs every 6h]
        A2 --> A3{Has email?}
        A3 -- No --> A4[Skip & log for manual follow-up]
        A3 -- Yes --> A5[Create portal account\ninactive, no password]
        A5 --> A6[Send Welcome Email via AWS SES\nVendor ID + set-password link 24h]
        A6 --> A7[Vendor sets password\nAccount becomes active]
    end

    subgraph RFQ_PO["🟨 RFQ → PO Confirmation  ·  Cửa hàng / Vendor Portal"]
        B1([Cửa hàng tạo RFQ trên Odoo]) --> B2[Sent RFQ\nEmail gửi cho NCC]
        B2 --> B4[Vendor clicks 'Confirm PO'\non Vendor Portal]
        B4 --> B5[Portal calls button_confirm\non purchase.order via XML-RPC]
        B5 --> B6[PO Confirmed\nOdoo cập nhật trạng thái: sent → purchase]
    end

    subgraph DO["🟧 Delivery Order  ·  Vendor Portal"]
        C1[DO sinh ra tự động\nsố ref theo PO] --> C2[NCC chỉnh DO trên portal\nNgày giao + số lượng\nSL giao ≤ SL đặt]
        C2 --> C3[NCC ký xác nhận DO\ntrên Portal]
        C3 --> C4[Portal tạo PDF có chữ ký\nDO bị khóa trên Portal]
        C4 --> C5[NCC tải PDF\nvà in 2 bản DO]
    end

    subgraph DELIVERY["🟩 Physical Delivery  ·  Cửa hàng"]
        D1[NCC cầm 2 tờ DO\nra cửa hàng giao hàng] --> D2[Cửa hàng ký physically\n2 tờ DO — mỗi bên giữ 1 tờ]
        D2 --> D3[Cửa hàng xác nhận\ntrên Odoo]
    end

    subgraph PORTAL_CONFIRM["🟪 Receipt Confirmation  ·  Vendor Portal"]
        E1[Vendor mở Receipt\ntrạng thái: Ready / assigned] --> E2[Vendor nhập qty_done\nthực tế từng sản phẩm\ncó thể lưu nhiều lần]
        E2 --> E3[Vendor clicks 'Sign & Confirm']
        E3 --> E4[Vendor ký tên trên màn hình\nsignature_pad.js → PNG]
        E4 --> E5[Portal tạo signed PDF\nOdoo delivery slip + trang xác nhận\nWeasyPrint + pypdf]
        E5 --> E6[Receipt bị khóa trên portal\nKhông chỉnh sửa được]
        E6 --> E7A[Email xác nhận → Vendor\nkèm PDF đính kèm]
        E6 --> E7B[Email cảnh báo → Internal team\nVendor name, PO ref, link Odoo]
    end

    subgraph ODOO_VALIDATE["🟥 Odoo Validation  ·  Internal Team"]
        F1[Internal team review\nsố lượng trong Odoo] --> F2{Số lượng OK?}
        F2 -- Yes --> F3[Admin validates Receipt trong Odoo]
        F2 -- No / Short --> F4[Admin điều chỉnh + quyết định backorder]
        F4 --> F3
        F3 --> F5([Receipt Done\nHoàn tất nhập kho ✓])
    end

    subgraph UNLOCK["🔓 Exception: Unlock Receipt"]
        G1([Vendor nhập sai số lượng]) --> G2[Portal Admin mở khóa\ntại /admin/vendors/:id]
        G2 --> G3[Unlock ghi vào receipt_locks\nunlocked_at + unlocked_by]
        G3 --> G4[Vendor cập nhật qty_done\nvà ký lại]
        G4 --> E3
    end

    %% Connect major phases
    A7 --> B2
    B6 --> C1
    C5 --> D1
    D3 --> E1
    E6 --> F1
    E6 -.->|"Signed / Locked state"| UNLOCK
```

---

## Swimlane View (4 Actors)

```mermaid
sequenceDiagram
    actor Vendor
    participant Portal as Vendor Portal
    participant Odoo as Odoo (XML-RPC)
    participant Kho as Kho nhận
    participant Team as Internal Team

    Note over Vendor,Team: ── Phase 1: RFQ & PO Confirmation ──
    Odoo->>Vendor: Sent RFQ email (NCC)
    Vendor->>Portal: Confirm PO
    Portal->>Odoo: button_confirm on purchase.order
    Odoo-->>Portal: PO state: sent → purchase
    Odoo-->>Portal: DO / Receipt created (assigned)

    Note over Vendor,Team: ── Phase 2: Delivery Order — Ký & Tải PDF ──
    Portal->>Vendor: DO available — editable
    Vendor->>Portal: Chỉnh DO: ngày giao + SL giao (≤ SL đặt)
    Vendor->>Portal: Ký xác nhận DO trên Portal
    Portal->>Portal: Tạo PDF có chữ ký, khóa DO
    Portal->>Vendor: Tải PDF (in 2 bản)

    Note over Vendor,Team: ── Phase 3: Physical Delivery — Cửa hàng ──
    Vendor->>Kho: Cầm 2 tờ DO (PDF) ra cửa hàng giao hàng
    Kho->>Vendor: Ký physically 2 tờ DO — mỗi bên 1 tờ
    Kho->>Odoo: Xác nhận trên Odoo

    Note over Vendor,Team: ── Phase 4: Portal Receipt Confirmation ──
    Vendor->>Portal: Mở Receipt (Ready), nhập qty_done
    Vendor->>Portal: Sign & Confirm + vẽ chữ ký
    Portal->>Portal: Tạo signed PDF (Odoo slip + chữ ký)
    Portal->>Odoo: (Receipt vẫn assigned — chờ admin validate)
    Portal->>Vendor: Email xác nhận + PDF đính kèm
    Portal->>Team: Email cảnh báo: vendor, PO ref, link Odoo

    Note over Vendor,Team: ── Phase 5: Odoo Validation ──
    Team->>Odoo: Review qty, validate Receipt
    Odoo-->>Portal: Receipt state: done
    Note over Odoo: Hoàn tất nhập kho ✓
```

---

## Receipt State Machine

```mermaid
stateDiagram-v2
    [*] --> Ready : PO confirmed,\nDO signed by vendor

    Ready --> Signed : Vendor nhập qty_done\nvà ký tên

    Signed --> Ready : Portal Admin unlock\n(logged in receipt_locks)

    Signed --> Done : Internal team\nvalidates in Odoo

    Done --> [*]

    note right of Ready
        Odoo state: assigned
        Vendor: có thể nhập qty_done
        Portal: editable
    end note

    note right of Signed
        Odoo state: assigned (pending)
        Vendor: read-only, có thể tải PDF
        Portal: locked
    end note

    note right of Done
        Odoo state: done
        Vendor: read-only, có thể tải PDF
        Portal: locked
    end note
```

---

**Glossary**

| Term | Meaning |
|---|---|
| NCC | Nhà cung cấp (Vendor) |
| DO | Delivery Order |
| SL | Số lượng (Quantity) |
| RFQ | Request for Quotation |
| PO | Purchase Order |
| Receipt | Phiếu nhập kho Odoo |
