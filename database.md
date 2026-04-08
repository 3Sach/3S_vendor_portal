# Đặc Tả Dữ Liệu — Odoo  Tables
**Hệ thống:** Backend FastAPI kết nối Odoo thông qua XML-RPC (`xmlrpc.client`)
**Đối tượng:** Backend developer (Python/FastAPI) & Data engineer
**Mục đích:** Tích hợp hệ thống bên ngoài (ERP, WMS, v.v.)
**Phiên bản:** 1.2.0
**Ngày cập nhật:** 2026-04-08
**Lưu ý:** Các trường được liệt kê trong tài liệu này là các trường cần thiết cho vendor portal; ngoài ra còn các trường khác nhưng không liên quan đến mục đích. Dữ liệu trong BigQuery (`PRD_Staging`) phục vụ báo cáo — một số state Odoo (ví dụ: `sent` trên `purchase.order`) có thể không xuất hiện trong dataset BigQuery nhưng vẫn tồn tại trong Odoo thực tế.

**Quyết định đã xác nhận:**
- Protocol: **XML-RPC** (không phải JSON-RPC)
- `purchase.order.state = 'sent'` tồn tại trong Odoo — không có trong BigQuery vì dataset này chỉ phục vụ báo cáo
- `stock.move.line.qty_done` là tên field đúng trên Odoo 16 CE (Odoo 17+ đổi thành `quantity`)
- `purchase_order_line.display_type`: filter `['display_type', '=', False]` trong XML-RPC domain để lấy real line items (tương đương `IS NULL` trong SQL)
- RPO (Return Purchase Order): `purchase_order.order_type = 'purchase_return'`
- UoM name: lưu tiếng Việt sẵn trong Odoo; cách lấy từ `uom.uom` qua XML-RPC — **⚠️ PENDING: cần xác nhận field name và cách map**
- Warehouse code cho DO PDF: lấy từ `stock.warehouse.code` — **⚠️ PENDING: cần xác nhận path từ `stock.picking` → warehouse** 



---

## Mục Lục

