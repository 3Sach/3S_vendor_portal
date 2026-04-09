# 3SACH Vendor Portal — Luồng Quy Trình

---

## Tổng Quan Nghiệp Vụ (Nhìn Đơn Giản)

```mermaid
flowchart LR
    classDef store    fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold
    classDef vendor   fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef portal   fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef internal fill:#FEF3C7,stroke:#F59E0B,color:#451A03,font-weight:bold

    S1([🏪 Cửa hàng\ntạo RFQ\ntrên Odoo]):::store
    S2([📧 Gửi RFQ\ncho nhà cung cấp]):::store

    V1([✅ Nhà cung cấp\nxác nhận PO\ntrên Portal]):::vendor
    P_DO([🔄 Odoo tự tạo DO\ndựa trên PO đã confirmed]):::portal
    V2([📦 Điền số lượng\nghi chú, ngày giao\ntrên Portal]):::vendor

    P_CUT{{23:00 đêm trước\nngày giao\nDO tự động Locked}}:::portal

    V3([🖨️ In DO PDF\nbản cuối — 2 bản]):::vendor
    V4([🚚 Cầm 2 tờ DO\nra cửa hàng]):::vendor

    I1([🏪 Cửa hàng ký\nphysically 2 tờ DO\nmỗi bên giữ 1 tờ]):::store
    I2([✔️ Cửa hàng\nxác nhận Receipt\ntrên Odoo]):::store

    S1 --> S2 --> V1
    V1 --> P_DO --> V2
    V2 --> P_CUT
    P_CUT --> V3
    V3 --> V4
    V4 --> I1
    I1 --> I2
    I2 --> DONE([🎉 Hoàn tất]):::store
```

> **Màu sắc:** 🔵 Xanh dương = Cửa hàng | 🟢 Xanh lá = Nhà cung cấp | 🟣 Tím = Hệ thống Portal | 🟡 Vàng = Nội bộ 3Sach

---

## Toàn Bộ Quy Trình Mua Hàng: RFQ -> Xác Nhận PO -> DO -> Giao Hàng -> Biên Nhận

