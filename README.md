# Vendor Portal — Project Roadmap v4
> Specs & Approach Document — no implementation code

---

## Project Summary

An independent bilingual (Vietnamese + English) web portal serving two types of users: **vendors** and **portal admins**. Vendors log in using their Odoo Vendor ID, view their confirmed Purchase Orders, update actual delivered quantities on receipts across multiple sessions, draw a signature to confirm delivery, and download a signed delivery PDF. Portal admins share the same interface but have additional access to view all vendors, all POs and receipts across the system, trigger the Odoo sync manually, and download any vendor's signed PDF. Admins can also unlock signed receipts directly in the portal. The portal runs on a separate VM from Odoo and integrates exclusively via Odoo's XML-RPC API using a dedicated service account. Internal team and vendor both receive email notifications on validation. All email is delivered via AWS SES.

---

## Confirmed Decisions

| Concern | Decision |
|---|---|
| Odoo version | 16 Community Edition |
| Odoo API protocol | XML-RPC (Python `xmlrpc.client`) |
| Vendor login identifier | `res.partner.id` (integer, assigned by Odoo) |
| Vendor password | Portal-owned, stored in portal PostgreSQL only |
| Profile data source | One-way sync from Odoo `res.partner` (read only) |
| Account provisioning | Auto from Odoo partners where `supplier_rank > 0` |
| Portal language | Vietnamese + English (bilingual, user-switchable) |
| HTTP client (frontend) | Native `fetch` API — no axios or third-party HTTP lib |
| PO statuses visible | Sent RFQ (`sent`), Confirmed (`purchase`), Cancelled (`cancel`), Done (`done`) — Draft (`draft`) is not shown |
| Receipt statuses visible | Ready (`assigned`) and Done (`done`) only |
| qty_done updates | Multiple updates allowed before validation |
| Backorder handling | Vendor submits qty only — admin validates and decides in Odoo |
| Post-validation locking | Receipt locked after vendor signs — portal admin can unlock directly in the portal |
| Admin role | Separate `admin_users` table, password set via env variable initially |
| Admin capabilities | View all vendors, trigger sync, view all POs/receipts, unlock receipts, download any PDF |
| Admin UI | Same layout as vendor portal with additional admin menu items |
| PDF content | Odoo delivery slip + vendor-signed confirmation page |
| Signature capture | `signature_pad.js` → PNG sent to backend |
| PO list search | Filter by PO number and date range |
| Vendor dashboard summary | Total POs, pending receipts, signed receipts shown above PO list |
| Vendor comment on submission | Free-text note added per receipt when submitting qty_done |
| PDF retention | No expiry — stored permanently on server |
| Responsive design | Works on both desktop and mobile equally |
| Vendor accounts | One account per Odoo partner — no multi-user per company |
| Profile changes | All profile changes must go through Odoo — admin cannot edit in portal |
| Admin language | Bilingual (Vietnamese + English) — same as vendor portal |
| SSL / TLS | Handled at infrastructure level (load balancer / reverse proxy) |
| Vendor PO confirmation | Vendor confirms Sent RFQ via portal → portal calls `button_confirm` on `purchase.order` in Odoo |
| Audit logging | Key actions logged in DB: login, po_confirm, qty update, sign, unlock |
| Email notifications | Invite, password reset, validation confirmation (vendor), validation alert (internal team) |
| Email service | AWS SES |
| Frontend stack | React + Vite |
| Backend stack | FastAPI (Python) |
| Database | PostgreSQL (portal-owned) |
| Cache / rate limiting | Redis |
| Deployment | Docker Compose on a separate VM, Nginx reverse proxy |

---

## Architecture Overview

```
Vendor Browser
     │  HTTPS
     ▼
Nginx (reverse proxy + TLS termination)
     ├── /        → React SPA (frontend container)
     └── /api/    → FastAPI (backend container)
                        │
                        ├── PostgreSQL  (vendor accounts, tokens, signatures)
                        ├── Redis       (rate limiting, token blacklist)
                        ├── AWS SES     (all outbound email)
                        │
                        └── Odoo 16 CE (separate VM)
                              XML-RPC : data read/write
                              HTTP session : PDF download
```

**Key principle:** The React frontend never contacts Odoo. All Odoo communication is proxied through the FastAPI backend using a single service account. Vendor credentials never leave the portal's own database.

---

## Data Flow Summary

### Authentication flow
1. Vendor enters their Vendor ID and password on the login page
2. Backend looks up `vendor_users` by `odoo_partner_id`
3. Password verified with bcrypt — same error message for all failure cases
4. On success, issues a JWT access token (30 min) and refresh token (7 days)
5. All subsequent requests carry the access token in the `Authorization` header
6. Expired access tokens are silently refreshed using the refresh token
7. On refresh failure, vendor is redirected to the login page

### Profile sync flow
1. Scheduled job runs every 6 hours, reads `res.partner` where `supplier_rank > 0`
2. **New partner:** creates `vendor_users` row (no password, inactive), generates 24h invite token, sends welcome email via AWS SES containing the Vendor ID and set-password link
3. **Existing partner:** updates profile fields only — `hashed_password` is never touched
4. **Partner deactivated in Odoo:** sets `is_active = FALSE` — vendor can no longer log in
5. Partners with no email are skipped and logged for manual follow-up

