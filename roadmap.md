# Vendor Portal — Implementation Roadmap

> Technical phases, DB schema, API endpoints, and developer gotchas.
> For business logic and process flow see [README.md](README.md) and [PROCESS_FLOW.md](PROCESS_FLOW.md).

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
- ~~`INTERNAL_NOTIFICATION_EMAIL`~~ — removed: store notifications go to the PO creator's email from Odoo, not a generic inbox
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
- **Sync job scheduling:** APScheduler running inside the FastAPI process, every 6 hours for vendor profiles. Can also be triggered manually via an internal admin endpoint
- **Odoo → Portal sync (TBD):** Mechanism for portal to learn when store confirms a Receipt is pending team confirmation. Options: webhook (requires Odoo module), polling, or nightly batch. Regardless of mechanism, portal will read confirmed `qty_done` via XML-RPC and update DO status to Done
- **PDF session:** a separate `requests.Session()` authenticated via `/web/session/authenticate` is maintained for PDF downloads — the XML-RPC `uid` cannot access `/report/pdf/` endpoints

### Odoo models in scope
| Model | Purpose | Key fields |
|---|---|---|
| `res.partner` | Vendor profile sync | `id`, `name`, `email` (initial account creation only), `company_name`, `phone`, `mobile`, `vat` (Tax ID), `supplier_rank` |
| `purchase.order` | PO list, detail, confirmation, rejection | `name`, `partner_id`, `state`, `date_order`, `date_planned` (Expected Arrival), `amount_total`, `order_line`, `picking_ids`, `user_id` (PO creator for email) — `button_confirm` or `button_cancel` called on vendor action |
| `purchase.order.line` | PO line items | `product_id`, `name`, `product_qty`, `qty_received`, `price_unit`, `product_uom` (UoM for this line) |
| `product.product` | Product info for DO PDF | `id`, `name`, `barcode` |
| `stock.picking` | Receipt header | `name`, `state`, `scheduled_date`, `origin`, `move_line_ids` |
| `stock.move.line` | Receipt line items | `product_id`, `product_uom_qty`, `qty_done`, `lot_id`, `state` |
| `stock.warehouse` | Store ID for DO PDF | `code` (short name, used as Store ID on printed DO) |

### Odoo 16 CE field name gotchas
- `stock.move.line.qty_done` → renamed to `quantity` in Odoo 17+
- `stock.picking.move_lines` → renamed to `move_ids` in Odoo 17+
- Always specify explicit field lists in `search_read` — reading all fields on Odoo 16 can trigger serialization errors on computed fields like `tax_totals`

### Data filtering rules (applied at backend, not frontend)
- POs: only `state IN ('sent', 'purchase', 'cancel')` are returned — `draft` and `done` are excluded
- Receipts: only `state IN ('assigned', 'done')` are returned
- All queries are scoped by `partner_id` via VendorScopedOdooClient

---

## Phase 2 — Auth System
**Effort: 1 day**

### Objectives
- Define the `vendor_users`, `admin_users`, `password_reset_tokens`, `delivery_orders`, `delivery_order_lines`, `return_notes`, `return_note_lines`, and `audit_log` database tables
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
| `odoo_partner_id` | integer, unique | **login identifier** — `res.partner.id` from Odoo, permanent, never changes |
| `email` | varchar | for email notifications only — synced from Odoo `res.partner.email` on account creation, never used for login |
| `hashed_password` | varchar | NULL until invite accepted |
| `full_name` | varchar | synced from Odoo — contact person name |
| `company_name` | varchar | synced from Odoo |
| `phone` | varchar | synced from Odoo — mobile/phone number |
| `tax_id` | varchar | synced from Odoo `res.partner.vat` — for DO PDF |
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