```mermaid
flowchart TD
    subgraph ONBOARD["Onboarding Nhà Cung Cấp"]
        A1(["Nhà cung cấp mới được thêm vào Odoo\nis_vendor = True + có email"]) --> A2[Job đồng bộ chạy mỗi 6h]
        A2 --> A3{Có email?}
        A3 -- Không --> A4[Bỏ qua & ghi log để xử lý thủ công]
        A3 -- Có --> A5["Tạo tài khoản portal\nchưa kích hoạt, chưa có mật khẩu"]
        A5 --> A6["Gửi Email Chào Mừng qua AWS SES\nVendor ID + link đặt mật khẩu 24h"]
        A6 --> A7["Nhà cung cấp đặt mật khẩu\nTài khoản được kích hoạt"]
    end

    subgraph RFQ_PO["RFQ đến Xác Nhận / Từ Chối PO"]
        B1(["Cửa hàng tạo RFQ trong Odoo"]) --> B2["Sent RFQ\nEmail gửi cho nhà cung cấp"]
        B2 --> B3{"Hành động của nhà cung cấp\ntrên Portal?"}
        B3 -- Xác nhận --> B4["Nhà cung cấp nhấp Xác nhận PO\ntrên Vendor Portal"]
        B4 --> B5["Portal gọi button_confirm\ntrên purchase.order qua XML-RPC"]
        B5 --> B6["PO được xác nhận trong Odoo\nstate: sent → purchase\nKhông gửi email"]
        B3 -- Từ chối --> B7["Nhà cung cấp nhấp Từ chối\ntrên Vendor Portal"]
        B7 --> B8["Portal gọi button_cancel\ntrên purchase.order qua XML-RPC"]
        B8 --> B9["RFQ bị hủy trong Odoo\nstate: sent → cancel"]
        B9 --> B10["Email đến PO creator\nRFQ bị từ chối bởi nhà cung cấp"]
        B3 -- Không hành động / 7 ngày sau Expected Arrival --> B3X["Portal tự động hủy\nbutton_cancel qua XML-RPC"]
        B3X --> B3Y["Email đến Nhà cung cấp + PO creator\nPO tự động hủy — không phản hồi"]
    end

    subgraph DO["Delivery Order — Vendor Portal"]
        C1["Odoo tự tạo 1 DO (stock.picking)\nkhi PO confirmed\nNgày giao mặc định = ngày mong muốn từ PO"] --> C2["Nhà cung cấp sửa DO trên portal:\n- Ngày giao thực tế (→ scheduled_date)\n- Số lượng giao (≤ qty đặt)\n- Ghi chú inline trên từng dòng SP"]
        C2 --> C3["Nhà cung cấp in DO PDF\n(in trực tiếp, không cần ký)\nCó thể in nhiều lần"]
        C3 --> C4["23:00 đêm trước ngày giao trên DO\nDO tự động → Locked\nPush qty lên Odoo Receipt"]
        C4 --> C5["Nhà cung cấp in DO PDF phiên bản cuối\nIn 2 bản"]
    end

    subgraph DELIVERY["Giao Hàng Thực Tế — Cửa Hàng"]
        D1["Nhà cung cấp mang 2 bản DO in\nđến cửa hàng cùng với hàng hoá"] --> D2["Cửa hàng nhận hàng\nKiểm tra số lượng\n(qty đã pre-fill từ DO)"]
        D2 --> D3["Cả hai bên ký tay\n2 bản giấy DO\nMỗi bên giữ 1 bản"]
        D3 --> D4["Cửa hàng xác nhận trên Odoo\n(có thể điều chỉnh qty nếu hàng hư)"]
    end

    subgraph STORE_CONFIRM["Xác Nhận Biên Nhận — Odoo"]
        E1["Cửa hàng xác nhận Receipt\nqty_done finalized\nstock move được tạo"] --> E2["Nightly sync 23:00\nPortal cập nhật DO → Done"]
        E2 --> E3["Email đến nhà cung cấp\nXác nhận biên nhận\nCảnh báo nếu qty chênh lệch"]
    end

    subgraph UNLOCK["Ngoại Lệ: Mở Khoá DO"]
        G1(["Nhà cung cấp nhập sai số lượng"]) --> G2["Portal Admin mở khoá DO\nTrạng thái DO trở về Draft"]
        G2 --> G3[Email thông báo gửi đến nhà cung cấp]
        G3 --> G4[Nhà cung cấp cập nhật DO và in lại]
    end

    subgraph RETURNS["Quy Trình Trả Hàng"]
        R1(["Cửa hàng tạo RPO trong Odoo\nEmail gửi cho nhà cung cấp"]) --> R2["RPO xuất hiện trên portal\nNhà cung cấp thấy các mặt hàng trả\n(không cần bấm Xác Nhận RPO)"]
        R2 --> R3["Nhà cung cấp chỉnh ngày nhận hàng\nKhông thể từ chối hoặc thay đổi qty"]
        R3 --> R4["Nhà cung cấp in RN PDF"]
        R4 --> R5["Nhà cung cấp đến cửa hàng\nđể lấy hàng trả về\nCả hai ký 2 bản giấy"]
        R5 --> R6["Cửa hàng xác nhận biên nhận trả hàng\ntrong Odoo"]
        R6 --> R7[Trạng thái RN: Done]
    end

    %% Kết nối các giai đoạn chính
    A7 --> B2
    B6 --> C1
    C5 --> D1
    D4 --> E1
    C4 -.->|"Trạng thái Locked / Khoá"| G1
    G4 -.->|"Quay lại chỉnh sửa"| C2
```

---

## Sơ Đồ Swimlane (4 Bên Tham Gia)