### Receipt update flow
1. Vendor browses their PO list (filtered to `purchase` and `done` states), searchable by PO number or date range
2. Vendor opens a PO and sees linked receipts in `assigned` or `done` state
3. Vendor opens a ready receipt and enters actual delivered quantities — saves are incremental, multiple updates are allowed
4. When ready, vendor proceeds to the signature step, draws their signature, and confirms
5. Backend verifies ownership, stores the signature PNG, fetches the Odoo delivery slip PDF via HTTP session, appends a signed confirmation page (WeasyPrint + pypdf), and marks the receipt as **locked** in the portal database
6. Vendor downloads the final merged PDF
7. AWS SES sends a confirmation email to the vendor and an alert email to the internal team
8. The receipt is now read-only in the portal — only an Odoo-side admin action can unlock it (the portal checks a `locked` flag in its own DB; it does not push a lock back to Odoo)

---

## Business Logic and Flow

This section describes the portal's behaviour in plain business terms, intended for stakeholder communication. It covers the four main scenarios: onboarding a new vendor, day-to-day portal usage, the delivery confirmation workflow, and exception handling.

---

### 1. Vendor Onboarding

**Trigger:** A new vendor is registered in Odoo with `supplier_rank > 0` and has a valid email address on their partner record.

**What happens automatically:**
1. The portal sync job runs every 6 hours and detects the new vendor in Odoo
2. A portal account is created, linked to the vendor's Odoo ID
3. The vendor receives a **Welcome Email** (in Vietnamese by default) containing:
   - Their **Vendor ID** — a permanent number they will use to log in
   - A **set-password link** valid for 24 hours
4. The vendor clicks the link, sets their own password, and their account becomes active
5. From this point, the vendor can log in at any time using their Vendor ID and password

**If the vendor misses the 24h window:** they use the "Forgot Password" option on the login page, enter their Vendor ID, and receive a new reset link by email.

**If the vendor has no email in Odoo:** the sync job skips them and logs the case. The internal team must add an email to the Odoo partner record — the account will be created on the next sync cycle.

**Profile updates:** If the vendor's name, email, phone, or company name changes in Odoo, the portal reflects those changes automatically on the next sync. The vendor's password is never affected by sync.

---

### 2. Viewing Purchase Orders

**Who can see what:** Each vendor sees only their own Purchase Orders. It is technically impossible for a vendor to view another vendor's data.

**Which POs are visible:**
- **Sent RFQ** — RFQ has been sent to the vendor and is awaiting confirmation. Vendor can confirm it directly in the portal.
- **Confirmed** — PO has been approved and is active
- **Cancelled** — PO has been cancelled (shown for vendor awareness, read-only)
- **Done** — all receipts have been fully processed

Draft RFQs are not shown.

**Confirming a Sent RFQ:**
- A "Confirm PO" button is shown on Sent RFQ detail pages
- Vendor clicks "Confirm PO" → portal calls `button_confirm` on `purchase.order` via XML-RPC using the service account
- Odoo updates the PO state from `sent` to `purchase` and generates the linked Delivery Order / Receipt
- The portal reflects the new `purchase` state immediately after the call succeeds
- Once confirmed, the button is no longer shown — the record is now read-only on the PO level
- **Note:** Whether Odoo enforces an Order Deadline on RFQs is pending IT confirmation (see Section 8)

**Searching and filtering:**
- Vendor can search by PO number (e.g. typing "PO004" filters the list instantly)
- Vendor can filter by date range (e.g. "POs from January to March")
- Results are paginated — 20 POs per page

**PO detail view:** clicking a PO shows the full list of ordered products with quantities and expected delivery dates, plus all linked delivery receipts.

---

### 3. Delivery Confirmation Workflow

This is the core business process of the portal. It covers the full flow from DO creation to physical delivery at the store.

```
Purchase Order confirmed in Odoo
         │
         ▼
DO (Delivery Order) sinh ra tự động với số ref theo PO
         │
         ▼
Vendor logs into portal, chỉnh DO:
  - Ngày giao hàng
  - Số lượng giao (≤ số lượng đặt)
  (can save and come back multiple times)
         │
         ▼
Vendor clicks "Sign & Confirm" DO on Portal
         │
         ▼
Vendor draws their signature on screen
         │
         ▼
Portal generates signed DO PDF (có chữ ký xác nhận)
DO is locked — no further edits on Portal
         │
         ▼
Vendor downloads and prints PDF (2 copies)
         │
         ▼
Vendor brings 2 DO copies to the store (cửa hàng)
         │
         ▼
Store physically signs both DO copies
  — each party keeps 1 copy
         │
         ▼
Store confirms on Odoo
```

**Key points for stakeholders:**
- Vendor ký xác nhận DO trên Portal trước khi giao hàng — PDF có chữ ký là chứng từ chính thức
- Vendor cầm 2 bản DO (PDF đã ký) ra cửa hàng — cửa hàng ký physically cả 2 bản
- Cửa hàng xác nhận trên Odoo sau khi nhận hàng và ký DO
- Không có quy trình tự động khoá DO lúc 23:00 — vendor chủ động ký khi sẵn sàng giao

---

### 4. Receipt States and What They Mean

| State in Portal | State in Odoo | What the vendor can do |
|---|---|---|
| **Ready** | `assigned` | Enter qty_done, save multiple times, sign |
| **Signed / Locked** | `assigned` (pending admin) | View only, download PDF |
| **Done** | `done` | View only, download PDF |

**Note:** A receipt remains in Odoo's `assigned` state after the vendor signs — it moves to `done` only after the internal team validates it in Odoo. The vendor sees the "Signed" status on the portal, which is a portal-level flag, not an Odoo state.

---

### 5. Post-Signature Locking and Unlocking

**Why receipts are locked after signing:** Once a vendor confirms delivery quantities with their signature, that record becomes the official delivery confirmation. Allowing edits after signing would undermine the document's legal and operational value.

