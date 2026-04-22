# Workflow: XML to Odoo Vendor Bills

> **Workflow ID:** `556wjyTaOwmybPul`
> **Tên:** `[DUC] XML to Odoo Vendor Bills`
> **Trạng thái:** `inactive` (chưa kích hoạt)
> **Số nodes:** 47
> **Version:** V8 (Apr 2026)
> **Project:** Trang Nguyen <trang.nat@3sach.vn>

---

## 1. Mục đích

Tự động hoá quy trình tạo **Vendor Bill** trên Odoo ERP từ file **XML hoá đơn điện tử** của NCC đặt trên SharePoint.

### Input
- Folder SharePoint: `/sites/accounting.teams/Shared Documents/03. VENDOR BILLS/Input/`
- File `.xml` hoá đơn điện tử (theo chuẩn VN)

### Output
- Vendor Bill được tạo / cập nhật / confirm trên Odoo ERP (`erpdev.3sach.vn`)
- File XML/PDF được move sang:
  - `Output/{ngày HĐ}/` nếu thành công
  - `Failed/` nếu lỗi
- **1 email tổng hợp** báo cáo kết quả gửi về `duc.ngo@parainsight.com`

### Trigger
- **Schedule Trigger:** chạy tự động mỗi ngày lúc **23:00** (giờ VN).

---

## 2. Tổng quan 4 Phase

| Phase | Tên | Mục đích |
|-------|-----|----------|
| 1 | Parse XML → Extract PO | Đọc file, bóc dữ liệu hoá đơn, tìm số PO |
| 2 | Tìm Partner → Tìm PO | Match dữ liệu với Odoo (NCC + PO) |
| 3 | Xử lý Bill | Tạo / update / confirm Vendor Bill |
| 4 | Email Tổng Hợp | Move file + gửi 1 email tổng hợp (tạm thời chưa thực hiện ) |

---

## 3. Phase 1 — Parse XML + Extract PO

**Mục tiêu:** Lấy danh sách XML từ SharePoint và trích xuất dữ liệu hoá đơn.
**Sắp tới sẽ có thay đổi vì dùng nguồn dữ liệu từ Matbao**

### Các node

| # | Node | Loại | Mô tả |
|---|------|------|-------|
| 1 | **Schedule Trigger** | `scheduleTrigger` | Kích hoạt lúc 23:00 hằng ngày |
| 2 | **List Item in a Folder on SharePoint** | `httpRequest` | Gọi REST API SharePoint list files trong folder `Input` |
| 3 | **Split Out** | `splitOut` | Tách mảng `value` thành từng item (mỗi file 1 item) |
| 4 | **If: Is XML?** | `if` | Lọc chỉ giữ file `.xml`; nhánh FALSE dẫn tới `Download PDF` (đang disabled) |
| 5 | **Download XML** | `httpRequest` | Tải binary XML từ SharePoint |
| 6 | **Extract From File** | `extractFromFile` | Convert binary → text UTF-8 (`xmlText`) |
| 7 | **XML** | `xml` | Parse text XML → JSON có cấu trúc |
| 8 | **Code: Parse XML + Extract PO** | `code` | Trích xuất các trường hoá đơn |



### Dữ liệu được extract

Từ XML parse ra các field:
- **Thông tin hoá đơn:** `ky_hieu`, `so_hoa_don`, `mau_so` (từ `KHMSHDon`), `ngay_lap`
- **Thông tin NCC:** `mst_ncc`, `ten_ncc`
- **Thông tin tiền:** `tong_tien`, `tien_chua_thue`, `tien_thue`
- **Chi tiết hàng:** `invoice_lines[]` (stt, mã SP, tên SP, ĐVT, SL, đơn giá, thành tiền, thuế suất, chiết khấu)
- **Số PO:** parse bằng regex `/PO\s*\d{6,}/gi` → `po_number`, `po_all[]`, `has_po`

---

## 4. Phase 2 — Tìm Partner → Tìm PO

**Mục tiêu:** Match dữ liệu XML với Odoo (NCC và PO tương ứng).

1. Tìm Partner của Invoice dựa theo mã số thuế
   1. không tìm được thì : ...............
   2. Nếu tìm được thì tiếp tục xử lý tiếp
2. Tìm PO của partner này, dựa trên Partner ID và **PO NUMBER** được trích xuất từ Invoice XML
3. Nếu không có **PO Number** thì dùng Partner ID và giá trị PO = giá trị XML Invoice (Field nào ???)