```mermaid
sequenceDiagram
    actor Vendor as Nhà cung cấp
    participant Portal as Vendor Portal
    participant Odoo as Odoo (XML-RPC)
    participant Store as Cửa hàng (chỉ Odoo)

    Note over Vendor,Store: ── Giai đoạn 1: RFQ & Xác Nhận PO ──
    Odoo->>Vendor: Email Sent RFQ
    Vendor->>Portal: Xác nhận hoặc Từ chối PO
    Portal->>Odoo: button_confirm / button_cancel trên purchase.order
    Odoo-->>Portal: Trạng thái PO được cập nhật
    Odoo-->>Portal: DO được tạo tự động (nếu xác nhận)

    Note over Vendor,Store: ── Giai đoạn 2: Delivery Order — Sửa & In ──
    Portal->>Vendor: DO sẵn sàng — có thể chỉnh sửa (Draft)
    Vendor->>Portal: Chỉnh sửa DO: số lượng + ghi chú inline trên từng dòng SP
    Vendor->>Portal: In DO PDF (in trực tiếp, không cần ký, có thể in nhiều lần)
    Note over Portal: 23:00 cutoff → DO tự động Locked, push qty lên Odoo
    Portal->>Vendor: In DO PDF phiên bản cuối (2 bản)

    Note over Vendor,Store: ── Giai đoạn 3: Giao Hàng Thực Tế ──
    Vendor->>Store: Mang 2 bản DO + hàng hoá đến cửa hàng
    Store->>Vendor: Cả hai ký tay 2 bản giấy DO — mỗi bên giữ 1 bản

    Note over Vendor,Store: ── Giai đoạn 4: Xác Nhận Biên Nhận trên Odoo ──
    Store->>Odoo: Xem xét Receipt, điều chỉnh số lượng nếu hàng hư
    Store->>Odoo: Xác nhận Receipt — qty_done finalized
    Odoo->>Portal: Nightly sync 23:00 → portal cập nhật DO → Done
    Portal->>Vendor: Email: xác nhận biên nhận (cảnh báo nếu qty chênh lệch)

    Note over Vendor,Store: ── Giai đoạn 5: Trả Hàng ──
    Store->>Odoo: Tạo RPO
    Odoo->>Vendor: Email: thông báo đơn trả hàng
    Vendor->>Portal: Xem RPO, chỉnh ngày nhận hàng (không cần xác nhận RPO)
    Vendor->>Portal: In RN PDF
    Vendor->>Store: Lấy hàng trả về với RN in ra (2 bản)
    Store->>Odoo: Xác nhận biên nhận trả hàng
    Odoo->>Portal: Trạng thái RN -> Done

    Note over Vendor,Store: ── Giai đoạn 6: Xuất Dữ Liệu ──
    Vendor->>Portal: Xuất PDF (riêng lẻ hoặc tổng hợp) hoặc CSV

    Note over Vendor,Store: ── Ngoại Lệ: Mở Khoá DO ──
    Portal->>Vendor: Admin mở khoá DO (trạng thái -> Draft) + email thông báo
    Vendor->>Portal: Chỉnh sửa lại số lượng DO và in lại
```

---

## Luồng Xác Thực

### Xác Thực Nhà Cung Cấp
1. Nhà cung cấp nhập **Vendor ID** (số nguyên — `res.partner.id` từ Odoo) và mật khẩu trên trang đăng nhập
2. Backend tra cứu `vendor_users` theo `odoo_partner_id`; xác minh bcrypt luôn chạy (kể cả khi ID không tồn tại — an toàn về mặt timing)
3. Cùng một thông báo lỗi chung cho cả sai ID lẫn sai mật khẩu — không tiết lộ ID có tồn tại hay không
4. Thành công: cấp phát **JWT access token** (30 phút) và **refresh token** (7 ngày), cả hai đều mang `role: vendor`
5. Tất cả các request tiếp theo mang access token trong header `Authorization`
6. Khi nhận 401: frontend gọi thầm `/api/auth/refresh` → thử lại một lần với token mới
7. Khi refresh thất bại: xóa storage → chuyển hướng về `/login`