**What "locked" means in practice:**
- All quantity fields on the receipt are read-only in the portal
- The signed PDF remains downloadable at any time
- The vendor cannot re-sign or submit a new signature

**Unlocking process:**
If quantities were entered incorrectly and the vendor needs to resubmit, a portal admin can unlock the receipt directly from the admin section (`/admin/vendors/:id`). Unlocking removes the portal lock record and notifies no one automatically — the admin should communicate with the vendor directly. Once unlocked, the vendor can update quantities and re-sign. The unlock action is logged in `receipt_locks` (unlocked_at, unlocked_by).

---

### 6. Email Notifications Summary

| Event | Recipient | Language | Content |
|---|---|---|---|
| New vendor account created | Vendor | Vietnamese (default) | Vendor ID + set-password link |
| Password reset requested | Vendor | Vendor's preferred language | Reset link (24h expiry) |
| Receipt signed by vendor | Vendor | Vendor's preferred language | Confirmation + PDF attachment |
| Receipt signed by vendor | Internal team | English | Vendor name, PO ref, receipt ref, link to Odoo |

---

### 7. What the Portal Does NOT Do

It is equally important for stakeholders to understand the boundaries of the portal:

- **Does not validate stock movements** — all Odoo validation is done by the internal team
- **Does not create Purchase Orders** — vendors can confirm a Sent RFQ but cannot create, edit lines, or cancel a PO
- **Does not handle invoicing or payments** — outside the scope of this portal
- **Does not manage backorders** — the internal team decides on backorders in Odoo after reviewing vendor-submitted quantities
- **Does not expose any Odoo credentials to vendors** — vendors have no access to Odoo, directly or indirectly
- **Does not allow vendors to see other vendors' data** — enforced at every layer of the system

---

### 8. Data Sync between Portal and Odoo (⚠️ Pending IT Confirmation)

> **Lưu ý:** Section này tập hợp tất cả các điểm liên quan đến việc đồng bộ dữ liệu giữa Portal và Odoo. Cần hỏi lại IT team để xác nhận cơ chế và thời điểm sync trước khi implement.

**Các điểm cần xác nhận với IT:**

| # | Chủ đề | Mô tả hiện tại | Cần xác nhận |
|---|---|---|---|
| 1 | **Order Deadline trên Odoo** | Specs hiện tại giả định PO có Order Deadline, vendor phải confirm trước deadline | Odoo 16 CE có trường Order Deadline trên `purchase.order` không? Nếu có, trường nào? Nếu không, cơ chế hết hạn RFQ hoạt động ra sao? |
| 2 | **Vendor profile sync** | Sync job chạy mỗi 6h, đọc `res.partner` từ Odoo → tạo/cập nhật `vendor_users` trên Portal | Tần suất sync? Real-time hay batch? |
| 3 | **PO data sync** | Portal đọc PO trực tiếp từ Odoo qua XML-RPC mỗi khi vendor truy cập | Cache strategy? Có cần sync PO về Portal DB không? |
| 4 | **PO confirmation → Odoo** | Portal gọi `button_confirm` trên `purchase.order` qua XML-RPC | Có cần thêm validation nào phía Odoo không? |
| 5 | **DO data** | DO sinh ra từ Odoo khi PO confirmed, Portal đọc qua XML-RPC | Cách Portal nhận biết DO mới? Polling hay webhook? |
| 6 | **DO sign → Odoo** | Sau khi vendor ký DO trên Portal, thông tin có cần push về Odoo không? | Push gì? Khi nào? |
| 7 | **Receipt data sync** | Portal đọc `stock.picking` + `stock.move.line` từ Odoo | Vendor nhập `qty_done` trên Portal — có write ngược về Odoo không? |
| 8 | **Receipt PDF** | Portal fetch delivery slip PDF từ Odoo HTTP session | Có cần lưu PDF trên Odoo hay chỉ Portal? |
| 9 | **Vendor deactivation** | Khi vendor bị deactivate trên Odoo → sync job set `is_active = FALSE` trên Portal | Có cần deactivate ngay lập tức hay chờ sync cycle? |

**Cơ chế sync hiện tại (draft — chờ IT xác nhận):**
- **Odoo → Portal:** Scheduled job (mỗi 6h) cho vendor profiles; XML-RPC read-on-demand cho PO/DO/Receipt
- **Portal → Odoo:** XML-RPC write cho PO confirmation (`button_confirm`); các thao tác khác chưa xác định
- **PDF fetch:** HTTP session riêng (không dùng XML-RPC) để tải delivery slip từ Odoo

---

## Phase 0 — Project Setup & Infrastructure
**Effort: 0.5–1 day**

### Objectives
- Establish monorepo structure with `/frontend`, `/backend`, `/infra` folders
- Configure Docker Compose with all five services: frontend, backend, PostgreSQL, Redis, Nginx
- Set up environment variable strategy and secrets management
- Establish network connectivity between the portal VM and the Odoo VM
- Configure AWS SES: verify sender domain, create IAM user with SES send permissions, store credentials as secrets

### Repo structure
```
vendor-portal/
├── frontend/               # React + Vite
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/            # fetch wrapper + TanStack Query hooks
│   │   └── i18n/           # Vietnamese + English translation files
│   └── Dockerfile
├── backend/                # FastAPI
│   ├── app/
│   │   ├── api/            # route handlers
│   │   ├── services/       # odoo_client, pdf_service, email_service
│   │   ├── models/         # SQLAlchemy models
│   │   ├── schemas/        # Pydantic schemas
│   │   ├── jobs/           # sync_vendors, scheduler
│   │   └── core/           # config, security, dependencies
│   ├── alembic/            # DB migrations
│   └── Dockerfile
├── infra/
│   ├── docker-compose.yml
│   ├── docker-compose.prod.yml
│   ├── nginx/
│   │   └── nginx.conf
│   └── secrets/            # gitignored
├── .env.example
└── README.md
```