### Các node

| # | Node | Loại | Mô tả |
|---|------|------|-------|
| 9 | **Get Partner** | `odoo` | Search `res.partner` theo `vat = mst_ncc` → lấy `partner_id` |
| 10 | **If: Tìm thấy Partner?** | `if` | Check có `id` không |
| 11 | **IF: has_po?** | `if` | Check XML có chứa số PO không |
| 12a | **Odoo: Search PO by number** | `odoo` | Search `purchase.order` theo `name` + `partner_id` + `invoice_status != 'no'` |
| 12b | **Odoo: Search PO by amount & date** | `odoo` | Search `purchase.order` theo `partner_id` + `net_received_subtotal` + `effective_date` |
| 13 | **Merge PO Results** | `merge` | Gộp kết quả 2 nhánh search PO |
| 14 | **Code: Prep Data** | `code` | Chuẩn hoá data cho Phase 3 |

### Logic phân nhánh

- **Không tìm thấy Partner** → `email_type = no_partner` → đi thẳng vào LOG
- **Tìm thấy Partner + có PO trong XML** → search PO theo số
- **Tìm thấy Partner + không có PO** → search PO theo tiền + ngày (fallback)
- **Sau Prep Data:**
  - Không tìm thấy PO → `email_type = no_po` → Path B (tạo bill nháp)
  - Tìm thấy PO → Path A (xử lý bill đầy đủ)

---

## 5. Phase 3 — Xử lý Bill (V6 Simplified)

**Mục tiêu:** Tạo / update / confirm Vendor Bill trên Odoo.

### Phân nhánh chính

**If: Tìm thấy PO?**
- **TRUE** → Path A (có PO)
  - Chia thành 2 nhánh con nhánh có Invoice và nhánh không có Invoice. 
- **FALSE** → Path B (không PO)

### Path A — Có PO




| # | Node | Loại | Mô tả |
|---|------|------|-------|
| 15 | **If: Có Hoá Đơn?** | `if` | PO này đã có bill chưa? (check `has_bill`) |
| 16a | **Odoo: Get Bill State** | `odoo` | Lấy state bill hiện có |
| 17a | **If: state != 'posted'?** | `if` | Nếu đã posted → skip (tag `posted_ok`); nếu chưa → update tiếp |
| 16b | **HTTP: Tạo Vendor Bill** | `httpRequest` | Gọi `purchase.order.action_create_invoice` |
| 17b | **Odoo: Reload PO** | `odoo` | Load lại PO để lấy `invoice_ids` mới |
| 18 | **Set: Bill ID (PO)** | `set` | Gán `bill_id = invoice_ids[0]`, `email_type = ''` |
| 19 | **Odoo: Update account.move** | `odoo` | Update `invoice_date`, `payment_reference`, `ref` |
| 20 | **Odoo: Search Tax IDs** | `odoo` | Query `account.invoice.tax WHERE invoice_id = bill_id` |
| 21 | **Loop Over Tax IDs** | `splitInBatches` | Loop từng tax line |
| 22 | **Odoo: Update Tax Line** | `odoo` | Update `invoice_reference`, `invoice_number`, `date_invoice` cho từng dòng thuế |
| 23 | **Odoo: Get Bill Amount** | `odoo` | Lấy `amount_total` thực tế trên Odoo (sau khi update) |
| 24 | **If: Tiền Khớp?** | `if` | So sánh `amount_total` vs `tong_tien`, tolerance ±1000đ |
| 25a | **HTTP: Confirm Bill** | `httpRequest` | Gọi `account.move.action_post` → state = `posted` |

### Path B — Không PO

| # | Node | Loại | Mô tả |
|---|------|------|-------|
| 15' | **HTTP: Tao Bill (No PO)** | `httpRequest` | Gọi raw JSON-RPC `account.move.create` với partner + metadata |
| 16' | **Set Email Type for No_PO** | `set` | Gán `bill_id = result`, `email_type = 'no_po'` |

Bill từ Path B **không được confirm**, chỉ để lại state `draft` cho kế toán xử lý thủ công.

### Các điểm exit vào Merge

| Exit | Điều kiện | Tag `email_type` | Merge input |
|------|-----------|------------------|-------------|
| 1 | Không tìm thấy Partner | `no_partner` | 0 |
| 2 | Bill đã `posted` từ trước | `posted_ok` | 1 |
| 3 | Path B (không PO) / Sau Confirm Bill | `no_po` hoặc `posted_ok` | 2 |
| 4 | Tiền không khớp | `amount_mismatch` | 3 |