**`delivery_orders`** (lock fields merged — no separate `do_locks` table)
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal DO ID |
| `po_odoo_id` | integer, unique | Odoo `purchase.order.id` this DO belongs to |
| `picking_id` | integer | Odoo `stock.picking.id` (linked Receipt) |
| `vendor_id` | FK → vendor_users | vendor who owns this DO |
| `delivery_date` | date | vendor-set planned delivery date |
| `status` | varchar | `draft`, `signed`, `done`, or `cancelled` |
| `signature_path` | varchar | NULL until signed; server-side path to stored PNG |
| `pdf_path` | varchar | NULL until signed; server-side path to signed DO PDF |
| `vendor_comment` | text | optional free-text note submitted by vendor at signing |
| `signed_at` | timestamp | NULL until signed |
| `unlocked_at` | timestamp | NULL unless admin unlocked |
| `unlocked_by` | FK → admin_users | admin who unlocked, NULL if not unlocked |
| `created_at` | timestamp | when DO was auto-created (on PO confirm) |
| `updated_at` | timestamp | last edit by vendor |

**`delivery_order_lines`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `do_id` | FK → delivery_orders | parent DO |
| `line_number` | integer | sequential row number |
| `product_id` | integer | Odoo `product.product.id` |
| `product_barcode` | varchar | cached product barcode from Odoo |
| `product_name` | varchar | cached product name |
| `uom` | varchar | unit of measure, inherited from PO line (e.g., Thùng 12 Chai, Kg). Cannot be changed by vendor |
| `ordered_qty` | decimal | quantity from PO line (read-only reference) |
| `delivery_qty` | decimal | vendor-entered delivery quantity (must be <= ordered_qty) |
| `received_qty` | decimal | NULL until store confirms receipt; final qty_done from Odoo |

**`return_notes`** (same structure as delivery_orders, for returns)
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal RN ID |
| `rpo_odoo_id` | integer, unique | Odoo `purchase.order.id` (the RPO) |
| `picking_id` | integer | Odoo `stock.picking.id` (linked return receipt) |
| `vendor_id` | FK → vendor_users | vendor who owns this RN |
| `pickup_date` | date | vendor-set date to collect returned goods |
| `status` | varchar | `draft`, `signed`, or `done` (no cancelled — vendor cannot reject RPO) |
| `signature_path` | varchar | NULL until signed |
| `pdf_path` | varchar | NULL until signed |
| `signed_at` | timestamp | NULL until signed |
| `created_at` | timestamp | |
| `updated_at` | timestamp | |

**`return_note_lines`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `rn_id` | FK → return_notes | parent RN |
| `line_number` | integer | sequential row number |
| `product_id` | integer | Odoo `product.product.id` |
| `product_barcode` | varchar | cached product barcode |
| `product_name` | varchar | cached product name |
| `uom` | varchar | unit of measure from RPO line |
| `return_qty` | decimal | quantity being returned (read-only — set by store, vendor cannot change) |

**`audit_log`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `actor_type` | varchar | `vendor` or `admin` |
| `actor_id` | integer | `vendor_users.id` or `admin_users.id` |
| `action` | varchar | one of: `login`, `po_confirm`, `po_reject`, `po_auto_cancel`, `do_update`, `do_sign`, `do_unlock`, `rn_confirm_sign`, `receipt_validated` |
| `target_model` | varchar | e.g. `receipt`, `vendor` |
| `target_id` | integer | e.g. `picking_id`, `partner_id` |
| `detail` | jsonb | action-specific metadata (e.g. which lines were updated, old/new qty) |
| `created_at` | timestamp | when the action occurred |
| `ip_address` | varchar | client IP for login events |