### Key decisions
- **Monorepo** keeps frontend and backend versioned together, simplifying deployment
- **Docker Compose** with separate dev and prod configs — prod uses Docker secrets for sensitive values
- **AWS SES setup:** verify the sending domain in SES, create a dedicated IAM user with `ses:SendEmail` permission only, store `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as Docker secrets. Use `boto3` in the FastAPI email service.
- **Odoo connectivity:** portal VM IP must be whitelisted on Odoo VM firewall for port 8069. A dedicated Odoo service account with an API key is the only credential the portal uses.

### Environment variables needed
- `ODOO_URL`, `ODOO_DB`, `ODOO_USERNAME`, `ODOO_API_KEY`
- `JWT_SECRET_KEY`, `JWT_REFRESH_SECRET`
- `DB_PASSWORD`, `REDIS_URL`
- `FRONTEND_URL` (for CORS and email links)
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SES_REGION`, `SES_SENDER_EMAIL`
- `INTERNAL_NOTIFICATION_EMAIL` (team inbox for validation alerts)
- `ADMIN_INITIAL_PASSWORD` (used to seed the first admin account on first startup)

---

## Phase 1 — Odoo Integration Layer
**Effort: 1–2 days**

### Objectives
- Build the Odoo XML-RPC client as a singleton service
- Implement the vendor-scoped query wrapper (row-level isolation)
- Implement and schedule the partner sync job
- Validate all required Odoo queries work correctly against the live instance

### Key design decisions
- **Singleton OdooClient:** authenticates once at startup, reuses the `uid` for all calls. Reconnects automatically on connection drop
- **VendorScopedOdooClient:** a wrapper that injects `[['partner_id', '=', partner_id]]` into every domain filter at the service layer — no route handler can accidentally omit vendor scoping
- **Sync job scheduling:** APScheduler running inside the FastAPI process, every 6 hours. Can also be triggered manually via an internal admin endpoint
- **PDF session:** a separate `requests.Session()` authenticated via `/web/session/authenticate` is maintained for PDF downloads — the XML-RPC `uid` cannot access `/report/pdf/` endpoints

### Odoo models in scope
| Model | Purpose | Key fields |
|---|---|---|
| `res.partner` | Vendor profile sync | `id`, `name`, `email`, `company_name`, `phone`, `supplier_rank` |
| `purchase.order` | PO list, detail, and vendor confirmation | `name`, `partner_id`, `state`, `date_order`, `amount_total`, `order_line`, `picking_ids` — `button_confirm` called on vendor action |
| `purchase.order.line` | PO line items | `product_id`, `name`, `product_qty`, `qty_received`, `price_unit` |
| `stock.picking` | Receipt header | `name`, `state`, `scheduled_date`, `origin`, `move_line_ids` |
| `stock.move.line` | Receipt line items | `product_id`, `product_uom_qty`, `qty_done`, `lot_id`, `state` |

### Odoo 16 CE field name gotchas
- `stock.move.line.qty_done` → renamed to `quantity` in Odoo 17+
- `stock.picking.move_lines` → renamed to `move_ids` in Odoo 17+
- Always specify explicit field lists in `search_read` — reading all fields on Odoo 16 can trigger serialization errors on computed fields like `tax_totals`

### Data filtering rules (applied at backend, not frontend)
- POs: only `state IN ('sent', 'purchase', 'cancel', 'done')` are returned — `draft` is excluded
- Receipts: only `state IN ('assigned', 'done')` are returned
- All queries are scoped by `partner_id` via VendorScopedOdooClient

---

## Phase 2 — Auth System
**Effort: 1 day**

### Objectives
- Define the `vendor_users`, `admin_users`, `password_reset_tokens`, and `receipt_locks` database tables
- Implement JWT access + refresh token issuance and validation, with role claim (`vendor` or `admin`)
- Implement the vendor invite flow (first-time password set via token from AWS SES email)
- Implement the forgot-password flow for vendors (same token mechanism)
- Implement admin account seeding from `ADMIN_INITIAL_PASSWORD` environment variable on first startup
- Implement the FastAPI auth dependency used by all protected routes, with role-based access control

### Database tables

**`admin_users`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal ID |
| `username` | varchar, unique | login identifier for admin (not an Odoo ID) |
| `hashed_password` | varchar | seeded from `ADMIN_INITIAL_PASSWORD` on first startup |
| `full_name` | varchar | manually set |
| `email` | varchar | for notifications and password reset |
| `is_active` | boolean | TRUE by default |
| `created_at` | timestamp | |
| `last_login` | timestamp | |

**Admin account seeding:** On first application startup, if no rows exist in `admin_users`, the backend inserts a default admin account using `ADMIN_INITIAL_PASSWORD` from the environment. The admin must change this password after first login.

**`vendor_users`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal ID |
| `odoo_partner_id` | integer, unique | login identifier shown to vendor |
| `hashed_password` | varchar | NULL until invite accepted |
| `full_name` | varchar | synced from Odoo, never manually edited |
| `email` | varchar | synced from Odoo, used for emails only |
| `company_name` | varchar | synced from Odoo |
| `phone` | varchar | synced from Odoo |
| `preferred_language` | varchar | `'vi'` or `'en'`, set by vendor, default `'vi'` |
| `is_active` | boolean | FALSE until first password set |
| `created_at` | timestamp | |
| `last_login` | timestamp | updated on each successful login |