### Xác Thực Admin
- Admin đăng nhập tại `/admin/login` với **username** (không phải Vendor ID) và mật khẩu
- JWT mang `role: admin` — token admin không thể truy cập route vendor và ngược lại
- Tài khoản admin đầu tiên được seed từ biến môi trường `ADMIN_INITIAL_PASSWORD` khi khởi động

### Token Blacklist
- Khi đăng xuất, refresh token được thêm vào Redis với TTL khớp với thời gian còn lại của token

---

## Thông Báo Email

| Sự kiện | Người nhận | Ngôn ngữ | Nội dung |
|---|---|---|---|
| Tài khoản vendor mới được tạo (job đồng bộ) | Nhà cung cấp | Tiếng Việt (mặc định) | Vendor ID (số nguyên) + link đặt mật khẩu (hết hạn sau 24h) |
| Yêu cầu đặt lại mật khẩu | Nhà cung cấp | Ngôn ngữ ưa thích của nhà cung cấp | Link đặt lại (hết hạn sau 24h) |
| RFQ bị từ chối bởi nhà cung cấp | PO creator (nhân viên cửa hàng) | Tiếng Việt | Mã RFQ, tên nhà cung cấp |
| PO tự động hủy (7 ngày sau Expected Arrival) | Nhà cung cấp + PO creator | Ngôn ngữ ưa thích / Tiếng Việt | PO tự động hủy — không phản hồi trong 7 ngày |
| DO được in bởi nhà cung cấp | — | — | Không gửi email khi in |
| Biên nhận được cửa hàng xác nhận (qty khớp) | Nhà cung cấp | Ngôn ngữ ưa thích của nhà cung cấp | Xác nhận kèm số PO, mã biên nhận |
| Biên nhận được cửa hàng xác nhận (qty chênh lệch) | Nhà cung cấp | Ngôn ngữ ưa thích của nhà cung cấp | Cảnh báo kèm chi tiết chênh lệch |
| DO được admin mở khoá | Nhà cung cấp | Ngôn ngữ ưa thích của nhà cung cấp | Thông báo DO đã sẵn sàng để chỉnh sửa lại |

> Tất cả email gửi đến nhà cung cấp tuân theo `vendor_users.preferred_language` (`vi` hoặc `en`). Email nội bộ luôn bằng tiếng Việt. Gửi email qua **AWS SES** với IAM user chuyên dụng (chỉ có quyền `ses:SendEmail`).

---

## Quyền Hạn Admin

Portal admin dùng chung bố cục giao diện với nhà cung cấp và có thêm các mục menu: **Nhà cung cấp**, **Trạng thái Sync**, **Audit Log**.

| Quyền hạn | Endpoint |
|---|---|
| Xem toàn bộ tài khoản nhà cung cấp (trạng thái, lần đăng nhập cuối, số biên nhận) | `GET /api/admin/vendors` |
| Xem chi tiết PO và biên nhận của một nhà cung cấp cụ thể | `GET /api/admin/vendors/{partner_id}` |
| Kích hoạt / vô hiệu hoá tài khoản nhà cung cấp | `PATCH /api/admin/vendors/{partner_id}/deactivate|reactivate` |
| Tải xuống PDF DO của bất kỳ nhà cung cấp nào | `GET /api/admin/delivery-orders/{do_id}/pdf` |
| Mở khoá DO đã Locked (nhà cung cấp có thể chỉnh sửa và in lại) | `POST /api/admin/delivery-orders/{do_id}/unlock` |
| Kích hoạt thủ công job đồng bộ partner Odoo | `POST /api/admin/sync` |
| Xem trạng thái sync (lần chạy cuối, số vendor đã sync, số bị bỏ qua) | `GET /api/admin/sync/status` |
| Xem audit log có phân trang | `GET /api/admin/audit-log` |

**Audit log** ghi lại các loại hành động: `login`, `po_confirm`, `po_reject`, `po_auto_cancel`, `do_update`, `do_print`, `do_unlock`, `rn_print`, `receipt_validated`. Chỉ được ghi thêm — không thể chỉnh sửa hoặc xoá qua portal.