### Auth endpoints
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/login` | Vendor ID (integer) + password → access + refresh tokens (role: `vendor`) |
| POST | `/api/admin/auth/login` | Username + password → access + refresh tokens (role: `admin`) |
| POST | `/api/auth/refresh` | Refresh token → new access token (preserves role) |
| POST | `/api/auth/set-password` | First-time invite or password reset via token (vendors only) |
| POST | `/api/auth/forgot-password` | Sends reset email via AWS SES by Vendor ID |
| POST | `/api/auth/logout` | Blacklists refresh token in Redis |

### JWT role claim
The JWT access token carries a `role` field (`vendor` or `admin`). All admin-only endpoints check for `role == 'admin'` via a dedicated FastAPI dependency. Vendor endpoints check for `role == 'vendor'`. A vendor token cannot access admin routes, and an admin token cannot be used to impersonate a vendor.

### Security considerations
- **Timing-safe login:** always run `bcrypt.verify` even when the Vendor ID does not exist
- **Consistent error messages:** same message for unknown Vendor ID and wrong password — never reveal whether the ID exists
- **Token blacklist:** on logout, refresh token is added to a Redis set with TTL matching remaining token lifetime
- **Invite token expiry:** 24 hours. If expired, vendor must request a new one via forgot-password

---

## Phase 3 — Core API Endpoints
**Effort: 2–3 days**

### Objectives
- Implement all data endpoints with vendor-scoped Odoo queries
- Enforce DO locking logic server-side
- Implement the PDF generation and signing pipeline
- Implement webhook receiver for Odoo receipt validation
- Implement all email notification triggers via AWS SES

### Full endpoint list
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/vendors/me` | Current vendor profile |
| PATCH | `/api/vendors/me/language` | Update preferred language (`vi` or `en`) |
| GET | `/api/purchase-orders` | Paginated PO list with portal statuses, filterable by PO number and date range |
| GET | `/api/purchase-orders/{id}` | PO detail with order lines + linked DO + receipt comparison |
| POST | `/api/purchase-orders/{id}/confirm` | Confirm a Sent RFQ — calls `button_confirm` on Odoo; returns 409 if PO is not in `sent` state |
| POST | `/api/purchase-orders/{id}/reject` | Reject a Sent RFQ — calls `button_cancel` on Odoo + email to PO creator; returns 409 if PO is not in `sent` state |
| GET | `/api/delivery-orders/{do_id}` | DO detail with product lines, delivery date, delivery qty, and received qty (when Done) |
| PATCH | `/api/delivery-orders/{do_id}/lines` | Update quantities + delivery date on the DO (blocked if signed/locked); qty must be <= ordered qty |
| POST | `/api/delivery-orders/{do_id}/sign` | Submit signature PNG + optional comment, lock DO, push to Odoo Receipt, generate PDF |
| GET | `/api/delivery-orders/{do_id}/pdf` | Download signed DO PDF (available post-signature, can be called multiple times for printing) |
| GET | `/api/return-orders` | Paginated RPO list for current vendor |
| GET | `/api/return-orders/{id}` | RPO detail with return lines and linked Return Note |
| GET | `/api/return-notes/{rn_id}` | Return Note detail with product lines and pickup date |
| PATCH | `/api/return-notes/{rn_id}/pickup-date` | Set or update pickup date on the RN (blocked if signed) |
| POST | `/api/return-notes/{rn_id}/confirm-and-sign` | Confirm RPO + submit signature PNG in one step. Locks RN, generates PDF |
| GET | `/api/return-notes/{rn_id}/pdf` | Download signed RN PDF (same format as DO PDF) |
| GET | `/api/export` | Export data as PDF or CSV. Query params: `format` (pdf/csv), `type` (individual/summary), `ids` (for bulk), `date_from`, `date_to`. Includes POs, DOs, RPOs, RNs |
| POST | `/api/webhooks/odoo/receipt-validated` | Webhook receiver: Odoo notifies portal when Receipt is validated — triggers DO status → Done + vendor email |