---

## 6. Phase 4 — Email Tổng Hợp (V7 Fixed)

**Mục tiêu:** Gom tất cả exit paths → move file → gửi **1 email duy nhất / lần chạy**.

### Các node

| # | Node | Loại | Mô tả |
|---|------|------|-------|
| 26 | **Merge: Gộp All Branches** | `merge` v3.2 | 4 inputs, mode `append` — gom tất cả exit paths |
| 27 | **Prepare Email Content** | `code` | Per-item: tag `email_type` + build email data |
| 28 | **Aggregate: Gộp Items** | `aggregate` | Gộp tất cả items thành 1 array `items[]` |
| 29 | **Code: Build Summary Email** | `code` | Dedup + build HTML email theme 3SACH |
| 30 | **Send Email (Động)** | `emailSend` | Gửi qua AWS SES |

### Fan-out từ Prepare Email Content

**Nhánh 1 — File operations:**
- `Tạo Folder Output theo Ngày HĐ` → `Move XML to Output` → `Move PDF to Output`
- Nếu `Move XML to Output` fail → `HTTP: Move XML to Failed` → `HTTP: Move PDF to Failed`

**Nhánh 2 — Email:**
- `Aggregate: Gộp Items` → `Code: Build Summary Email` → `Send Email (Động)`

### Cấu trúc email

- **Subject:** `[AP Invoice] Ngày DD/MM/YYYY — Tổng file X, Cần xử lý Y`
- **Theme màu 3SACH:** Primary Green `#1A7567`, đỏ `#DC2626` (error), cam `#EA5514` (warning)
- **Font:** Arial (vì email client)
- **5 nhóm status** (mỗi nhóm có icon, màu, bảng chi tiết):
  - ❌ `no_partner` — NCC chưa có trên Odoo
  - ⚠️ `no_po` — Không tìm thấy PO
  - ⚠️ `amount_mismatch` — Tiền không khớp
  - ❌ `confirm_error` — Lỗi confirm Bill
  - ✅ `posted_ok` — Đã posted thành công

### V7 Fix quan trọng

- Đổi `HTTP: Confirm Bill` từ `onError: continueErrorOutput` → `continueRegularOutput`
- Upgrade Merge `typeVersion 3 → 3.2` (cải thiện async buffering)
- Giảm Merge từ 5 inputs → 4 inputs (bỏ nhánh async rỗng)
- Kết quả: Merge buffer đúng → **chính xác 1 email/lần chạy**

---

## 7. Odoo Models & Relationships

### Các model được sử dụng

| Model | Table | Mục đích | Field quan trọng |
|-------|-------|----------|-------------------|
| `res.partner` | `res_partner` | NCC / Đối tác | `id`, `name`, `vat` (MST) |
| `purchase.order` | `purchase_order` | Đơn đặt hàng | `id`, `name`, `partner_id`, `invoice_status`, `effective_date`, `net_received_subtotal` |
| `account.move` | `account_move` | Vendor Bill | `id`, `state`, `move_type`, `invoice_date`, `ref`, `payment_reference`, `amount_total` |
| `account.invoice.tax` | `account_invoice_tax` | Dòng thuế (custom VN) | `id`, `invoice_id`, `invoice_reference`, `invoice_number`, `date_invoice` |
| `ir.attachment` | `ir_attachment` | File đính kèm polymorphic | `res_model`, `res_id`, `datas`, `mimetype` |

### Sơ đồ quan hệ

```
res.partner ──┬─→ purchase.order (partner_id)
              └─→ account.move (partner_id)

purchase.order ──M:N──→ account.move (via account_move_purchase_order_rel)

account.move ──┬─→ account.move.line (move_id)
               ├─→ account.invoice.tax (invoice_id)
               └─→ ir.attachment (polymorphic: res_model='account.move', res_id=id)
```

---

## 8. Lịch sử version

### V4 (Apr 2026)
- XML luôn attach + PDF conditional
- Dead-letter folder `/Failed/`
- Retry 3x/2s cho all HTTP/Odoo
- Regex PO match cả `PO 250704657`
- Extract `mau_so` từ `KHMSHDon`
- Fixed PDF attach pair-index bug
- Email gộp 1 email tổng hợp/lần chạy
- Thêm status `posted_ok`