**Không thể chỉnh sửa hồ sơ trong portal.** Mọi thay đổi hồ sơ nhà cung cấp (tên, email, điện thoại, công ty) phải thực hiện trong Odoo và sẽ được đồng bộ trong chu kỳ 6 giờ tiếp theo. Admin không thể chỉnh sửa hồ sơ nhà cung cấp trực tiếp.

---

## Ánh Xạ Trạng Thái PO: Portal vs Odoo

Portal và Odoo duy trì **nhãn trạng thái khác nhau**. Hành vi gốc của Odoo không bao giờ bị thay đổi.

| Trạng thái PO trên Portal | Trạng thái Odoo | Điều kiện kích hoạt | Nhà cung cấp có thể làm |
|---|---|---|---|
| **Waiting** | `sent` | Cửa hàng gửi RFQ | Xác nhận hoặc Từ chối |
| **Confirmed** | `purchase` | Nhà cung cấp xác nhận PO trên portal | Xem DO, xuất dữ liệu |
| **Canceled** | `cancel` | Nhà cung cấp từ chối, cửa hàng hủy, hoặc tự động hủy (7 ngày sau Expected Arrival) | Chỉ đọc |

---

## State Machine của DO

```mermaid
stateDiagram-v2
    [*] --> Draft : PO được xác nhận,\nOdoo tự động tạo DO

    Draft --> Locked : 23:00 đêm trước ngày giao trên DO\nPush scheduled_date + quantity_done lên Odoo

    Locked --> Draft : Portal Admin mở khoá\n(nhà cung cấp được thông báo qua email)

    Locked --> Done : Nightly sync phát hiện\ncửa hàng đã xác nhận Receipt

    Draft --> Done : Nightly sync phát hiện\nReceipt đã confirm\n(vendor chưa dùng portal)

    Done --> [*]

    Draft --> Canceled : PO bị hủy\n(trước khi xác nhận biên nhận)

    Canceled --> [*]

    note right of Draft
        Nhà cung cấp: sửa qty + ghi chú inline, in PDF — tự do
        In = tạo PDF on-the-fly (không khoá, không push)
        Có thể sửa/in nhiều lần nhiều phiên bản
        Qty phải <= qty đặt hàng, UoM không đổi
    end note

    note right of Locked
        23:00 cutoff đêm trước ngày giao trên DO
        Ngày giao mặc định = ngày mong muốn từ PO
        NCC có thể đổi → cutoff dời theo
        Tự động khoá + push Odoo
        Nhà cung cấp: chỉ đọc, in PDF phiên bản cuối
        Odoo Receipt: quantity_done đã pre-fill
    end note

    note right of Done
        Nhà cung cấp: chỉ đọc, thấy qty thực tế nhận được
        Có thể xuất PDF/CSV
        Gửi email nếu qty chênh lệch
    end note

    note right of Canceled
        Nhà cung cấp: chỉ đọc
        Chỉ có thể xảy ra nếu Receipt chưa được xác nhận
    end note
```

---

## State Machine của RN (Return Note)

```mermaid
stateDiagram-v2
    [*] --> Draft : RPO được cửa hàng tạo,\nRN được tạo tự động

    Draft --> Locked : 23:00 cutoff đêm trước ngày lấy hàng\n(cùng cơ chế với DO)

    Locked --> Done : Nightly sync phát hiện\ncửa hàng xác nhận return receipt

    Done --> [*]

    note right of Draft
        Nhà cung cấp: chỉnh ngày nhận hàng, in PDF — tự do
        Không thể thay đổi số lượng, không thể từ chối
        Không cần bấm Xác Nhận RPO
        In = tạo PDF on-the-fly (không khoá)
    end note

    note right of Locked
        23:00 cutoff: tự động khoá
        Nhà cung cấp: chỉ đọc, in RN PDF phiên bản cuối
    end note

    note right of Done
        Trả hàng hoàn tất
        Có thể xuất PDF/CSV
    end note
```