### Admin-only endpoints
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/admin/vendors` | List all vendor accounts with status, last login, signed DO count, pending DO count |
| GET | `/api/admin/vendors/{partner_id}` | Drill into a specific vendor — their POs and DOs (read-only) |
| PATCH | `/api/admin/vendors/{partner_id}/deactivate` | Deactivate a vendor account |
| PATCH | `/api/admin/vendors/{partner_id}/reactivate` | Reactivate a vendor account |
| POST | `/api/admin/sync` | Manually trigger the Odoo partner sync job |
| GET | `/api/admin/sync/status` | Return last sync time, number of vendors synced, list of skipped vendors (no email) |
| GET | `/api/admin/delivery-orders/{do_id}/pdf` | Download any vendor's signed DO PDF |
| POST | `/api/admin/delivery-orders/{do_id}/unlock` | Remove the DO lock — sends email to vendor, vendor can update and re-sign |
| GET | `/api/admin/audit-log` | Paginated audit log view, filterable by actor, action type, and date range |

All admin endpoints require `role == 'admin'` in the JWT. They are prefixed with `/api/admin/` to make the distinction explicit and apply a separate rate limiting policy.

### PO list filtering
The `GET /api/purchase-orders` endpoint accepts the following optional query parameters:
- `q` — partial match on PO number (e.g. `PO004`)
- `date_from` — filter POs with `date_order >= date_from`
- `date_to` — filter POs with `date_order <= date_to`
- `page` and `page_size` — pagination (default page size: 20)

Filtering is applied server-side in the Odoo domain filter, not in the frontend.

### DO locking and unlocking approach
- When `POST /api/delivery-orders/{do_id}/sign` is called, the backend sets `status = 'signed'`, stores signature/PDF paths, and sets `signed_at` on the `delivery_orders` row
- Any subsequent `PATCH /lines` call on a signed DO returns HTTP 423 (Locked)
- The DO detail endpoint returns `status` — the frontend disables all editing when status is not `draft`
- On signing, the backend pushes delivery date + set quantities to the Odoo Receipt via XML-RPC
- **Unlocking:** a portal admin calls `POST /api/admin/delivery-orders/{do_id}/unlock`, which resets status to `draft`, sets `unlocked_at` + `unlocked_by`, clears signature/PDF paths, sends an email to the vendor, and logs the action. The vendor can then update quantities and re-sign.

### Validation approach (no portal-side `button_validate`)
Since the store decides on backorders in Odoo, the portal does **not** call `button_validate`. The vendor's role is:
1. Edit the DO: delivery date + quantities (qty <= ordered qty, multiple saves allowed)
2. Sign the DO (locks the portal record, pushes set quantities to Odoo Receipt)

The Odoo picking remains in `assigned` state after the vendor signs — the store sees the set quantities and validates in Odoo at their discretion. Only when the store confirms the Receipt does `qty_done` get finalized.

### Receipt validated (Odoo → Portal)
When the store confirms a Receipt in Odoo, the portal is notified (sync mechanism TBD — pending team confirmation):
1. Portal receives notification with `picking_id`
2. Portal reads the confirmed `qty_done` values from Odoo via XML-RPC
3. DO status on portal changes to **Done** — final received amounts are stored on DO lines
4. Portal sends email to vendor confirming receipt. If any quantities differ between DO and receipt, the email includes a discrepancy alert
5. Vendor can log into portal to view details and export PDF/CSV

### Audit logging approach
Every key action is written to the `audit_log` table synchronously within the same request. The logged action types are:

| Action | Trigger | Detail stored |
|---|---|---|
| `login` | Successful vendor or admin login | actor type, actor ID, IP address, timestamp |
| `po_confirm` | `POST /purchase-orders/:id/confirm` | PO ID, PO name, previous state (`sent`), new state (`purchase`) |
| `po_reject` | `POST /purchase-orders/:id/reject` | PO ID, PO name, previous state (`sent`), new state (`cancel`) |
| `po_auto_cancel` | Scheduled job (7 days past Expected Arrival) | PO ID, PO name, Expected Arrival date, days overdue |
| `do_update` | `PATCH /delivery-orders/:id/lines` | DO ID, list of lines with old and new quantities |
| `do_sign` | `POST /delivery-orders/:id/sign` | DO ID, vendor comment (if any), PDF path, signature path |
| `do_unlock` | `POST /admin/delivery-orders/:id/unlock` | DO ID, admin who unlocked |
| `receipt_validated` | Odoo → Portal notification | picking ID, DO status → Done, any quantity differences noted |

The audit log is viewable by admins via `GET /api/admin/audit-log`, filterable by actor, action type, and date range, paginated at 50 rows per page. It is never editable or deletable through the portal.

### Email notifications (AWS SES)

| Trigger | Recipient | Content |
|---|---|---|
| New vendor synced | Vendor | Welcome email with Vendor ID (integer) + set-password link |
| Forgot password requested | Vendor | Reset link (24h expiry) |
| Vendor rejects RFQ | PO creator (store staff) | PO rejected + cancelled in Odoo |
| PO auto-cancelled (7 days) | Vendor + PO creator | PO auto-cancelled — no response within 7 days past Expected Arrival |
| Store confirms receipt | Vendor | Receipt confirmed. Alerts if any quantities differ between DO and receipt |
| DO unlocked by admin | Vendor | DO has been unlocked for re-editing |
| RPO created by store | Vendor | Sent by Odoo natively (Send by Email button) — not via AWS SES |

**Not emailed:** Vendor confirms PO (data pushed to Odoo in real-time), Vendor signs DO/RN (data pushed to Odoo automatically).

All portal-sent emails use AWS SES in the vendor's `preferred_language` for vendor-facing emails, and in Vietnamese for store-facing emails. Store email goes to the PO creator's email (from Odoo `purchase.order.user_id`), not a generic inbox. RPO notification is sent by Odoo itself, not by the portal.

### DO / RN PDF pipeline approach
1. Check lock record — PDF only generated after signature is submitted
2. Generate the document as HTML rendered to PDF with WeasyPrint, **in Vietnamese**
3. Store the PDF on the server filesystem (retained for 24 months, matching PO retention)
4. Return the PDF as a binary response with `Content-Disposition: attachment` — vendor can call this endpoint multiple times to print

**Note:** The Return Note (RN) PDF uses the **same layout and content** as the DO PDF, with the title changed to "Bien Ban Tra Hang" and the delivery date replaced by the pickup date.

### DO PDF content specification
The printed DO PDF includes the following sections, all in Vietnamese:

**Header:**
- PO number displayed as a **Code128 barcode** (scannable by store's handheld device)
- Vendor ID / Mã NCC (`res.partner.id`)
- Vendor Tax ID / Mã số thuế (`res.partner.vat`)
- Vendor mobile phone / Số điện thoại (`res.partner.phone` or `mobile`)
- Vendor contact email / Email liên hệ (`res.partner.email`)
- Store ID / Mã cửa hàng (`stock.warehouse.code` — the warehouse short name)
- PO Confirmation Date / Ngày xác nhận đơn hàng
- Delivery Date / Ngày giao hàng (the single date set by vendor on the DO)

**Product table columns:**

| # | Column (Vietnamese) | Column (English) | Description |
|---|---|---|---|
| 1 | Số thứ tự | Line number | Sequential row number |
| 2 | Mã vạch | Barcode | Product barcode |
| 3 | Tên sản phẩm | Product name | Product description |
| 4 | Đơn vị tính | UoM | Unit of measure (inherited from PO, e.g., Thùng 12 Chai, Kg) |
| 5 | Số lượng giao | Delivery qty | Quantity vendor plans to deliver |
| 6 | Số lượng thực nhận | Received qty | Left blank — filled by store on paper |
| 8 | SL chênh lệch | Discrepancy qty | Left blank — for store to note differences |
| 9 | Ghi chú | Notes | Left blank — for store notes |

**Footer:**
- Vendor's digital signature (PNG embedded)
- Vendor comment (if provided at signing)
- Timestamp of signature
- Blank signature space for store's physical signature

---

## Phase 4 — React Frontend
**Effort: 3–4 days**

### Objectives
- Build all portal pages with clean, mobile-friendly UI
- Implement bilingual support (Vietnamese + English, user-switchable)
- Implement the `fetch`-based API client with automatic JWT refresh
- Integrate `signature_pad.js` for signature capture
- Implement the DO edit form with quantity validation (qty <= ordered) and lock state handling

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
| `/login` | Login | Both | Vendor ID (integer) + password + language toggle |
| `/admin/login` | Admin Login | Admin | Username + password (separate login page) |
| `/set-password` | Set Password | Vendor | First-time invite and forgot-password reset |
| `/forgot-password` | Forgot Password | Vendor | Enter Vendor ID to receive reset link |
| `/dashboard` | Dashboard | Vendor | PO list with search/filter, status badges, checkbox selection + "Export" button (PDF individual/summary or CSV) |
| `/purchase-orders/:id` | PO Detail | Vendor | Order lines + linked DO + receipt comparison (vendor qty vs store qty) |
| `/delivery-orders/:id` | DO Detail | Vendor | Product lines with editable delivery qty + single delivery date (locked if signed) |
| `/delivery-orders/:id/sign` | Sign DO | Vendor | Signature pad + comment field + final confirmation step |
| `/delivery-orders/:id/pdf` | DO PDF | Vendor | Download/print signed DO PDF (can print multiple times) |
| `/returns` | Returns List | Vendor | RPO list with search/filter, status badges |
| `/return-orders/:id` | RPO Detail | Vendor | Return lines + linked Return Note |
| `/return-notes/:id` | RN Detail | Vendor | Return product lines + pickup date picker (qty read-only) + Confirm & Sign |
| `/return-notes/:id/pdf` | RN PDF | Vendor | Download/print signed RN PDF |
| `/profile` | Profile | Vendor | Read-only vendor profile + language preference |
| `/admin/dashboard` | Admin Dashboard | Admin | Summary stats: vendor counts, pending DOs/RNs, sync status |
| `/admin/vendors` | Vendor List | Admin | All vendor accounts with status, last login, DO/RN counts |
| `/admin/vendors/:id` | Vendor Detail | Admin | Specific vendor's POs, DOs, RPOs, RNs (read-only drill-down) |
| `/admin/sync` | Sync Status | Admin | Last sync time, skipped vendors, manual trigger button |

### Admin dashboard content
The admin dashboard shows the following at a glance:
- **Vendor summary:** total vendors, active count, inactive count
- **Vendor table:** sortable list showing vendor name, Vendor ID, account status (active/inactive), last login date, number of signed DOs, number of pending DOs (Draft, not yet signed)
- **Sync status panel:** last sync timestamp, number of vendors synced, number of vendors skipped (no email), manual "Run Sync Now" button
- Clicking any vendor row navigates to `/admin/vendors/:id` — a read-only view of that vendor's POs and DOs, with a download button for any signed DO PDF and an unlock button on locked DOs

### Admin navigation
Admin users see the same top navigation bar as vendors, with additional items: **Vendors**, **Sync Status**, **Audit Log**. The language toggle is present — admin section supports both Vietnamese and English. Admin routes are protected — a vendor JWT cannot access `/admin/*` routes.

### HTTP client approach (native fetch)
- A single `apiFetch(path, options)` wrapper handles all API calls
- Attaches Bearer token from `localStorage` on every request
- On 401 response: calls `/api/auth/refresh`, stores new access token, retries once
- On refresh failure: clears storage, redirects to `/login`
- No direct `fetch` calls in components — all calls go through this wrapper

### State management approach
- **TanStack Query** for all server state — PO list, DO detail, profile
- **React local state** (`useState`) for form inputs, signature state, UI toggles
- No global state manager needed at this scope

### Responsive design approach
- The portal is fully responsive — works on desktop and mobile equally
- Tailwind CSS utility classes for layout (or equivalent CSS framework)
- Signature pad canvas resizes to fit the screen width on mobile
- Tables on mobile collapse to card-style layouts for readability

### Vendor dashboard summary
Above the PO list, the vendor dashboard shows summary cards:
- **Waiting** — POs awaiting vendor confirmation
- **Confirmed** — POs confirmed, DOs in various states
- **Cancelled** — POs that have been cancelled

### PO list search & filter UI
- Search bar for PO number (partial match, debounced 300ms before API call)
- Date range pickers for `date_from` and `date_to`
- Status badge on each PO row (Waiting / Confirmed / Cancelled) with colour coding
- DO status shown alongside PO status (Draft / Signed / Done / Cancelled)
- Pagination controls (previous / next), page size fixed at 20

### DO detail — visible fields per product line
| Field | Vietnamese | Source |
|---|---|---|
| Line number | Số thứ tự | Sequential row number |
| Product barcode | Mã vạch | from Odoo `product.product` → `barcode` |
| Product name | Tên sản phẩm | `delivery_order_lines` → `product_name` |
| UoM (read-only) | Đơn vị tính | `delivery_order_lines` → `uom` (inherited from PO line, cannot be changed) |
| Ordered quantity (read-only) | SL đặt hàng | `delivery_order_lines` → `ordered_qty` (from PO line) |
| Delivery quantity (editable) | Số lượng giao | `delivery_order_lines` → `delivery_qty` (must be <= ordered_qty) |
| Received quantity (read-only) | Số lượng thực nhận | `delivery_order_lines` → `received_qty` (NULL until store confirms; shows store's final qty_done) |
| Unit price | Đơn giá | from Odoo `purchase.order.line` → `price_unit` |
| Subtotal | Thành tiền | Calculated: unit price x delivery qty (frontend only, not stored) |

### DO detail behaviour
- Delivery date picker at the top of the DO form
- Each product line shows all fields above; delivery qty is the only editable field (plus delivery date)
- Validation: delivery qty must be <= ordered qty — frontend and backend both enforce this
- "Save" button submits `PATCH /delivery-orders/:id/lines` — can be pressed multiple times before signing
- If `locked: true` is returned by the API, all inputs are disabled and a lock notice is shown
- A "Sign DO" button at the bottom is enabled only when delivery date is set and all delivery qty values are > 0
- Once signed, the page shows a read-only summary with a "Print DO" button
- After store confirms receipt, the `received_qty` column is populated — vendor sees both their delivery qty and the store's received qty side by side

### PO detail — DO and receipt view
The PO detail page shows the linked DO with its status. When the DO is Done (store confirmed receipt):
- **Vendor DO quantities** — what the vendor planned to deliver
- **Store received quantities** — what the store actually confirmed in Odoo
- Per-line comparison, highlighting any differences between delivery and received quantities

### PDF history page
- Lists all DOs the vendor has signed, in reverse chronological order
- Each row shows: PO number, DO reference, delivery date, date signed, vendor comment (if any), download button
- Download calls `GET /api/delivery-orders/{do_id}/pdf` — PDFs are retained for 24 months (matching PO retention)

### Signature and comment capture
- `signature_pad.js` renders on an HTML5 canvas (resizes for mobile)
- Optional free-text comment field above the signature pad — vendor can describe delivery conditions or notes
- "Clear" button resets the canvas only
- "Confirm & Submit" sends both the signature PNG (base64) and the comment text to `POST /api/delivery-orders/:id/sign`
- Pad and comment field are disabled after successful submission
- A loading state is shown while the PDF is being generated server-side

### Key UX considerations
- Login field label: **"Mã nhà cung cấp / Vendor ID"** with helper text explaining it was provided in the welcome email (must be a number input)
- DO lines show ordered qty, delivery qty, and received qty (when available) side by side for easy comparison
- Waiting POs show "Confirm PO" and "Reject" buttons prominently
- Lock notice on signed DOs: "Phiếu giao hàng đã được ký / Delivery Order has been signed"
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
- PDF files stored on server filesystem (24-month retention) — include in VM backup strategy
- Scheduled cleanup job removes POs and associated DOs, PDFs older than 24 months

### Go-live checklist
- [ ] Odoo service account API key generated and connectivity tested from portal VM
- [ ] Firewall: portal VM → Odoo VM port 8069 open
- [ ] AWS SES sender domain verified, IAM credentials configured
- [ ] Verify PO creator email field is accessible via Odoo XML-RPC (`purchase.order` → `user_id` → `email`)
- [ ] `ADMIN_INITIAL_PASSWORD` set and first admin account seeded
- [ ] All production secrets set in Docker secrets files
- [ ] Upstream load balancer / proxy configured to forward HTTPS traffic
- [ ] Partner sync job run manually once — initial vendor accounts created
- [ ] Odoo → Portal sync mechanism configured and tested (webhook, polling, or batch — TBD)
- [ ] At least one test vendor: invite received → password set → login → dashboard visible → PO confirmed → DO edited + signed → DO PDF printed → delivered to store → store confirms receipt in Odoo → DO status Done → vendor email received
- [ ] Test RFQ rejection: vendor rejects → Odoo cancels → store email received
- [ ] Notification emails received by vendor and store
- [ ] Locked DO confirmed uneditable by vendor
- [ ] Admin can view vendor, download DO PDF, unlock DO (vendor email received)
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

1. **Vendor login uses Vendor ID (integer), not email** — frontend login field must be a `number` input; welcome email must clearly state the Vendor ID; `odoo_partner_id` is the lookup key in `vendor_users`
2. **Admin login is separate** — admin uses username (not Vendor ID) on a separate login page `/admin/login`; admin JWT carries `role: admin`
3. **Profile sync never touches `hashed_password`** — only `full_name`, `company_name`, `phone`, `tax_id` are overwritten by the sync job; `email` is synced on initial account creation only, never again
3. **Portal does not call `button_validate`** — vendor edits DO and signs; Odoo validation is done by the store. The Odoo picking stays in `assigned` state after the vendor signs
4. **DO locking is portal-side only** — lock state lives in the `delivery_orders` table; Odoo is not aware of it. Admins unlock via the portal admin section
5. **Admin can unlock DOs — vendor cannot** — the unlock endpoint is admin-only; vendors have no self-service unlock
6. **PDF requires a separate HTTP session** — the XML-RPC connection cannot fetch `/report/pdf/` endpoints; a distinct `requests.Session()` authenticated via `/web/session/authenticate` is required
7. **PDFs are stored for 24 months** — a scheduled cleanup job enforces this; ensure the server filesystem has adequate space and is included in the VM backup strategy
8. **Vendor comment is optional but must be stored** — even if empty, the `vendor_comment` field in `delivery_orders` must be explicitly stored (empty string, not NULL) to distinguish "no comment provided" from a data error
9. **Audit log is append-only** — no delete or update endpoints for `audit_log`; write failures should be logged but must not cause the parent action to fail (non-blocking)
10. **Odoo 16 field names** — `qty_done` on `stock.move.line` (not `quantity`), `move_lines` on `stock.picking` (not `move_ids`)
12. **Partners with no email are skipped** — sync job must log these; they cannot receive the welcome email and must be handled manually (Vendor ID is still assigned by Odoo, email is only needed to deliver the invite)
13. **Timing-safe authentication** — bcrypt verification must always run, even for unknown Vendor IDs, to prevent user enumeration via response timing
13. **No axios** — all frontend HTTP calls use native `fetch` wrapped in a single `apiFetch` utility with automatic JWT refresh
14. **i18n covers both vendor and admin UI** — all portal pages including admin section support Vietnamese and English; vendor-facing emails follow `preferred_language`; store-facing emails are in Vietnamese
15. **TLS is upstream** — the portal Nginx does not manage certificates; ensure the upstream proxy passes `X-Forwarded-Proto` and `X-Real-IP` headers correctly to the FastAPI backend
16. **`button_confirm` is a write action on Odoo** — unlike all other vendor-facing data calls which are read-only, PO confirmation calls `execute_kw` with `button_confirm` on `purchase.order`. The Odoo service account must have write permission on `purchase.order`. Verify this permission explicitly — read-only service accounts will silently fail or return an access error
17. **Guard PO confirmation against double-submit** — the confirm endpoint must re-read the PO state from Odoo before calling `button_confirm`; if the state is already `purchase`, return 409 without calling Odoo again. The frontend must also disable the "Confirm PO" button immediately on click to prevent duplicate calls