**`password_reset_tokens`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `user_id` | FK → vendor_users | |
| `token` | varchar, unique | `secrets.token_urlsafe(32)` |
| `expires_at` | timestamp | 24 hours from creation |
| `used` | boolean | marked TRUE after use |

**`receipt_locks`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `picking_id` | integer, unique | Odoo `stock.picking.id` |
| `locked_by` | FK → vendor_users | vendor who signed |
| `locked_at` | timestamp | when signature was submitted |
| `signature_path` | varchar | server-side path to stored PNG |
| `pdf_path` | varchar | server-side path to stored signed PDF |
| `vendor_comment` | text | optional free-text note submitted by vendor at signing |
| `unlocked_at` | timestamp | NULL unless admin unlocked via portal |
| `unlocked_by` | FK → admin_users | admin who unlocked, NULL if not unlocked |

**`audit_log`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `actor_type` | varchar | `vendor` or `admin` |
| `actor_id` | integer | `vendor_users.id` or `admin_users.id` |
| `action` | varchar | one of: `login`, `qty_update`, `sign`, `unlock` |
| `target_model` | varchar | e.g. `receipt`, `vendor` |
| `target_id` | integer | e.g. `picking_id`, `partner_id` |
| `detail` | jsonb | action-specific metadata (e.g. which lines were updated, old/new qty) |
| `created_at` | timestamp | when the action occurred |
| `ip_address` | varchar | client IP for login events |

### Auth endpoints
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/login` | Vendor ID + password → access + refresh tokens (role: `vendor`) |
| POST | `/api/admin/auth/login` | Username + password → access + refresh tokens (role: `admin`) |
| POST | `/api/auth/refresh` | Refresh token → new access token (preserves role) |
| POST | `/api/auth/set-password` | First-time invite or password reset via token (vendors only) |
| POST | `/api/auth/forgot-password` | Sends reset email via AWS SES by Vendor ID |
| POST | `/api/auth/logout` | Blacklists refresh token in Redis |

### JWT role claim
The JWT access token carries a `role` field (`vendor` or `admin`). All admin-only endpoints check for `role == 'admin'` via a dedicated FastAPI dependency. Vendor endpoints check for `role == 'vendor'`. A vendor token cannot access admin routes, and an admin token cannot be used to impersonate a vendor.

### Security considerations
- **Timing-safe login:** always run `bcrypt.verify` even when the Vendor ID does not exist
- **Consistent error messages:** same message for unknown ID and wrong password — never reveal whether the ID exists
- **Token blacklist:** on logout, refresh token is added to a Redis set with TTL matching remaining token lifetime
- **Invite token expiry:** 24 hours. If expired, vendor must request a new one via forgot-password

---

## Phase 3 — Core API Endpoints
**Effort: 2–3 days**

### Objectives
- Implement all data endpoints with vendor-scoped Odoo queries
- Enforce receipt locking logic server-side
- Implement the PDF generation and signing pipeline
- Implement all email notification triggers via AWS SES

### Full endpoint list
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/vendors/me` | Current vendor profile |
| PATCH | `/api/vendors/me/language` | Update preferred language (`vi` or `en`) |
| GET | `/api/purchase-orders` | Paginated PO list, filterable by PO number and date range |
| GET | `/api/purchase-orders/{id}` | PO detail with order lines |
| POST | `/api/purchase-orders/{id}/confirm` | Confirm a Sent RFQ — calls `button_confirm` on Odoo; returns 409 if PO is not in `sent` state |
| GET | `/api/receipts/{picking_id}` | Receipt detail with move lines and current `qty_done` |
| PATCH | `/api/receipts/{picking_id}/lines` | Update `qty_done` on one or more move lines (blocked if locked) |
| POST | `/api/receipts/{picking_id}/sign` | Submit signature PNG + optional comment, lock receipt, trigger emails, generate PDF |
| GET | `/api/receipts/{picking_id}/pdf` | Download signed delivery PDF (available post-signature) |

