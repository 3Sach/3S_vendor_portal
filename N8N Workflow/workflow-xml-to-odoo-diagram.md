# Sơ đồ Workflow: XML → Hoá đơn NCC Odoo → Upload File

> Sơ đồ trực quan cho workflow [workflow-xml-to-odoo-vendor-bills.md](workflow-xml-to-odoo-vendor-bills.md)
> Workflow ID: `556wjyTaOwmybPul` — Phiên bản V8 (Tháng 4/2026)

---

## Workflow chia thành 5 Block

| # | Block | Mục đích |
|---|-------|----------|
| 1 | **Trích xuất dữ liệu XML** | Gọi API MatBao → nhận dữ liệu bán cấu trúc (JSON) của hoá đơn + dò số PO |
| 2 | **Kiểm tra Nhà cung cấp** | Tìm NCC trên Odoo theo MST (`res.partner.vat`) |
| 3 | **Kiểm tra Đơn đặt hàng (PO)** | Tìm PO trên Odoo theo số PO hoặc dự phòng theo tiền + ngày |
| 4 | **Kiểm tra Hoá đơn** | Tạo / cập nhật / xác nhận Hoá đơn NCC (`account.move`) |
| 5 | **Upload file PDF/XML vào Hoá đơn** | Đính kèm file XML/PDF (tải từ MatBao) vào Hoá đơn qua `ir.attachment` |

---

## Toàn bộ quy trình (Flow Summary)

```mermaid
flowchart TD
    Start([Trigger lịch 23:00])

    B1["① TRÍCH XUẤT DỮ LIỆU XML<br/>MatBao API + Regex POxxxxxx"]
    B2["② KIỂM TRA NHÀ CUNG CẤP<br/>res.partner theo MST"]
    B3["③ KIỂM TRA ĐƠN ĐẶT HÀNG<br/>purchase.order theo PO / tiền+ngày"]
    B4["④ KIỂM TRA HOÁ ĐƠN<br/>Tạo / Cập nhật / Xác nhận account.move"]
    B5["⑤ UPLOAD FILE PDF/XML<br/>Đính kèm ir.attachment"]
    End([Hoàn tất])

    LogNoPartner[(LOG: no_partner<br/>NCC chưa có trên Odoo)]
    LogNoPO[(LOG: no_po<br/>hoá đơn nháp, xử lý thủ công)]
    LogMismatch[(LOG: amount_mismatch<br/>tiền Odoo ≠ XML)]

    Start --> B1 --> B2
    B2 -- Không thấy NCC --> LogNoPartner
    B2 -- Thấy NCC --> B3
    B3 -- Không thấy PO --> LogNoPO
    B3 -- Thấy PO --> B4
    B4 -- Tiền không khớp --> LogMismatch
    B4 -- Đã posted --> B5
    B5 --> End

    classDef b1 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef b2 fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef b3 fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    classDef b4 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef b5 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef logError fill:#dc2626,stroke:#991b1b,stroke-width:3px,color:#fff

    class B1 b1
    class B2 b2
    class B3 b3
    class B4 b4
    class B5 b5
    class LogNoPartner,LogNoPO,LogMismatch logError
```

---

## Block ① Trích xuất dữ liệu XML

```mermaid
flowchart TD
    Trigger([Trigger lịch 23:00]) --> MatBao[Gọi API MatBao<br/>JSON bán cấu trúc<br/>ky_hieu, so_hoa_don,<br/>mst_ncc, tong_tien,<br/>invoice_lines]
    MatBao --> Regex[Dò Regex PO<br/>POxxxxxx trong XML]
    Regex --> Check{Tìm thấy PO?}
    Check -- Có --> WithPO[has_po = true<br/>po_number, po_all]
    Check -- Không --> NoPO[has_po = false<br/>match bằng tiền+ngày ở Block ③]
    WithPO --> Next[→ Block ②]
    NoPO --> Next
```

**Đầu vào:** API MatBao (thay thế cho việc đọc file XML từ SharePoint)
**Bước Regex:** quét text trong XML để tìm pattern `POxxxxxx` (PO + 6 chữ số trở lên) vì số PO không phải field chuẩn trong hoá đơn điện tử VN — NCC thường ghi vào `ghi_chu` / `ten_hang` / `dien_giai`
**2 nhánh đầu ra:**
- `has_po = true` → Block ③ match PO theo `name`
- `has_po = false` → Block ③ dự phòng match theo `tiền + ngày` (vẫn tiếp tục workflow, không DỪNG)

---

## Block ② Kiểm tra Nhà cung cấp

```mermaid
flowchart TD
    In[mst_ncc từ XML] --> Search[Odoo: tìm res.partner<br/>WHERE vat = mst_ncc]
    Search --> Found{Tìm thấy?}
    Found -- Có --> OK[partner_id → Block ③]
    Found -- Không --> Log[(LOG: no_partner<br/>NCC chưa có trên Odoo)]

    classDef logError fill:#dc2626,stroke:#991b1b,stroke-width:3px,color:#fff
    class Log logError
```

**Model:** `res.partner` | **Field khoá:** `vat` (MST NCC)
**Không có NCC:** ghi vào LOG để kế toán xem + tạo NCC thủ công — workflow dừng xử lý hoá đơn này

---

## Block ③ Kiểm tra Đơn đặt hàng