1. [purchase_order](#1-purchase_order)
2. [purchase_order_line](#2-purchase_order_line)
3. [res_partner](#3-res_partner)
4. [stock_picking](#4-stock_picking)
5. [product_template](#5-product_template)
6. [product_product](#6-product_product)
7. [Quan hệ giữa các bảng](#7-quan-hệ-giữa-các-bảng)
8. [Hướng dẫn tích hợp hệ thống ngoài](#8-hướng-dẫn-tích-hợp-hệ-thống-ngoài)

---

## 1. `purchase_order`

**Mô tả:** Lưu thông tin header của các đơn đặt hàng (Purchase Order) gửi đến nhà cung cấp. Mỗi bản ghi đại diện cho một PO hoàn chỉnh với trạng thái, tổng tiền, ngày đặt hàng và thông tin theo dõi nhập hàng.

**Dataset:** `PRD_Staging` | **Odoo model:** `purchase.order`

### Các cột

| Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---|---|---|
| `id` | INT64 | NO | Khóa chính, định danh duy nhất của PO |
| `name` | STRING | YES | Số PO (ví dụ: `PO/2024/00001`) |
| `state` | STRING | YES | Trạng thái PO — xem Enum bên dưới |
| `order_type` | STRING | YES | Loại đơn hàng — xem Enum bên dưới |
| `priority` | STRING | YES | Mức độ ưu tiên: `0` = bình thường, `1` = khẩn |
| `origin` | STRING | YES | Nguồn gốc tạo PO (số SO, MRP order, v.v.) |
| `partner_id` | INT64 | YES | FK → `res_partner.id` — Nhà cung cấp |
| `partner_ref` | STRING | YES | Số tham chiếu từ phía nhà cung cấp |
| `user_id` | INT64 | YES | FK → `res_users.id` — Người phụ trách PO |
| `warehouse_id` | INT64 | YES | FK → `stock_warehouse.id` — Kho nhận hàng |
| `currency_id` | INT64 | YES | FK → `res_currency.id` — Đơn vị tiền tệ |
| `currency_rate` | NUMERIC | YES | Tỷ giá áp dụng tại thời điểm tạo PO |
| `company_currency_id` | INT64 | YES | FK → `res_currency.id` — Tiền tệ công ty |
| `payment_term_id` | INT64 | YES | FK — Điều khoản thanh toán |
| `picking_type_id` | INT64 | YES | FK → `stock.picking.type` — Loại phiếu nhập kho |
| `fiscal_position_id` | INT64 | YES | FK — Vị trí thuế áp dụng |
| `incoterm_id` | INT64 | YES | FK — Điều kiện giao hàng Incoterms |
| `dest_address_id` | INT64 | YES | FK → `res_partner.id` — Địa chỉ nhận hàng nếu khác kho |
| `analytic_account_id` | INT64 | YES | FK — Tài khoản phân tích chi phí |
| `group_id` | INT64 | YES | FK — Nhóm mua hàng (dùng gom PO) |
| `date_order` | DATETIME | YES | Ngày tạo / xác nhận đơn hàng |
| `date_approve` | DATETIME | YES | Ngày duyệt đơn hàng |
| `date_planned` | DATETIME | YES | Ngày dự kiến nhận hàng (tổng hợp từ các dòng) |
| `date_planned_mps` | DATETIME | YES | Ngày dự kiến theo kế hoạch MPS |
| `effective_date` | DATETIME | YES | Ngày hiệu lực thực tế của PO |
| `vendor_date_planned` | DATETIME | YES | Ngày giao hàng do NCC cam kết |
| `vendor_date_planned_tz` | STRING | YES | Timezone của `vendor_date_planned` |
| `date_sent_rfq` | DATETIME | YES | Ngày gửi RFQ cho NCC |
| `date_calendar_start` | DATETIME | YES | Ngày bắt đầu tính lịch giao hàng |
| `amount_untaxed` | NUMERIC | YES | Tổng tiền trước thuế |
| `amount_tax` | NUMERIC | YES | Tổng thuế VAT |
| `amount_total` | NUMERIC | YES | Tổng tiền sau thuế (bao gồm VAT) |
| `amount_tax_by_received_qty` | NUMERIC | YES | Thuế VAT tính theo số lượng đã nhận |
| `order_discount` | NUMERIC | YES | Chiết khấu cấp đơn hàng |
| `comp_amount_untaxed` | NUMERIC | YES | Tổng tiền chưa thuế quy về tiền tệ công ty |
| `comp_amount_tax` | NUMERIC | YES | Tổng thuế quy về tiền tệ công ty |
| `comp_amount_total` | NUMERIC | YES | Tổng tiền quy về tiền tệ công ty |
| `total_qty` | NUMERIC | YES | Tổng số lượng đặt mua |
| `total_received_qty` | NUMERIC | YES | Tổng số lượng đã nhận thực tế |
| `amount_total_received` | NUMERIC | YES | Tổng tiền hàng đã nhận theo giá PO |
| `total_received_subtotal` | NUMERIC | YES | Tổng tiền hàng đã nhận (chưa VAT) |
| `total_received_total` | NUMERIC | YES | Tổng tiền hàng đã nhận (có VAT) |
| `net_received_subtotal` | NUMERIC | YES | Giá trị thực nhận ròng (chưa VAT) = nhận − trả |
| `net_received_tax_amount` | NUMERIC | YES | Thuế VAT trên giá trị thực nhận ròng |
| `net_received_total` | NUMERIC | YES | Giá trị thực nhận ròng (có VAT) = nhận − trả |
| `bill_subtotal` | NUMERIC | YES | Tổng tiền hóa đơn chưa VAT |
| `bill_tax_amount` | NUMERIC | YES | Thuế VAT trên hóa đơn |
| `bill_total` | NUMERIC | YES | Tổng tiền hóa đơn có VAT |
| `invoice_status` | STRING | YES | Trạng thái hóa đơn — xem Enum bên dưới |
| `receipt_status` | STRING | YES | Trạng thái nhận hàng — xem Enum bên dưới |
| `invoice_count` | INT64 | YES | Số lượng hóa đơn liên quan |
| `has_order_line` | BOOL | YES | TRUE nếu PO có ít nhất một dòng hàng |
| `is_replenishment` | BOOL | YES | TRUE nếu PO được tạo từ lệnh bổ sung tự động |
| `has_date_planned_different` | BOOL | YES | TRUE nếu các dòng có ngày dự kiến khác nhau |
| `is_vendor_confirm_date_planned` | BOOL | YES | TRUE nếu NCC đã xác nhận ngày giao |
| `trading_terms_fail` | BOOL | YES | TRUE nếu PO vi phạm điều kiện thương mại |
| `mail_reminder_confirmed` | BOOL | YES | TRUE nếu đã gửi email nhắc nhở xác nhận |
| `mail_reception_confirmed` | BOOL | YES | TRUE nếu đã gửi email xác nhận nhận hàng |
| `vendor_matching_status` | STRING | YES | Trạng thái đối chiếu biên bản NCC — xem Enum bên dưới |
| `min_order_qty` | INT64 | YES | Số lượng đặt hàng tối thiểu theo NCC |
| `min_order_value` | NUMERIC | YES | Giá trị đặt hàng tối thiểu theo NCC |
| `original_order_id` | INT64 | YES | FK → `purchase_order.id` — PO gốc nếu là trả hàng |
| `request_vendor_return_id` | INT64 | YES | FK — ID yêu cầu trả hàng NCC gốc |
| `notes` | STRING | YES | Ghi chú nội bộ của PO |
| `note_on_po` | STRING | YES | Ghi chú hiển thị trên PO gửi NCC |
| `log_failed` | STRING | YES | Log lỗi nếu PO xử lý thất bại |
| `failed` | INT64 | YES | Số lần xử lý thất bại |
| `old_name` | STRING | YES | Số PO cũ trước khi đổi tên |
| `input_order_number` | STRING | YES | Số đơn hàng nhập từ bên ngoài |
| `return_reason` | STRING | YES | Lý do trả hàng NCC |
| `create_uid` | INT64 | YES | ID người tạo bản ghi |
| `write_uid` | INT64 | YES | ID người chỉnh sửa lần cuối |
| `create_date` | DATETIME | YES | Thời điểm tạo bản ghi |
| `write_date` | DATETIME | YES | Thời điểm cập nhật lần cuối |

### Enum Values (thực tế từ dữ liệu)

| Cột | Giá trị | Số bản ghi | Ý nghĩa |
|---|---|---|---|
| `state` | `purchase` | 167,246 | PO đã xác nhận, đang thực hiện |
| `state` | `cancel` | 7,193 | PO đã hủy |
| `state` | `done` | 258 | PO đã hoàn thành (locked) |
| `state` | `draft` | 72 | PO nháp chưa gửi |
| `order_type` | `purchase` | 174,655 | Đơn mua hàng thông thường |
| `order_type` | `purchase_return` | 114 | Đơn trả hàng NCC |
| `invoice_status` | `invoiced` | 160,023 | Đã lập hóa đơn đầy đủ |
| `invoice_status` | `no` | 9,726 | Không cần lập hóa đơn |
| `invoice_status` | `to invoice` | 5,020 | Còn cần lập hóa đơn |
| `receipt_status` | `full` | 164,117 | Đã nhận đủ hàng |
| `receipt_status` | `pending` | 397 | Đang chờ nhận hàng |
| `receipt_status` | `partial` | 1 | Nhận một phần |
| `receipt_status` | `NULL` | 10,254 | PO chưa có phiếu nhập (draft/cancel) |
| `vendor_matching_status` | `none` | 174,195 | Chưa đối chiếu biên bản NCC |
| `vendor_matching_status` | `completely_reconciled` | 476 | Đã đối chiếu khớp hoàn toàn |
| `vendor_matching_status` | `tobe_processed` | 98 | Đang chờ xử lý đối chiếu |

### Cột nên dùng để Filter / Index

| Cột | Lý do |
|---|---|
| `state` | Filter PO đang hoạt động (`purchase`, `done`) |
| `partner_id` | Filter theo NCC — dùng trong mọi API liên quan NCC |
| `date_order` | Range filter theo kỳ báo cáo — phổ biến nhất |
| `warehouse_id` | Filter theo kho — quan trọng cho tích hợp WMS |
| `invoice_status` | Filter PO cần lập hóa đơn |
| `receipt_status` | Filter PO cần nhận hàng |
| `vendor_matching_status` | Filter PO cần đối chiếu DCCN |
| `order_type` | Phân biệt mua hàng vs trả hàng |
| `name` | Lookup theo số PO — dùng để join với `stock_picking.origin` |

### Business Rules & Validation

- **PO hợp lệ để xử lý:** `state IN ('purchase', 'done')` — không xử lý `draft` hoặc `cancel`.
- **Giá trị thực nhận:** Dùng `net_received_total` / `net_received_subtotal` cho DCCN, không dùng `amount_total` (là giá trị đặt, chưa trừ trả hàng).
- **Phát hiện trả hàng:** `order_type = 'purchase_return'` hoặc `original_order_id IS NOT NULL`.
- **Đối chiếu NCC:** Chỉ xử lý khi `vendor_matching_status != 'completely_reconciled'` và `state = 'purchase'`.
- **Vi phạm trading terms:** Khi `trading_terms_fail = TRUE`, cần notify team mua hàng trước khi duyệt.
- **Incremental sync:** Dùng `write_date` làm watermark để chỉ pull bản ghi đã thay đổi.

---

## 2. `purchase_order_line`

**Mô tả:** Lưu từng dòng hàng (line item) trong một Purchase Order. Mỗi bản ghi tương ứng với một sản phẩm được đặt mua, bao gồm số lượng, đơn giá, chiết khấu và theo dõi nhận/trả hàng.

**Dataset:** `PRD_Staging` | **Odoo model:** `purchase.order.line`

### Các cột

| Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---|---|---|
| `id` | INT64 | NO | Khóa chính |
| `order_id` | INT64 | YES | FK → `purchase_order.id` — PO cha |
| `name` | STRING | YES | Tên Phiếu Nhập/Trả Hàng |
| `state` | STRING | YES | Trạng thái kế thừa từ PO cha |
| `sequence` | INT64 | YES | Thứ tự hiển thị dòng trong PO |
| `display_type` | STRING | YES | Kiểu hiển thị — xem Enum bên dưới |
| `product_id` | INT64 | YES | FK → `product_product.id` — Sản phẩm variant |
| `categ_id` | INT64 | YES | FK → `product.category` — Danh mục sản phẩm |
| `partner_id` | INT64 | YES | FK → `res_partner.id` — Nhà cung cấp |
| `company_id` | INT64 | YES | FK → `res_company.id` — Công ty |
| `warehouse_id` | INT64 | YES | FK → `stock_warehouse.id` — Kho nhận hàng |
| `currency_id` | INT64 | YES | FK → `res_currency.id` — Đơn vị tiền tệ |
| `product_qty` | NUMERIC | YES | Số lượng đặt mua (theo UoM mua) |
| `product_uom` | INT64 | YES | FK → `uom.uom` — Đơn vị đo lường mua |
| `product_uom_qty` | NUMERIC | YES | Số lượng quy đổi về UoM cơ bản |
| `min_order_qty` | NUMERIC | YES | Số lượng đặt hàng tối thiểu theo NCC |
| `price_unit` | NUMERIC | YES | Đơn giá mua (chưa VAT, chưa chiết khấu) |
| `comp_price_unit` | NUMERIC | YES | Đơn giá quy về tiền tệ công ty |
| `discount` | NUMERIC | YES | Chiết khấu theo dòng (%, ví dụ 5.0 = 5%) |
| `price_subtotal` | NUMERIC | YES | Thành tiền chưa thuế = `price_unit × qty × (1 − discount%)` |
| `price_tax` | NUMERIC | YES | Tiền thuế VAT của dòng |
| `price_total` | NUMERIC | YES | Thành tiền có VAT |
| `comp_price_subtotal` | NUMERIC | YES | Thành tiền chưa thuế quy về tiền tệ công ty |
| `comp_price_tax` | NUMERIC | YES | Thuế VAT quy về tiền tệ công ty |
| `comp_price_total` | NUMERIC | YES | Thành tiền có thuế quy về tiền tệ công ty |
| `qty_received` | NUMERIC | YES | Số lượng đã nhận thực tế |
| `qty_returned` | NUMERIC | YES | Số lượng đã trả lại NCC |
| `qty_invoiced` | NUMERIC | YES | Số lượng đã được lập hóa đơn |
| `qty_to_invoice` | NUMERIC | YES | Số lượng còn cần lập hóa đơn |
| `qty_received_manual` | NUMERIC | YES | Số lượng nhận nhập tay |
| `qty_received_method` | STRING | YES | Phương thức tính qty nhận — xem Enum bên dưới |
| `received_subtotal` | NUMERIC | YES | Giá trị hàng đã nhận (chưa VAT) |
| `received_tax_amount` | NUMERIC | YES | Thuế VAT trên hàng đã nhận |
| `received_total` | NUMERIC | YES | Giá trị hàng đã nhận (có VAT) |
| `received_value` | NUMERIC | YES | Giá trị tồn kho của hàng đã nhận (theo giá vốn) |
| `returned_subtotal` | NUMERIC | YES | Giá trị hàng đã trả NCC (chưa VAT) |
| `returned_tax_amount` | NUMERIC | YES | Thuế VAT trên hàng đã trả |
| `returned_total` | NUMERIC | YES | Giá trị hàng đã trả NCC (có VAT) |
| `net_received_subtotal` | NUMERIC | YES | Giá trị thực nhận ròng chưa VAT = received − returned |
| `net_received_tax_amount` | NUMERIC | YES | Thuế VAT trên giá trị thực nhận ròng |
| `net_received_total` | NUMERIC | YES | Giá trị thực nhận ròng có VAT = received − returned |
| `vendor_product_code` | STRING | YES | Mã sản phẩm theo catalog NCC |
| `vendor_product_name` | STRING | YES | Tên sản phẩm theo catalog NCC |
| `date_order` | DATETIME | YES | Ngày đặt hàng (kế thừa từ PO cha) |
| `date_planned` | DATETIME | YES | Ngày dự kiến nhận hàng cho dòng này |
| `expiration_date` | DATETIME | YES | Ngày hết hạn dự kiến của lô hàng |
| `orderpoint_id` | INT64 | YES | FK — Điểm bổ sung tự động tạo ra dòng này |
| `sale_order_id` | INT64 | YES | FK → `sale_order.id` — SO liên quan |
| `sale_line_id` | INT64 | YES | FK → `sale_order_line.id` — Dòng SO liên quan |
| `product_packaging_id` | INT64 | YES | FK — Quy cách đóng gói |
| `product_packaging_qty` | NUMERIC | YES | Số lượng theo quy cách đóng gói |
| `origin_returned_line_id` | INT64 | YES | FK → `purchase_order_line.id` — Dòng gốc bị trả |
| `request_vendor_return_line_id` | INT64 | YES | FK — Dòng yêu cầu trả hàng NCC |
| `analytic_account_id` | INT64 | YES | FK — Tài khoản phân tích chi phí |
| `analytic_distribution` | STRING | YES | Phân bổ phân tích (JSON `{account_id: percent}`) |
| `dummy_product_uom` | INT64 | YES | UoM tạm khi chưa chọn sản phẩm |
| `reasons` | STRING | YES | Lý do điều chỉnh / trả hàng |
| `product_description_variants` | STRING | YES | Mô tả attribute variant |
| `create_uid` | INT64 | YES | ID người tạo |
| `write_uid` | INT64 | YES | ID người chỉnh sửa lần cuối |
| `create_date` | DATETIME | YES | Thời điểm tạo bản ghi |
| `write_date` | DATETIME | YES | Thời điểm cập nhật lần cuối |

### Enum Values (thực tế từ dữ liệu)

| Cột | Giá trị | Số bản ghi | Ý nghĩa |
|---|---|---|---|
| `state` | `purchase` | 1,329,714 | Dòng đang hoạt động |
| `state` | `cancel` | 21,134 | Dòng đã hủy |
| `state` | `done` | 1,256 | Dòng đã hoàn thành (locked) |
| `state` | `draft` | 253 | Dòng nháp |
| `display_type` | `NULL` | 1,352,199 | **Dòng hàng thật** — có dữ liệu số lượng/giá |
| `display_type` | `line_note` | 152 | Dòng ghi chú — không có giá trị số |
| `display_type` | `line_section` | 6 | Dòng tiêu đề nhóm — không có giá trị số |
| `qty_received_method` | `stock_moves` | 1,346,638 | Số lượng nhận tính từ stock moves (98%) |
| `qty_received_method` | `manual` | 5,561 | Số lượng nhận nhập tay |
| `qty_received_method` | `NULL` | 158 | Dòng section/note không áp dụng |

### Cột nên dùng để Filter / Index

| Cột | Lý do |
|---|---|
| `order_id` | Join chính với `purchase_order` |
| `product_id` | Lookup theo sản phẩm |
| `partner_id` | Filter theo NCC |
| `state` | Loại bỏ dòng cancel/draft |
| `display_type` | **Bắt buộc filter `IS NULL`** để chỉ lấy dòng hàng thật |
| `date_order` | Range filter theo kỳ |
| `warehouse_id` | Filter theo kho |

### Business Rules & Validation

- **Bỏ qua section/note:** Luôn thêm `display_type IS NULL` khi tính toán số lượng hoặc tiền — không có ngoại lệ.
- **Net received là nguồn sự thật cho DCCN:** `net_received_subtotal` và `net_received_total` đã được Odoo tính sẵn, không tính lại thủ công.
- **Kiểm tra dòng trả hàng:** `origin_returned_line_id IS NOT NULL` xác định đây là dòng trả hàng.
- **Validate giá:** `price_subtotal ≈ price_unit × product_qty × (1 − discount/100)` — chênh lệch nhỏ do làm tròn là bình thường.
- **Không dùng `product_uom_qty` cho DCCN:** Dùng `product_qty` (UoM mua) hoặc `qty_received` (số lượng thực nhận).

---

## 3. `res_partner`

**Mô tả:** Bảng trung tâm lưu thông tin đối tác — bao gồm nhà cung cấp (NCC), khách hàng, nhân viên, địa chỉ giao hàng và các pháp nhân. Được tham chiếu bởi hầu hết các bảng nghiệp vụ.

**Dataset:** `PRD_Staging` | **Odoo model:** `res.partner`

### Các cột

| Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---|---|---|
| `id` | INT64 | NO | Khóa chính |
| `name` | STRING | YES | Tên đầy đủ |
| `display_name` | STRING | YES | Tên hiển thị (có thể bao gồm tên công ty cha) |
| `short_name` | STRING | YES | Tên viết tắt / thương mại |
| `company_name` | STRING | YES | Tên công ty khi đối tác là cá nhân |
| `commercial_company_name` | STRING | YES | Tên pháp lý của công ty thương mại |
| `type` | STRING | YES | Loại địa chỉ — xem Enum bên dưới |
| `is_company` | BOOL | YES | TRUE nếu là pháp nhân / công ty |
| `is_vendor` | BOOL | YES | TRUE nếu là nhà cung cấp |
| `is_customer` | BOOL | YES | TRUE nếu là khách hàng |
| `employee` | BOOL | YES | TRUE nếu là nhân viên nội bộ |
| `delivery_person_ok` | BOOL | YES | TRUE nếu là nhân viên giao hàng |
| `active` | BOOL | YES | TRUE nếu đang hoạt động |
| `parent_id` | INT64 | YES | FK → `res_partner.id` — Công ty cha |
| `commercial_partner_id` | INT64 | YES | FK → `res_partner.id` — Đối tác thương mại chính |
| `company_id` | INT64 | YES | FK → `res_company.id` |
| `user_id` | INT64 | YES | FK → `res_users.id` — Nhân viên phụ trách |
| `buyer_id` | INT64 | YES | FK → `res_users.id` — Buyer phụ trách NCC |
| `team_id` | INT64 | YES | FK — Nhóm kinh doanh |
| `industry_id` | INT64 | YES | FK — Ngành nghề |
| `country_id` | INT64 | YES | FK → `res_country.id` — Quốc gia |
| `state_id` | INT64 | YES | FK — Tỉnh / thành phố |
| `district_id` | INT64 | YES | FK — Quận / huyện |
| `ward_id` | INT64 | YES | FK — Phường / xã |
| `street` | STRING | YES | Địa chỉ dòng 1 |
| `street2` | STRING | YES | Địa chỉ dòng 2 |
| `city` | STRING | YES | Thành phố |
| `zip` | STRING | YES | Mã bưu chính |
| `contact_address_complete` | STRING | YES | Địa chỉ đầy đủ đã kết hợp |
| `partner_latitude` | NUMERIC | YES | Vĩ độ |
| `partner_longitude` | NUMERIC | YES | Kinh độ |
| `phone` | STRING | YES | Số điện thoại |
| `mobile` | STRING | YES | Số di động |
| `phone_sanitized` | STRING | YES | Số điện thoại đã chuẩn hóa |
| `email` | STRING | YES | Email chính |
| `email_normalized` | STRING | YES | Email chuẩn hóa (lowercase) |
| `accounting_email` | STRING | YES | Email kế toán nhận hóa đơn |
| `website` | STRING | YES | Website |
| `vat` | STRING | YES | Mã số thuế (MST) |
| `company_registry` | STRING | YES | Số đăng ký kinh doanh |
| `ref` | STRING | YES | Mã nội bộ của đối tác |
| `function` | STRING | YES | Chức danh / vị trí |
| `lang` | STRING | YES | Ngôn ngữ mặc định — xem Enum bên dưới |
| `tz` | STRING | YES | Múi giờ (ví dụ: `Asia/Ho_Chi_Minh`) |
| `gender` | STRING | YES | Giới tính: `male` \| `female` \| `other` |
| `birthday` | DATE | YES | Ngày sinh |
| `date` | DATE | YES | Ngày trở thành khách hàng |
| `store_anniversary` | INT64 | YES | Ngày kỷ niệm mở cửa hàng |
| `customer_rank` | INT64 | YES | Hạng khách hàng |
| `supplier_rank` | INT64 | YES | Hạng nhà cung cấp |
| `customer_note` | STRING | YES | Ghi chú nội bộ về khách hàng |
| `note_on_po` | STRING | YES | Ghi chú hiển thị trên PO gửi NCC |
| `comment` | STRING | YES | Ghi chú tự do |
| `sale_warn` | STRING | YES | Cảnh báo bán hàng — xem Enum bên dưới |
| `sale_warn_msg` | STRING | YES | Nội dung cảnh báo bán hàng |
| `purchase_warn` | STRING | YES | Cảnh báo mua hàng — xem Enum bên dưới |
| `purchase_warn_msg` | STRING | YES | Nội dung cảnh báo mua hàng |
| `invoice_warn` | STRING | YES | Cảnh báo lập hóa đơn |
| `invoice_warn_msg` | STRING | YES | Nội dung cảnh báo hóa đơn |
| `picking_warn` | STRING | YES | Cảnh báo tạo phiếu kho |
| `picking_warn_msg` | STRING | YES | Nội dung cảnh báo phiếu kho |
| `delivery_ok` | BOOL | YES | TRUE nếu NCC có thể giao hàng |
| `delivery_lead_time` | INT64 | YES | Thời gian giao hàng tiêu chuẩn (ngày) |
| `min_order_qty` | INT64 | YES | Số lượng đặt hàng tối thiểu |
| `min_order_value` | NUMERIC | YES | Giá trị đặt hàng tối thiểu |
| `max_price_increase` | NUMERIC | YES | Mức tăng giá tối đa cho phép (%) |
| `price_change_notice` | INT64 | YES | Số ngày báo trước khi tăng giá |
| `unchanged_price_duration` | INT64 | YES | Thời gian tối thiểu không được thay đổi giá (ngày) |
| `cutoff_day` | STRING | YES | Ngày chốt đơn trong tháng |
| `cutoff_time` | STRING | YES | Giờ chốt đơn trong ngày |
| `expir_date_at_delivery` | STRING | YES | Yêu cầu HSD tối thiểu khi giao hàng |
| `no_days_notice_before_expir_date` | INT64 | YES | Số ngày báo trước khi hàng hết hạn |
| `return_before_expir_date` | BOOL | YES | TRUE nếu cho phép trả hàng trước HSD |
| `debit_limit` | NUMERIC | YES | Hạn mức công nợ |
| `mov_currency_id` | INT64 | YES | FK — Tiền tệ cho giá trị đặt hàng tối thiểu |
| `listing_fee` | INT64 | YES | Phí niêm yết sản phẩm |
| `product_listing_fee` | INT64 | YES | Phí niêm yết từng sản phẩm |
| `barcode_printing_fee` | STRING | YES | Phí in barcode |
| `bas_loyalty_card_fee` | STRING | YES | Phí thẻ loyalty BAS |
| `distribution_center_fee` | NUMERIC | YES | Phí dịch vụ trung tâm phân phối |
| `opening_new_stores_support` | INT64 | YES | Hỗ trợ mở cửa hàng mới |
| `support_for_product_wastage` | STRING | YES | Hỗ trợ hàng hao hụt |
| `advertisement_and_promotion_support` | STRING | YES | Hỗ trợ quảng cáo & khuyến mãi |
| `bonus_for_yearly_turnover` | STRING | YES | Thưởng doanh thu cuối năm |
| `conditional_incentive_rebate` | STRING | YES | Chiết khấu incentive có điều kiện |
| `unconditional_incentive_rebate` | STRING | YES | Chiết khấu incentive vô điều kiện |
| `direct_discount_on_invoice` | NUMERIC | YES | Chiết khấu trực tiếp trên hóa đơn (%) |
| `discount_for_payment_on_time` | STRING | YES | Chiết khấu thanh toán đúng hạn |
| `used_in_group_pos_order` | BOOL | YES | TRUE nếu dùng trong đơn POS nhóm |
| `exclude_loyalty` | BOOL | YES | TRUE nếu loại trừ khỏi chương trình tích điểm |
| `auto_invoice` | BOOL | YES | TRUE nếu tự động tạo hóa đơn |
| `partner_share` | BOOL | YES | TRUE nếu là đối tác portal |
| `is_user_manager` | BOOL | YES | TRUE nếu user có quyền manager |
| `change_vendor_info` | INT64 | YES | Số ngày báo trước để đổi thông tin NCC |
| `change_product_info` | INT64 | YES | Số ngày báo trước để đổi thông tin sản phẩm |
| `change_bank_account_info` | INT64 | YES | Số ngày báo trước để đổi tài khoản ngân hàng |
| `it_support` | STRING | YES | Thông tin hỗ trợ IT |
| `delivery_support` | STRING | YES | Thông tin hỗ trợ giao vận |
| `display_support` | STRING | YES | Thông tin hỗ trợ trưng bày |
| `online_partner_information` | STRING | YES | Thông tin đối tác kênh online |
| `followup_reminder_type` | STRING | YES | Kiểu nhắc nhở theo dõi công nợ |
| `last_time_entries_checked` | DATETIME | YES | Lần cuối kiểm tra bút toán |
| `signup_token` | STRING | YES | Token đăng ký portal |
| `signup_type` | STRING | YES | Loại đăng ký portal |
| `signup_expiration` | DATETIME | YES | Thời hạn token đăng ký |
| `ocn_token` | STRING | YES | Token OCN thông báo di động |
| `message_bounce` | INT64 | YES | Số email bị bounce |
| `message_main_attachment_id` | INT64 | YES | FK — File đính kèm chính |
| `create_uid` | INT64 | YES | ID người tạo |
| `write_uid` | INT64 | YES | ID người chỉnh sửa lần cuối |
| `create_date` | DATETIME | YES | Thời điểm tạo bản ghi |
| `write_date` | DATETIME | YES | Thời điểm cập nhật lần cuối |

### Enum Values (thực tế từ dữ liệu)

| Cột | Giá trị | Số bản ghi | Ý nghĩa |
|---|---|---|---|
| `type` | `contact` | 37,756 | Liên hệ chính |
| `type` | `invoice` | 559 | Địa chỉ hóa đơn |
| `type` | `delivery` | 218 | Địa chỉ giao hàng |
| `type` | `other` | 110 | Địa chỉ khác |
| `type` | `private` | 1 | Địa chỉ riêng tư (nhân viên) |
| `lang` | `en_US` | 38,538 | Ngôn ngữ tiếng Anh (mặc định hệ thống) |
| `lang` | `vi_VN` | 106 | Ngôn ngữ tiếng Việt |
| `sale_warn` | `no-message` | 38,644 | Không có cảnh báo (100% data hiện tại) |
| `purchase_warn` | `no-message` | 38,644 | Không có cảnh báo (100% data hiện tại) |

> **Lưu ý:** `warning` và `block` là giá trị hợp lệ theo Odoo nhưng hiện không có bản ghi nào — API vẫn cần xử lý các giá trị này đề phòng phát sinh.

### Cột nên dùng để Filter / Index

| Cột | Lý do |
|---|---|
| `id` | Khóa join chính từ mọi bảng |
| `is_vendor` | Lọc danh sách NCC |
| `is_customer` | Lọc danh sách khách hàng |
| `active` | Luôn filter `active = TRUE` trừ khi cần lịch sử |
| `type` | Filter địa chỉ giao hàng (`delivery`) hoặc hóa đơn (`invoice`) |
| `parent_id` | Lấy tất cả địa chỉ của một công ty |
| `vat` | Lookup theo MST — dùng cho tích hợp kế toán |
| `ref` | Lookup theo mã nội bộ |

### Business Rules & Validation

- **NCC hợp lệ:** `is_vendor = TRUE AND active = TRUE AND is_company = TRUE`.
- **Cấu trúc cha-con:** Luôn resolve `parent_id` để lấy thông tin pháp nhân chính. Địa chỉ giao hàng (`type = 'delivery'`) thuộc về công ty cha.
- **MST unique:** `vat` nên là unique cho NCC — dùng để đối chiếu với hệ thống kế toán / thuế bên ngoài.
- **Trading terms read-only:** Các cột từ `listing_fee` → `discount_for_payment_on_time` lưu điều kiện thương mại ký hợp đồng. Không ghi đè từ hệ thống ngoài nếu không có approval.
- **Cutoff tự động:** Kết hợp `cutoff_day` + `cutoff_time` để tính deadline đặt hàng cho NCC — quan trọng với WMS tự động tạo PO.

---

## 4. `stock_picking`

**Mô tả:** Lưu thông tin các phiếu điều chuyển kho (Transfer). Bao gồm phiếu nhập hàng từ NCC (Receipt), phiếu xuất hàng bán (Delivery), phiếu điều chuyển nội bộ và phiếu trả hàng.

**Dataset:** `PRD_Staging` | **Odoo model:** `stock.picking`

### Các cột

| Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---|---|---|
| `id` | INT64 | NO | Khóa chính |
| `name` | STRING | YES | Số phiếu kho (ví dụ: `WH/IN/00001`) |
| `state` | STRING | YES | Trạng thái — xem Enum bên dưới |
| `origin` | STRING | YES | Tham chiếu nguồn — số PO hoặc SO tạo ra phiếu này |
| `move_type` | STRING | YES | Kiểu di chuyển — xem Enum bên dưới |
| `priority` | STRING | YES | Ưu tiên: `0` = bình thường, `1` = khẩn |
| `partner_id` | INT64 | YES | FK → `res_partner.id` — NCC hoặc khách hàng |
| `company_id` | INT64 | YES | FK → `res_company.id` |
| `warehouse_id` | INT64 | YES | FK → `stock_warehouse.id` |
| `location_id` | INT64 | YES | FK → `stock_location.id` — Vị trí kho nguồn |
| `location_dest_id` | INT64 | YES | FK → `stock_location.id` — Vị trí kho đích |
| `picking_type_id` | INT64 | YES | FK → `stock.picking.type` — Loại phiếu (nhập/xuất/nội bộ) |
| `user_id` | INT64 | YES | FK → `res_users.id` — Người phụ trách |
| `owner_id` | INT64 | YES | FK → `res_partner.id` — Chủ sở hữu hàng hóa |
| `group_id` | INT64 | YES | FK — Nhóm vận chuyển |
| `sale_id` | INT64 | YES | FK → `sale_order.id` — SO liên quan |
| `pos_order_id` | INT64 | YES | FK → `pos_order.id` |
| `pos_session_id` | INT64 | YES | FK → `pos_session.id` |
| `backorder_id` | INT64 | YES | FK → `stock_picking.id` — Phiếu gốc nếu là phiếu còn lại |
| `transfer_id` | INT64 | YES | FK — Phiếu điều chuyển cha |
| `request_warehouse_id` | INT64 | YES | FK — Kho yêu cầu điều chuyển |
| `supplier_warehouse_id` | INT64 | YES | FK — Kho NCC (inter-company) |
| `carrier_id` | INT64 | YES | FK → `delivery.carrier` — Đơn vị vận chuyển |
| `carrier_price` | NUMERIC | YES | Chi phí vận chuyển |
| `carrier_tracking_ref` | STRING | YES | Mã tracking đơn vận chuyển |
| `cost_computation_id` | INT64 | YES | FK — Cách tính chi phí nhập kho |
| `allocation_analytic_account_id` | INT64 | YES | FK — Tài khoản phân tích phân bổ |
| `date` | DATETIME | YES | Ngày lên lịch phiếu |
| `scheduled_date` | DATETIME | YES | Ngày dự kiến thực hiện |
| `date_deadline` | DATETIME | YES | Hạn chót cần hoàn thành |
| `date_done` | DATETIME | YES | Ngày thực tế hoàn thành (validate) |
| `weight` | NUMERIC | YES | Tổng khối lượng hàng hóa (kg) |
| `is_locked` | BOOL | YES | TRUE nếu phiếu đã bị khóa |
| `is_approved` | BOOL | YES | TRUE nếu phiếu đã được duyệt |
| `need_approval` | BOOL | YES | TRUE nếu yêu cầu duyệt trước khi validate |
| `immediate_transfer` | BOOL | YES | TRUE nếu chuyển kho ngay lập tức |
| `printed` | BOOL | YES | TRUE nếu đã in phiếu |
| `has_deadline_issue` | BOOL | YES | TRUE nếu trễ hạn chót |
| `has_changed_scheduled_date` | BOOL | YES | TRUE nếu ngày dự kiến đã bị thay đổi |
| `has_email_confirmation` | BOOL | YES | TRUE nếu đã gửi email xác nhận |
| `no_confirm_email` | BOOL | YES | TRUE nếu bỏ qua email xác nhận |
| `required_signature` | BOOL | YES | TRUE nếu cần chữ ký khi nhận hàng |
| `show_stock_moves` | BOOL | YES | TRUE nếu hiển thị chi tiết stock moves |
| `approved_uid` | INT64 | YES | FK → `res_users.id` — Người duyệt |
| `validate_uid` | INT64 | YES | FK → `res_users.id` — Người xác nhận hoàn thành |
| `responsible` | STRING | YES | Tên người chịu trách nhiệm (text) |
| `note` | STRING | YES | Ghi chú nội bộ |
| `log_failed` | STRING | YES | Log lỗi xử lý |
| `failed` | INT64 | YES | Số lần xử lý thất bại |
| `transfer_picking_count` | INT64 | YES | Số phiếu điều chuyển liên quan |
| `file_name` | STRING | YES | Tên file đính kèm |
| `message_main_attachment_id` | INT64 | YES | FK — File đính kèm chính |
| `create_uid` | INT64 | YES | ID người tạo |
| `write_uid` | INT64 | YES | ID người chỉnh sửa lần cuối |
| `create_date` | DATETIME | YES | Thời điểm tạo bản ghi |
| `write_date` | DATETIME | YES | Thời điểm cập nhật lần cuối |

### Enum Values (thực tế từ dữ liệu)

| Cột | Giá trị | Số bản ghi | Ý nghĩa |
|---|---|---|---|
| `state` | `done` | 1,554,575 | Phiếu đã hoàn thành (99.5%) |
| `state` | `cancel` | 7,372 | Phiếu đã hủy |
| `state` | `assigned` | 589 | Hàng đã sẵn sàng, chờ validate |
| `state` | `confirmed` | 77 | Đã xác nhận, chưa đủ hàng |
| `state` | `draft` | 2 | Phiếu nháp |
| `move_type` | `direct` | 1,560,857 | Xuất/nhập toàn bộ một lần (99.9%) |
| `move_type` | `one` | 1,758 | Xuất/nhập từng dòng riêng lẻ |

### Cột nên dùng để Filter / Index

| Cột | Lý do |
|---|---|
| `state` | Filter phiếu đã hoàn thành (`done`) |
| `origin` | **Join chính với `purchase_order.name`** |
| `picking_type_id` | Phân biệt loại phiếu (nhập/xuất/nội bộ) |
| `partner_id` | Filter theo NCC hoặc khách hàng |
| `warehouse_id` | Filter theo kho |
| `date_done` | Range filter ngày hoàn thành thực tế |
| `scheduled_date` | Range filter ngày dự kiến |
| `backorder_id` | Phát hiện phiếu còn lại |

### Business Rules & Validation

- **Không có FK trực tiếp với PO:** `stock_picking.origin = purchase_order.name` — STRING match, không phải INT FK. Một PO có thể có nhiều phiếu.
- **Phiếu hoàn thành:** Chỉ xử lý `state = 'done'`. `date_done` là thời điểm ghi nhận nhập kho thực tế.
- **Backorder:** `backorder_id IS NOT NULL` → phiếu này là phần còn lại của phiếu bị nhận thiếu. Khi tổng hợp số lượng nhận, cần gom tất cả phiếu cùng `origin`.
- **Phân loại phiếu nhập từ NCC:** Dùng `picking_type_id` (loại `incoming`), không dùng `origin` để phân loại vì origin còn chứa số SO, MRP.

---

## 5. `product_template`

**Mô tả:** Lưu thông tin template (mẫu) sản phẩm với các thuộc tính chung. Các biến thể cụ thể được lưu trong `product_product`.

**Dataset:** `PRD_Staging` | **Odoo model:** `product.template`

### Các cột

| Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---|---|---|
| `id` | INT64 | NO | Khóa chính |
| `name` | STRING | YES | Tên sản phẩm |
| `name_vi` | STRING | YES | Tên tiếng Việt |
| `name_en` | STRING | YES | Tên tiếng Anh |
| `default_code` | STRING | YES | Mã sản phẩm nội bộ (Internal Reference) |
| `active` | BOOL | YES | TRUE nếu đang hoạt động |
| `type` | STRING | YES | Loại sản phẩm — xem Enum bên dưới |
| `detailed_type` | STRING | YES | Loại chi tiết — xem Enum bên dưới |
| `brand` | STRING | YES | Thương hiệu |
| `origin` | STRING | YES | Xuất xứ (text) |
| `country_of_origin` | INT64 | YES | FK → `res_country.id` — Quốc gia xuất xứ |
| `manufacture` | STRING | YES | Nhà sản xuất |
| `certification` | STRING | YES | Chứng nhận chất lượng |
| `categ_id` | INT64 | YES | FK → `product.category.id` — Danh mục đầy đủ |
| `categ_lv1_id` | INT64 | YES | FK — ID danh mục cấp 1 |
| `parent_categ_id` | INT64 | YES | FK — ID danh mục cha trực tiếp |
| `pos_categ_id` | INT64 | YES | FK — Danh mục POS |
| `category_lv1` | STRING | YES | Tên danh mục cấp 1 (denormalized) |
| `category_lv2` | STRING | YES | Tên danh mục cấp 2 (denormalized) |
| `category_lv3` | STRING | YES | Tên danh mục cấp 3 (denormalized) |
| `uom_id` | INT64 | YES | FK → `uom.uom` — Đơn vị đo lường bán |
| `uom_po_id` | INT64 | YES | FK → `uom.uom` — Đơn vị đo lường mua |
| `uom_label` | STRING | YES | Nhãn đơn vị đo lường hiển thị |
| `categ_uom_id` | INT64 | YES | FK — UoM theo danh mục |
| `list_price` | NUMERIC | YES | Giá bán lẻ niêm yết |
| `weight` | NUMERIC | YES | Trọng lượng (kg) |
| `volume` | NUMERIC | YES | Thể tích (m³) |
| `width` | NUMERIC | YES | Chiều rộng (cm) |
| `height` | NUMERIC | YES | Chiều cao (cm) |
| `depth` | NUMERIC | YES | Chiều sâu (cm) |
| `weight_or_volume` | STRING | YES | Trọng lượng tịnh hoặc thể tích thực tế |
| `product_spec` | STRING | YES | Quy cách (ví dụ: 500g, 1L) |
| `storable_temperature` | STRING | YES | Nhiệt độ bảo quản — xem Enum bên dưới |
| `use_time` | INT64 | YES | Hạn sử dụng (ngày) |
| `shelf_life` | INT64 | YES | Hạn lưu kho (ngày) |
| `alert_time` | INT64 | YES | Số ngày cảnh báo trước khi hết hạn |
| `expiration_time` | INT64 | YES | Thời gian hết hạn từ ngày sản xuất (ngày) |
| `removal_time` | INT64 | YES | Số ngày loại bỏ trước khi hết hạn |
| `uom_shelf_life` | STRING | YES | Đơn vị tính hạn sử dụng |
| `use_expiration_date` | BOOL | YES | TRUE nếu theo dõi ngày hết hạn |
| `tracking` | STRING | YES | Theo dõi tồn kho — xem Enum bên dưới |
| `sale_ok` | BOOL | YES | TRUE nếu có thể bán |
| `purchase_ok` | BOOL | YES | TRUE nếu có thể mua |
| `available_in_pos` | BOOL | YES | TRUE nếu bán được trên POS |
| `manufacturing_ok` | BOOL | YES | TRUE nếu có thể sản xuất |
| `replenishment_ok` | BOOL | YES | TRUE nếu bổ sung tự động |
| `replenished_all_warehouses` | BOOL | YES | TRUE nếu bổ sung cho tất cả kho |
| `has_resupply_from_dc` | BOOL | YES | TRUE nếu lấy hàng từ DC |
| `has_resupply_from_ck` | BOOL | YES | TRUE nếu lấy hàng từ CK |
| `is_gift` | BOOL | YES | TRUE nếu là hàng tặng |
| `not_refund_product` | BOOL | YES | TRUE nếu không được hoàn tiền |
| `discount_excluded` | BOOL | YES | TRUE nếu loại trừ khỏi chiết khấu |
| `is_trade_discount` | BOOL | YES | TRUE nếu áp dụng chiết khấu thương mại |
| `is_bom_kit_component` | BOOL | YES | TRUE nếu là thành phần BoM kit |
| `is_manufactured_component` | BOOL | YES | TRUE nếu là thành phần sản xuất |
| `has_manufacturing_uom` | BOOL | YES | TRUE nếu có UoM sản xuất riêng |
| `has_vendor_returnable` | BOOL | YES | TRUE nếu có thể trả NCC |
| `has_configurable_attributes` | BOOL | YES | TRUE nếu có thuộc tính cấu hình |
| `to_weight` | BOOL | YES | TRUE nếu cân theo trọng lượng |
| `hs_code` | STRING | YES | Mã HS Code (hải quan) |
| `qa_code` | STRING | YES | Mã QA kiểm tra chất lượng |
| `caution` | STRING | YES | Lưu ý / cảnh báo sử dụng |
| `description` | STRING | YES | Mô tả nội bộ |
| `description_sale` | STRING | YES | Mô tả trên đơn bán hàng |
| `description_purchase` | STRING | YES | Mô tả trên đơn mua hàng |
| `description_picking` | STRING | YES | Mô tả trên phiếu kho |
| `description_pickingin` | STRING | YES | Mô tả trên phiếu nhập kho |
| `description_pickingout` | STRING | YES | Mô tả trên phiếu xuất kho |
| `ingredients` | STRING | YES | Thành phần nguyên liệu |
| `instruction` | STRING | YES | Hướng dẫn sử dụng |
| `nutrition_fact` | STRING | YES | Thông tin dinh dưỡng |
| `responsible_company` | STRING | YES | Công ty chịu trách nhiệm sản phẩm |
| `cut_off_time` | STRING | YES | Giờ chốt đơn cho sản phẩm |
| `route_domain` | STRING | YES | Domain tuyến đường kho áp dụng |
| `priority` | STRING | YES | Mức độ ưu tiên |
| `sequence` | INT64 | YES | Thứ tự hiển thị |
| `color` | INT64 | YES | Màu nhãn trong Odoo |
| `sale_delay` | NUMERIC | YES | Thời gian xử lý trước khi giao (ngày) |
| `produce_delay` | NUMERIC | YES | Thời gian sản xuất (ngày) |
| `days_to_prepare_mo` | NUMERIC | YES | Số ngày chuẩn bị lệnh sản xuất |
| `wt_transfer_delay` | INT64 | YES | Thời gian điều chuyển kho (ngày) |
| `purchase_method` | STRING | YES | Phương thức nhận PO — xem Enum bên dưới |
| `invoice_policy` | STRING | YES | Chính sách hóa đơn bán — xem Enum bên dưới |
| `expense_policy` | STRING | YES | Chính sách chi phí |
| `service_type` | STRING | YES | Loại dịch vụ |
| `service_tracking` | STRING | YES | Theo dõi dịch vụ |
| `sale_line_warn` | STRING | YES | Cảnh báo trên dòng bán |
| `sale_line_warn_msg` | STRING | YES | Nội dung cảnh báo dòng bán |
| `purchase_line_warn` | STRING | YES | Cảnh báo trên dòng mua |
| `purchase_line_warn_msg` | STRING | YES | Nội dung cảnh báo dòng mua |
| `remain_days_noti_vendor` | STRING | YES | Số ngày cảnh báo NCC khi hàng sắp hết hạn |
| `x_label_layout` | STRING | YES | Layout nhãn in |
| `company_id` | INT64 | YES | FK → `res_company.id` |
| `planning_enabled` | BOOL | YES | TRUE nếu tích hợp module kế hoạch |
| `planning_role_id` | INT64 | YES | FK — Vai trò trong kế hoạch |
| `production_acc_id` | INT64 | YES | FK — Tài khoản kế toán sản xuất |
| `property_sales_returns_account_id` | INT64 | YES | FK — Tài khoản doanh thu trả hàng |
| `property_stock_valuation_account_id` | INT64 | YES | FK — Tài khoản định giá tồn kho |
| `message_main_attachment_id` | INT64 | YES | FK — File đính kèm chính |
| `create_uid` | INT64 | YES | ID người tạo |
| `write_uid` | INT64 | YES | ID người chỉnh sửa lần cuối |
| `create_date` | DATETIME | YES | Thời điểm tạo bản ghi |
| `write_date` | DATETIME | YES | Thời điểm cập nhật lần cuối |

### Enum Values (thực tế từ dữ liệu)

| Cột | Giá trị | Số bản ghi | Ý nghĩa |
|---|---|---|---|
| `type` / `detailed_type` | `product` | 5,074 | Sản phẩm lưu kho (storable) — 83% |
| `type` / `detailed_type` | `consu` | 604 | Hàng tiêu hao (consumable) |
| `type` / `detailed_type` | `service` | 441 | Dịch vụ — không có tồn kho |
| `tracking` | `none` | 6,119 | Không theo dõi lô/serial (100% hiện tại) |
| `purchase_method` | `receive` | 5,702 | Ghi nhận PO khi nhận hàng (94%) |
| `purchase_method` | `purchase` | 417 | Ghi nhận PO khi xác nhận đơn |
| `invoice_policy` | `delivery` | 5,343 | Lập hóa đơn theo số lượng giao thực tế (87%) |
| `invoice_policy` | `order` | 776 | Lập hóa đơn theo số lượng đặt hàng |
| `storable_temperature` | `25°C` (+ biến thể) | ~2,650 | Hàng khô — nhiệt độ phòng |
| `storable_temperature` | `0-10°C` (+ biến thể) | ~1,450 | Hàng mát — tủ lạnh |
| `storable_temperature` | `≤-18°C` (+ biến thể) | ~255 | Hàng đông lạnh |
| `storable_temperature` | `NULL` | 1,326 | Chưa khai báo |

> **⚠️ `storable_temperature` không chuẩn hóa:** `25°C`, `25C`, `Nhiệt độ phòng`, `Nhiệt độ thường` đều cùng nghĩa. API nên map về 3 nhóm chuẩn: `ambient` | `chilled` | `frozen`.

### Cột nên dùng để Filter / Index

| Cột | Lý do |
|---|---|
| `id` | Join từ `product_product.product_tmpl_id` |
| `active` | Luôn filter `active = TRUE` |
| `default_code` | Lookup theo mã sản phẩm nội bộ |
| `detailed_type` | Filter sản phẩm lưu kho (`product`) |
| `purchase_ok` | Filter sản phẩm có thể mua |
| `category_lv1/2/3` | Filter theo danh mục — đã denormalized |
| `storable_temperature` | Filter theo nhóm nhiệt độ |

### Business Rules & Validation

- **Sản phẩm có thể mua:** `purchase_ok = TRUE AND active = TRUE AND detailed_type = 'product'`.
- **Normalize nhiệt độ khi expose API:** Map `storable_temperature` → `ambient` (25°C, nhiệt độ phòng), `chilled` (0–10°C, 0–5°C, 0–4°C), `frozen` (≤−18°C).
- **Danh mục đã denormalized:** Dùng `category_lv1/2/3` trực tiếp, không cần join thêm bảng danh mục.
- **Sản phẩm dịch vụ:** `type = 'service'` không có tồn kho — loại bỏ khi query liên quan kho.

---

## 6. `product_product`

**Mô tả:** Lưu từng biến thể (variant) cụ thể của sản phẩm. Đây là bảng được tham chiếu trực tiếp trong tất cả giao dịch (PO line, SO line, stock move).

**Dataset:** `PRD_Staging` | **Odoo model:** `product.product`

### Các cột

| Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---|---|---|
| `id` | INT64 | NO | Khóa chính — **ID dùng trong mọi giao dịch** |
| `product_tmpl_id` | INT64 | YES | FK → `product_template.id` — Template cha |
| `active` | BOOL | YES | TRUE nếu variant đang hoạt động |
| `default_code` | STRING | YES | Mã SKU của variant |
| `barcode` | STRING | YES | Mã barcode (EAN13 hoặc tùy chỉnh) |
| `combination_indices` | STRING | YES | Chuỗi index các thuộc tính tạo nên variant |
| `categ_lv1_id` | INT64 | YES | FK — ID danh mục cấp 1 (kế thừa từ template) |
| `category_lv1` | STRING | YES | Tên danh mục cấp 1 (denormalized) |
| `category_lv2` | STRING | YES | Tên danh mục cấp 2 (denormalized) |
| `category_lv3` | STRING | YES | Tên danh mục cấp 3 (denormalized) |
| `volume` | NUMERIC | YES | Thể tích variant (m³) |
| `weight` | NUMERIC | YES | Trọng lượng variant (kg) |
| `can_image_variant_1024_be_zoomed` | BOOL | YES | TRUE nếu ảnh đủ độ phân giải để zoom |
| `message_main_attachment_id` | INT64 | YES | FK — File đính kèm chính |
| `create_uid` | INT64 | YES | ID người tạo |
| `write_uid` | INT64 | YES | ID người chỉnh sửa lần cuối |
| `create_date` | DATETIME | YES | Thời điểm tạo bản ghi |
| `write_date` | DATETIME | YES | Thời điểm cập nhật lần cuối |

### Cột nên dùng để Filter / Index

| Cột | Lý do |
|---|---|
| `id` | Khóa join từ `purchase_order_line.product_id` và mọi giao dịch |
| `product_tmpl_id` | Join với `product_template` |
| `barcode` | **Lookup ưu tiên** — scan từ WMS, POS, thiết bị di động |
| `default_code` | Lookup theo SKU nội bộ |
| `active` | Luôn filter `active = TRUE` |

### Business Rules & Validation

- **`product_product.id` là khóa giao dịch:** Không bao giờ dùng `product_template.id` trong các bảng giao dịch.
- **Barcode là unique:** Validate unique khi nhận data từ WMS.
- **Sản phẩm không có variant:** Khi template không có thuộc tính cấu hình, `combination_indices` rỗng và chỉ có đúng 1 bản ghi `product_product`.
- **Thông tin đầy đủ:** Luôn JOIN với `product_template` để lấy tên, giá, danh mục, nhiệt độ, v.v.

---

## 7. Quan Hệ Giữa Các Bảng

```
res_partner (id)
    ← purchase_order.partner_id          [NCC của PO]
    ← purchase_order_line.partner_id     [NCC của dòng PO]
    ← stock_picking.partner_id           [Đối tác phiếu kho]

product_template (id)
    ← product_product.product_tmpl_id   [1 template → N variants]

product_product (id)
    ← purchase_order_line.product_id    [Sản phẩm đặt mua]

purchase_order (id, name)
    ← purchase_order_line.order_id      [1 PO → N dòng hàng]
    ← purchase_order.original_order_id  [PO gốc của trả hàng]
    ~ stock_picking.origin              [STRING match — không có FK]

purchase_order_line (id)
    ← purchase_order_line.origin_returned_line_id  [Dòng gốc bị trả]
```


## 8. Hướng Dẫn Tích Hợp Hệ Thống Ngoài

### Chiến lược Sync Data (Odoo → FastAPI)

| Bảng | Tần suất pull | Watermark field | Filter khuyến nghị |
|---|---|---|---|
| `purchase_order` | 15–30 phút | `write_date` | `state IN ('purchase', 'done')` |
| `purchase_order_line` | 15–30 phút | `write_date` | `display_type IS NULL` |
| `res_partner` | 1 giờ | `write_date` | `active = TRUE` |
| `stock_picking` | 15–30 phút | `write_date` | `state = 'done'` |
| `product_template` | 1 giờ | `write_date` | `active = TRUE` |
| `product_product` | 1 giờ | `write_date` | `active = TRUE` |

### Xử lý Timezone

Tất cả trường `DATETIME` trong Odoo **không có timezone** — dữ liệu được lưu theo **UTC+7 (Asia/Ho_Chi_Minh)**. Khi expose API:
- Trả về ISO 8601 có timezone: `2024-01-15T14:30:00+07:00`
- Không convert về UTC để tránh nhầm lẫn với hệ thống ngoài

### Mapping cho WMS

| Thông tin WMS cần | Lấy từ |
|---|---|
| PO header | `purchase_order` |
| Danh sách hàng của PO | `purchase_order_line` — filter `display_type IS NULL` |
| Thông tin NCC | `res_partner` filter `is_vendor = TRUE` |
| Phiếu nhập kho | `stock_picking` filter `origin = po.name` AND `state = 'done'` |
| Thông tin sản phẩm | `product_product` JOIN `product_template` |
| Barcode lookup | `product_product.barcode` |
| Nhiệt độ bảo quản | `product_template.storable_temperature` (cần normalize) |

### Mapping cho ERP / Kế Toán

| Thông tin ERP cần | Lấy từ |
|---|---|
| Số PO | `purchase_order.name` |
| NCC + MST | `res_partner.name` + `res_partner.vat` |
| Giá trị PO gốc | `amount_untaxed` + `amount_tax` + `amount_total` |
| Giá trị thực nhận (DCCN) | `net_received_subtotal` + `net_received_tax_amount` + `net_received_total` |
| Trạng thái hóa đơn | `purchase_order.invoice_status` |
| Chi tiết từng sản phẩm | `purchase_order_line.net_received_subtotal` / `net_received_total` |
| Mã sản phẩm NCC | `purchase_order_line.vendor_product_code` |

### Checklist cho Developer

1. **Không có FK `stock_picking` → `purchase_order`** — JOIN qua `stock_picking.origin = purchase_order.name` (STRING).
2. **Luôn filter `display_type IS NULL`** khi query `purchase_order_line`.
3. **Normalize `storable_temperature`** trước khi expose ra API ngoài.
4. **Một PO có thể có nhiều `stock_picking`** — GROUP BY khi tổng hợp số lượng nhận.
5. **Dùng `net_received_*`** không phải `amount_*` khi cần giá trị thực tế sau trả hàng.
6. **Dùng `write_date` làm watermark** cho incremental sync.
7. **`product_product.id`** là khóa giao dịch — không nhầm với `product_template.id`.
8. **Tất cả DATETIME là UTC+7** — thêm offset `+07:00` khi trả về API.

---

*Phiên bản 1.1.0 — Bổ sung enum values thực tế, cột index/filter, business rules và hướng dẫn tích hợp. Cập nhật lại khi có thay đổi schema hoặc dữ liệu.*