### Admin-only endpoints
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/admin/vendors` | List all vendor accounts with status, last login, signed receipt count, pending receipt count |
| GET | `/api/admin/vendors/{partner_id}` | Drill into a specific vendor — their POs and receipts (read-only) |
| PATCH | `/api/admin/vendors/{partner_id}/deactivate` | Deactivate a vendor account |
| PATCH | `/api/admin/vendors/{partner_id}/reactivate` | Reactivate a vendor account |
| POST | `/api/admin/sync` | Manually trigger the Odoo partner sync job |
| GET | `/api/admin/sync/status` | Return last sync time, number of vendors synced, list of skipped vendors (no email) |
| GET | `/api/admin/receipts/{picking_id}/pdf` | Download any vendor's signed PDF |
| POST | `/api/admin/receipts/{picking_id}/unlock` | Remove the receipt lock — vendor can update and re-sign |
| GET | `/api/admin/audit-log` | Paginated audit log view, filterable by actor, action type, and date range |

All admin endpoints require `role == 'admin'` in the JWT. They are prefixed with `/api/admin/` to make the distinction explicit and apply a separate rate limiting policy.

### PO list filtering
The `GET /api/purchase-orders` endpoint accepts the following optional query parameters:
- `q` — partial match on PO number (e.g. `PO004`)
- `date_from` — filter POs with `date_order >= date_from`
- `date_to` — filter POs with `date_order <= date_to`
- `page` and `page_size` — pagination (default page size: 20)

Filtering is applied server-side in the Odoo domain filter, not in the frontend.

### Receipt locking and unlocking approach
- When `POST /api/receipts/{picking_id}/sign` is called, the backend inserts a row into `receipt_locks` and marks the receipt as locked
- Any subsequent `PATCH /lines` call on a locked receipt returns HTTP 423 (Locked)
- The receipt detail endpoint returns a `locked: true` flag — the frontend disables all editing accordingly
- **Unlocking:** a portal admin calls `POST /api/admin/receipts/{picking_id}/unlock`, which deletes the `receipt_locks` row and records who unlocked it and when. The vendor can then update quantities and re-sign. The Odoo picking state is not affected — it remains `assigned` throughout.

### Validation approach (no portal-side `button_validate`)
Since the admin decides on backorders in Odoo, the portal does **not** call `button_validate`. The vendor's role is:
1. Enter `qty_done` values (multiple saves allowed)
2. Sign the receipt (locks the portal record)

The Odoo picking remains in `assigned` state after the vendor signs — the internal team sees the vendor's `qty_done` values and validates in Odoo at their discretion.

### Audit logging approach
Every key action is written to the `audit_log` table synchronously within the same request. The four logged action types are:

| Action | Trigger | Detail stored |
|---|---|---|
| `login` | Successful vendor or admin login | actor type, actor ID, IP address, timestamp |
| `po_confirm` | `POST /purchase-orders/:id/confirm` | PO ID, PO name, previous state (`sent`), new state (`purchase`) |
| `qty_update` | `PATCH /receipts/:id/lines` | picking ID, list of move line IDs with old and new `qty_done` values |
| `sign` | `POST /receipts/:id/sign` | picking ID, vendor comment (if any), PDF path, signature path |
| `unlock` | `POST /admin/receipts/:id/unlock` | picking ID, admin who unlocked, reason if provided |

The audit log is viewable by admins via `GET /api/admin/audit-log`, filterable by actor, action type, and date range, paginated at 50 rows per page. It is never editable or deletable through the portal.

### Email notifications (AWS SES)

| Trigger | Recipient | Content |
|---|---|---|
| New vendor synced | Vendor | Welcome email with Vendor ID + set-password link |
| Forgot password requested | Vendor | Reset link (24h expiry) |
| Receipt signed by vendor | Vendor | Confirmation with PO number, receipt reference, PDF attachment |
| Receipt signed by vendor | Internal team (`INTERNAL_NOTIFICATION_EMAIL`) | Alert with vendor name, PO number, receipt reference, link to Odoo |

All emails are sent in the vendor's `preferred_language` for vendor-facing emails, and in English for internal team emails.

### PDF pipeline approach
1. Check `receipt_locks` — PDF only generated after signature is submitted
2. Fetch the Odoo delivery slip via authenticated HTTP session (`/report/pdf/stock.report_deliveryslip/{picking_id}`)
3. Generate a confirmation page as HTML rendered to PDF with WeasyPrint, embedding: vendor's signature PNG, vendor name, Vendor ID, PO reference, timestamp, and vendor comment (if provided)
4. Merge both PDFs with pypdf (delivery slip first, confirmation page appended)
5. Store the merged PDF permanently on the server filesystem (no expiry)
6. Return the PDF as a binary response with `Content-Disposition: attachment`

---

## Phase 4 — React Frontend
**Effort: 3–4 days**

### Objectives
- Build all portal pages with clean, mobile-friendly UI
- Implement bilingual support (Vietnamese + English, user-switchable)
- Implement the `fetch`-based API client with automatic JWT refresh
- Integrate `signature_pad.js` for signature capture
- Implement the qty_done update form with save and lock state handling

### Internationalisation (i18n) approach
- Use `react-i18next` with two locale files: `vi.json` and `en.json`
- Language is stored in `vendor_users.preferred_language` (persisted server-side)
- A language toggle (VI / EN) is visible in the top navigation bar on all pages
- On language switch, the frontend calls `PATCH /api/vendors/me/language` and updates the i18next locale immediately — no page reload needed
- Default language: Vietnamese (`vi`)
- All UI strings, labels, error messages, and button text are translated
- Email content sent by backend follows the vendor's stored `preferred_language`

### Pages & routes
| Route | Page | Role | Description |
|---|---|---|---|
| `/login` | Login | Both | Vendor ID (number input) + password + language toggle |
| `/admin/login` | Admin Login | Admin | Username + password (separate login page) |
| `/set-password` | Set Password | Vendor | First-time invite and forgot-password reset |
| `/forgot-password` | Forgot Password | Vendor | Enter Vendor ID to receive reset email |
| `/dashboard` | Dashboard | Vendor | PO list with search/filter bar and status badges |
| `/purchase-orders/:id` | PO Detail | Vendor | Order lines + linked receipts (Ready / Done) |
| `/receipts/:id` | Receipt Detail | Vendor | Move lines with editable qty_done fields (locked if signed) |
| `/receipts/:id/sign` | Sign & Confirm | Vendor | Signature pad + final confirmation step |
| `/receipts/:id/pdf` | PDF Download | Vendor | Download link for signed delivery PDF |
| `/pdfs` | PDF History | Vendor | List of all previously signed PDFs with download links |
| `/profile` | Profile | Vendor | Read-only vendor profile + language preference |
| `/admin/dashboard` | Admin Dashboard | Admin | Summary stats: vendor counts, pending receipts, sync status |
| `/admin/vendors` | Vendor List | Admin | All vendor accounts with status, last login, receipt counts |
| `/admin/vendors/:id` | Vendor Detail | Admin | Specific vendor's POs and receipts (read-only drill-down) |
| `/admin/sync` | Sync Status | Admin | Last sync time, skipped vendors, manual trigger button |

### Admin dashboard content
The admin dashboard shows the following at a glance:
- **Vendor summary:** total vendors, active count, inactive count
- **Vendor table:** sortable list showing vendor name, Vendor ID, account status (active/inactive), last login date, number of signed receipts, number of pending receipts (Ready but not yet signed)
- **Sync status panel:** last sync timestamp, number of vendors synced, number of vendors skipped (no email), manual "Run Sync Now" button
- Clicking any vendor row navigates to `/admin/vendors/:id` — a read-only view of that vendor's POs and receipts, with a download button for any signed PDF and an unlock button on locked receipts

### Admin navigation
Admin users see the same top navigation bar as vendors, with additional items: **Vendors**, **Sync Status**, **Audit Log**. The language toggle is present — admin section supports both Vietnamese and English. Admin routes are protected — a vendor JWT cannot access `/admin/*` routes.

### HTTP client approach (native fetch)
- A single `apiFetch(path, options)` wrapper handles all API calls
- Attaches Bearer token from `localStorage` on every request
- On 401 response: calls `/api/auth/refresh`, stores new access token, retries once
- On refresh failure: clears storage, redirects to `/login`
- No direct `fetch` calls in components — all calls go through this wrapper

### State management approach
- **TanStack Query** for all server state — PO list, receipt detail, profile
- **React local state** (`useState`) for form inputs, signature state, UI toggles
- No global state manager needed at this scope

### Responsive design approach
- The portal is fully responsive — works on desktop and mobile equally
- Tailwind CSS utility classes for layout (or equivalent CSS framework)
- Signature pad canvas resizes to fit the screen width on mobile
- Tables on mobile collapse to card-style layouts for readability

### Vendor dashboard summary
Above the PO list, the vendor dashboard shows three summary cards:
- **Total POs** — count of all visible POs (Confirmed + Done)
- **Pending receipts** — receipts in Ready state not yet signed
- **Signed receipts** — receipts the vendor has already confirmed

### PO list search & filter UI
- Search bar for PO number (partial match, debounced 300ms before API call)
- Date range pickers for `date_from` and `date_to`
- Status badge on each PO row (Confirmed / Done) with colour coding
- Pagination controls (previous / next), page size fixed at 20

### Receipt detail — visible fields per move line
| Field | Source |
|---|---|
| Product code / SKU | `stock.move.line` → `product_id.default_code` |
| Product description | `stock.move.line` → `product_id.name` |
| Unit of measure | `stock.move.line` → `product_uom_id.name` |
| Ordered quantity | `purchase.order.line` → `product_qty` |
| Delivered quantity (editable) | `stock.move.line` → `qty_done` |
| Unit price | `purchase.order.line` → `price_unit` |
| Subtotal | Calculated: unit price × delivered qty (frontend only, not stored) |

### Receipt detail behaviour
- Each move line shows all fields above; delivered qty is the only editable field
- "Save" button submits `PATCH /lines` — can be pressed multiple times before signing
- If `locked: true` is returned by the API, all inputs are disabled and a lock notice is shown
- A "Sign & Confirm" button at the bottom is enabled only when all `qty_done` values are > 0
- Once signed, the page shows a read-only summary and a PDF download button

### PDF history page
- Lists all receipts the vendor has signed, in reverse chronological order
- Each row shows: PO number, receipt reference, date signed, vendor comment (if any), download button
- Download calls `GET /api/receipts/{picking_id}/pdf` — PDFs are stored permanently and always available

### Signature and comment capture
- `signature_pad.js` renders on an HTML5 canvas (resizes for mobile)
- Optional free-text comment field above the signature pad — vendor can describe delivery conditions, discrepancies, or notes
- "Clear" button resets the canvas only
- "Confirm & Submit" sends both the signature PNG (base64) and the comment text to `POST /api/receipts/:id/sign`
- Pad and comment field are disabled after successful submission
- A loading state is shown while the PDF is being generated server-side

### Key UX considerations
- Login field label: **"Mã nhà cung cấp / Vendor ID"** with helper text explaining it was provided in the welcome email
- Receipt lines show ordered qty and delivered qty side by side for easy comparison
- Lock notice on signed receipts: "Phiếu nhập đã được xác nhận / Receipt has been confirmed"
- Comment field placeholder: "Ghi chú giao hàng / Delivery notes (optional)"
- All form submissions disable the button while in flight to prevent double submission

---

## Phase 5 — Security Hardening
**Effort: 0.5 day**

### Rate limiting strategy (SlowAPI + Redis)
| Endpoint group | Limit |
|---|---|
| `/api/auth/login` | 5 requests / minute / IP |
| `/api/auth/forgot-password` | 3 requests / minute / IP |
| `/api/auth/set-password` | 5 requests / minute / IP |
| All other `/api/` routes | 60 requests / minute / user |

### CORS
- Allow only the exact `FRONTEND_URL` — never wildcard `*` with credentials
- `allow_credentials=True`

### HTTPS / TLS
TLS termination is handled at the infrastructure level (load balancer or upstream reverse proxy) — the portal's Nginx does not manage SSL certificates. The portal Nginx listens on HTTP internally and trusts the upstream proxy to enforce HTTPS. Ensure the upstream proxy passes `X-Forwarded-Proto: https` and `X-Real-IP` headers to the backend.

### Nginx security headers
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Strict-Transport-Security: max-age=31536000` (set at upstream level if TLS is upstream)
- `Referrer-Policy: no-referrer`

### AWS SES security
- Dedicated IAM user with `ses:SendEmail` permission only — no other AWS permissions
- Sender domain verified in SES
- Credentials stored as Docker secrets, never in `.env` files in production

---

## Phase 6 — Production Deployment
**Effort: 0.5–1 day**

### Service restart policies
All containers: `restart: unless-stopped`. Health check on backend container (`GET /health`), 3 retries before marking unhealthy.

### Nginx reverse proxy routing
- All `/api/*` → FastAPI backend container
- All `/admin/*` → React SPA container (admin routes handled client-side by React Router)
- All other paths → React SPA container, catch-all serves `index.html`

### PostgreSQL backup
- Daily `pg_dump` via cron on the portal VM
- Compressed with gzip, stored in `/backups/`, 30-day retention
- Optionally synced to Azure Blob Storage
- PDF files stored permanently on server filesystem — include in VM backup strategy

### Go-live checklist
- [ ] Odoo service account API key generated and connectivity tested from portal VM
- [ ] Firewall: portal VM → Odoo VM port 8069 open
- [ ] AWS SES sender domain verified, IAM credentials configured
- [ ] `INTERNAL_NOTIFICATION_EMAIL` set to correct team inbox
- [ ] `ADMIN_INITIAL_PASSWORD` set and first admin account seeded
- [ ] All production secrets set in Docker secrets files
- [ ] Upstream load balancer / proxy configured to forward HTTPS traffic
- [ ] Partner sync job run manually once — initial vendor accounts created
- [ ] At least one test vendor: invite received → password set → login → dashboard summary visible → PO found via search → qty updated with comment → signed → PDF downloaded → PDF history shows entry
- [ ] Notification emails received by vendor and internal team
- [ ] Locked receipt confirmed uneditable by vendor
- [ ] Admin can view vendor, download PDF, unlock receipt
- [ ] Audit log shows all actions from the test run
- [ ] Vietnamese and English UI both verified on desktop and mobile

---

## Summary Timeline

| Phase | Description | Effort |
|---|---|---|
| 0 | Repo setup, Docker Compose, Odoo API key, AWS SES | 0.5–1 day |
| 1 | Odoo XML-RPC layer + partner sync job | 1–2 days |
| 2 | JWT auth + role system + invite/reset flow + DB schema | 1–1.5 days |
| 3 | Core API endpoints + admin endpoints + PDF + emails | 3–4 days |
| 4 | React frontend (vendor portal + admin section, bilingual) | 4–5 days |
| 5 | Security hardening | 0.5 day |
| 6 | Production deployment + go-live checklist | 0.5–1 day |
| **Total** | | **~11–15 days** |

---

## Critical Gotchas

1. **Login uses Vendor ID (integer), not email** — frontend login field must be a number input; welcome email must clearly state the Vendor ID
2. **Admin login is separate** — admin uses username (not Vendor ID) on a separate login page `/admin/login`; admin JWT carries `role: admin`
3. **Profile sync never touches `hashed_password`** — only `full_name`, `email`, `company_name`, `phone` are ever overwritten by the sync job; admin cannot edit vendor profiles in the portal
4. **Portal does not call `button_validate`** — vendor submits qty_done and signs; Odoo validation is done by the internal team. The Odoo picking stays in `assigned` state after the vendor signs
5. **Receipt locking is portal-side only** — the `receipt_locks` table lives in the portal DB; Odoo is not aware of it. Admins unlock via the portal admin section
6. **Admin can unlock receipts — vendor cannot** — the unlock endpoint is admin-only; vendors have no self-service unlock
7. **PDF requires a separate HTTP session** — the XML-RPC connection cannot fetch `/report/pdf/` endpoints; a distinct `requests.Session()` authenticated via `/web/session/authenticate` is required
8. **PDFs are stored permanently** — no expiry or cleanup job; ensure the server filesystem or storage volume has adequate space and is included in the VM backup strategy
9. **Vendor comment is optional but must be stored** — even if empty, the `vendor_comment` field in `receipt_locks` must be explicitly stored (empty string, not NULL) to distinguish "no comment provided" from a data error
10. **Audit log is append-only** — no delete or update endpoints for `audit_log`; write failures should be logged but must not cause the parent action to fail (non-blocking)
11. **Odoo 16 field names** — `qty_done` on `stock.move.line` (not `quantity`), `move_lines` on `stock.picking` (not `move_ids`)
12. **Partners with no email are skipped** — sync job must log these; they cannot receive an invite and must be handled manually
13. **Timing-safe authentication** — bcrypt verification must always run, even for unknown Vendor IDs, to prevent user enumeration via response timing
14. **No axios** — all frontend HTTP calls use native `fetch` wrapped in a single `apiFetch` utility with automatic JWT refresh
15. **i18n covers both vendor and admin UI** — all portal pages including admin section support Vietnamese and English; vendor-facing emails follow `preferred_language`; internal alert emails are always in English
16. **TLS is upstream** — the portal Nginx does not manage certificates; ensure the upstream proxy passes `X-Forwarded-Proto` and `X-Real-IP` headers correctly to the FastAPI backend
17. **`button_confirm` is a write action on Odoo** — unlike all other vendor-facing data calls which are read-only, PO confirmation calls `execute_kw` with `button_confirm` on `purchase.order`. The Odoo service account must have write permission on `purchase.order`. Verify this permission explicitly — read-only service accounts will silently fail or return an access error
18. **Guard PO confirmation against double-submit** — the confirm endpoint must re-read the PO state from Odoo before calling `button_confirm`; if the state is already `purchase`, return 409 without calling Odoo again. The frontend must also disable the "Confirm PO" button immediately on click to prevent duplicate calls