---

## Trả Hàng: RPO & Return Note (Biên Bản Trả Hàng)

| Khái niệm | Quy trình mua hàng | Quy trình trả hàng |
|---|---|---|
| Đơn hàng | PO (Purchase Order) | RPO (Return Purchase Order) |
| Chứng từ giao nhận | DO (Delivery Order) | RN (Return Note / Biên Bản Trả Hàng) |
| Nhà cung cấp có thể chỉnh sửa | Ngày giao + số lượng + ghi chú inline | Chỉ ngày nhận hàng (không thay đổi qty hay UoM) |
| Nhà cung cấp cần xác nhận | Có (bấm "Xác Nhận" PO) | Không (không cần bấm xác nhận RPO) |
| Chữ ký điện tử | Không — chỉ in PDF | Không — chỉ in PDF |
| PDF có thể in | Có (DO PDF, in nhiều lần) | Có (RN PDF, cùng định dạng) |
| Trao đổi vật lý | Nhà cung cấp giao hàng đến cửa hàng | Nhà cung cấp lấy hàng từ cửa hàng |

---

## Lưu Trữ Dữ Liệu

- Nhà cung cấp có thể xem dữ liệu PO trong **24 tháng** kể từ ngày tạo PO
- Áp dụng cho **tất cả trạng thái PO**: Waiting, Confirmed, Canceled
- Áp dụng cho **tất cả trạng thái DO**: Draft, Locked, Done, Canceled
- Áp dụng cho trả hàng (RPO/RN) như nhau
- PO cũ hơn 24 tháng bị **xoá vĩnh viễn** khỏi cơ sở dữ liệu portal
- Một scheduled cleanup job chạy định kỳ để thực thi quy tắc này

---

## Xuất Dữ Liệu

- Nhà cung cấp có thể xuất dữ liệu dưới dạng **PDF** hoặc **CSV**
- PDF: chứng từ riêng lẻ hoặc báo cáo tổng hợp nhiều bản ghi
- CSV: dữ liệu hàng loạt để nhà cung cấp chỉnh sửa và nhập vào hệ thống của họ
- Giao diện hỗ trợ chọn một hoặc nhiều bản ghi để xuất
- Có bộ lọc theo khoảng ngày
- Bao gồm cả PO/DO thông thường lẫn trả hàng (RPO/RN)

---

## Câu Hỏi Chờ Xác Nhận

| # | Chủ đề | Chi tiết |
|---|---|---|
| 1 | **Hai job 23:00 chạy cùng lúc** | Job cutoff (lock DO + push Odoo) và job nightly sync (check receipt confirmed → DO Done) đều chạy lúc 23:00. Nếu cửa hàng đã confirm receipt trước 23:00 mà DO vẫn Draft → thứ tự xử lý thế nào? DO đi Draft → Locked → Done hay Draft → Done? |
| 2 | **RN có cần Unlock không?** | DO có cơ chế Unlock (admin mở khoá Locked → Draft). RN hiện không có. Nếu ngày nhận hàng sai sau 23:00 cutoff, có cách sửa không? |

---

**Bảng Thuật Ngữ**

| Thuật ngữ | Ý nghĩa |
|---|---|
| NCC | Nhà cung cấp (Vendor) |
| DO | Delivery Order — chứng từ giao hàng của nhà cung cấp, được chỉnh sửa và in trên portal |
| RN | Return Note / Biên Bản Trả Hàng — tương đương DO cho quy trình trả hàng, được nhà cung cấp in |
| Receipt | Phiếu nhập kho — bản ghi hàng nhập trong Odoo, được xác nhận bởi cửa hàng |
| RPO | Return Purchase Order — tương đương PO cho quy trình trả hàng, do cửa hàng tạo |
| SL | Số lượng (Quantity) |
| RFQ | Request for Quotation |
| PO | Purchase Order |