```mermaid
flowchart TD
    In[partner_id + po_number] --> HasPO{has_po?}
    HasPO -- Có --> ByNum[Tìm purchase.order<br/>WHERE name = po_number<br/>AND partner_id = X<br/>AND invoice_status != 'no']
    HasPO -- Không --> ByAmt[Dự phòng: Tìm<br/>WHERE partner_id = X<br/>AND net_received_subtotal ≈ tong_tien<br/>AND effective_date ≈ ngay_lap]
    ByNum --> Merge[Gộp kết quả] --> Prep[Chuẩn hoá dữ liệu]
    ByAmt --> Merge
    Prep --> Out{Tìm thấy PO?}
    Out -- Có --> A[→ Block ④ Path A]
    Out -- Không --> B[→ Block ④ Path B<br/>tạo hoá đơn nháp không PO]
```

**Model:** `purchase.order` | **Field khoá:** `name`, `partner_id`, `invoice_status`

---

## Block ④ Kiểm tra Hoá đơn

```mermaid
flowchart TD
    In[partner_id + dữ liệu PO từ Block ③]

    In --> FoundPO{Tìm thấy PO?}

    FoundPO -- Không --> PathB[Path B: Không PO]
    PathB --> CreateDraft[Tạo account.move<br/>move_type=in_invoice, state=draft]
    CreateDraft --> LogB[(LOG: no_po)]

    FoundPO -- Có --> PathA[Path A: Có PO]
    PathA --> HasBill{PO đã có Hoá đơn?}

    HasBill -- Có --> GetState[Lấy trạng thái Hoá đơn]
    GetState --> Posted{state = posted?}
    Posted -- Có --> LogSkip[(LOG: posted_ok<br/>đã posted, bỏ qua)]
    Posted -- Không --> Update[Luồng cập nhật]

    HasBill -- Không --> Create[action_create_invoice] --> Reload[Reload PO → invoice_ids]
    Reload --> Update

    Update --> UM[Cập nhật account.move:<br/>invoice_date, ref,<br/>payment_reference]
    UM --> Tax[Tìm + Loop dòng thuế<br/>Cập nhật account.invoice.tax]
    Tax --> Amt{amount_total ≈ tong_tien<br/>±1000đ?}
    Amt -- Không --> LogMismatch[(LOG: amount_mismatch)]
    Amt -- Có --> Post[action_post → state=posted]
    Post --> Next[→ Block ⑤ Upload file]

    classDef logError fill:#dc2626,stroke:#991b1b,stroke-width:3px,color:#fff
    class LogB,LogSkip,LogMismatch logError
```

**Model:** `account.move` + `account.invoice.tax` | **Đầu ra:** `bill_id` đã `posted`

---

## Block ⑤ Upload file PDF/XML vào Hoá đơn

```mermaid
flowchart TD
    In[bill_id đã posted từ Block ④]
    In --> CheckExist[Tìm ir.attachment<br/>WHERE res_model=account.move<br/>AND res_id=bill_id<br/>AND name IN xml_name, pdf_name]

    CheckExist --> Dup{Đã upload?}
    Dup -- Có file trùng tên --> LogSkip[(LOG: attachment_exists<br/>bỏ qua upload)]
    Dup -- Không trùng --> Fetch[Tải file từ MatBao<br/>XML + PDF binary]

    Fetch --> Enc[Code: Encode file → base64]
    Enc --> AX[HTTP: POST ir.attachment<br/>res_model=account.move<br/>res_id=bill_id<br/>datas=base64 XML]
    AX --> AP[HTTP: POST ir.attachment<br/>datas=base64 PDF]

    classDef logError fill:#dc2626,stroke:#991b1b,stroke-width:3px,color:#fff
    class LogSkip logError
```

**Model:** `ir.attachment` (đa hình: `res_model=account.move`, `res_id=bill_id`)
**Bước Check trùng:** search `ir.attachment` theo `res_id = bill_id` và `name IN (xml_name, pdf_name)`
- Nếu đã tồn tại → log `attachment_exists`, bỏ qua upload (tránh duplicate khi workflow chạy lại)
- Nếu chưa có → tải từ MatBao + upload như bình thường

**Không còn di chuyển file trên SharePoint** — dữ liệu gốc nằm trên MatBao, workflow chỉ cần tải về + đính kèm vào Hoá đơn trên Odoo

---

## Quan hệ các Model trên Odoo

```mermaid
erDiagram
    res_partner ||--o{ purchase_order : "partner_id"
    res_partner ||--o{ account_move : "partner_id"
    purchase_order }o--o{ account_move : "qua account_move_purchase_order_rel"
    account_move ||--o{ account_move_line : "move_id"
    account_move ||--o{ account_invoice_tax : "invoice_id"
    account_move ||--o{ ir_attachment : "res_model=account.move, res_id"

    res_partner {
        int id
        string name
        string vat "MST NCC"
    }
    purchase_order {
        int id
        string name "Số PO"
        int partner_id
        string invoice_status
        date effective_date
        float net_received_subtotal
    }
    account_move {
        int id
        string state "draft|posted"
        string move_type "in_invoice"
        date invoice_date
        string ref
        string payment_reference
        float amount_total
    }
    account_invoice_tax {
        int id
        int invoice_id
        string invoice_reference
        string invoice_number
        date date_invoice
    }
    ir_attachment {
        string res_model
        int res_id
        binary datas
        string mimetype
    }
```