### V5 (Apr 2026)
- Fix multi-email — Merge 5 inputs trước Prepare Email
- Email gửi trực tiếp từ Prepare Email (fan-out)

### V6 (Apr 2026)
- Add Aggregate node (backup safety net)
- Merge explicit `mode: append`
- Split `Set: Bill ID` → PO / No PO
- Remove `If: Có PO data?` (redundant)
- Replace Odoo Tax nodes → HTTP batch write
- Dedup trong Build Summary Email
- Remove orphan `Merge: Gộp Paths Email`

### V7 (Apr 2026) — FIXED MULTI-EMAIL BUG
- Đổi `HTTP: Confirm Bill` `onError: continueErrorOutput` → `continueRegularOutput`
- Bỏ Confirm Bill error branch khỏi Merge (5 → 4 inputs)
- Upgrade Merge typeVersion 3 → 3.2
- If: Tiền Khớp? FALSE chuyển từ Merge[4] → Merge[3]

**Root cause V6:** Merge input 3 (Confirm Bill error branch) thường 0 items → Merge không wait all inputs → emit multiple batches → 2 emails. V7 bỏ nhánh async rỗng + bump typeVersion → Merge buffer đúng → 1 email.

### V8 (Apr 2026) — Refactor cho dễ debug
- Đức deactivate 1 số HTTP request nodes, thay bằng Odoo native node + các node N8N
- Lý do: Claude over-engineering → khó debug
- Nguyên tắc: Số nodes nhiều ít không quan trọng, ít node mà khó debug thì cũng không giải quyết được vấn đề

---

## 9. ⚠️ Điểm cần lưu ý (trạng thái hiện tại)

Workflow đang **inactive** (`active: false`), có nhiều node bị **disabled** — đang trong quá trình refactor V7 → V8.

### Nodes đang disabled

| Node | Hệ quả nếu không kích hoạt |
|------|----------------------------|
| `Merge: Gộp All Branches` | 4 nhánh exit KHÔNG gom được → email không chạy đúng |
| `Code: Encode + Email` | Không encode XML/PDF sang base64 |
| `HTTP: Attach XML` | XML không được attach vào bill trên Odoo |
| `HTTP: Attach PDF` | PDF không được attach |
| `If: Có PDF?` | Logic check PDF bị ngắt |
| `HTTP: Download PDF`, `Download PDF` | Không tải PDF từ SharePoint |
| `Tạo Folder Output theo Ngày HĐ` | Folder Output theo ngày không được tạo |
| `Move XML to Output`, `Move PDF to Output` | File không move sang Output sau khi xử lý |
| `HTTP: Move XML to Failed`, `HTTP: Move PDF to Failed` | File lỗi không move sang Failed |
| `Kiểm tra các files đã uploaded có tên = V_` | (orphan, không có input) |
| `IF PDF/XML đã upload (trùng tên => Skip)` | (orphan, không có input) |

### TODO / điểm cần kiểm tra

- [ ] Anti-duplicate: workflow hiện chưa check `account.move.ref` trước khi tạo bill → rủi ro duplicate
- [ ] `move_type` filter: khi check `has_bill` qua `invoice_ids`, chưa filter `move_type = in_invoice` → có thể match nhầm
- [ ] Multi-PO: nếu XML có nhiều PO numbers, workflow chỉ dùng `po_all[0]` → bỏ qua các PO còn lại
- [ ] Kích hoạt lại chuỗi node Email + File move để end-to-end hoàn chỉnh
- [ ] Test lại luồng sau khi đổi HTTP node → Odoo native node (V8)

---

## 10. Credentials sử dụng

| Credential | Loại | Dùng cho |
|------------|------|----------|
| **Accounting SharePoint** | `microsoftSharePointOAuth2Api` | Download/Upload/Move file trên SharePoint |
| **Odoo erpdev (TEST)** | `odooApi` | Odoo native nodes (search, get, create, update) |
| **AWS SES** | `smtp` | Send Email |
| (raw JSON-RPC) | hardcoded API key | `HTTP: Tạo Vendor Bill`, `HTTP: Tao Bill (No PO)`, `HTTP: Confirm Bill` |

> ⚠️ API key `8ff2d46d7e480333cecf88ca7915b1b81a41c031` đang hardcoded trong body các HTTP node — nên move sang n8n credentials.

---

_File này là bản mô tả workflow để reference & chỉnh sửa. Update khi workflow thay đổi._
