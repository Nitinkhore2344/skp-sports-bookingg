# SKP Sports: Backend API (Node.js + Express + PostgreSQL)

REST API for the SKP Sports pickleball booking website: customer accounts, court availability,
server-side pricing, booking requests with admin approval, blog, contact form and a full admin API.

> **Status:** Phase 1 (backend) is complete and covered by 18 integration tests that run against a real
> PostgreSQL database. The React front-end still uses demo data and must be switched to this API (see section 11).
> **Not included, needs client input or credentials:** online payments, real SMTP email, production hosting.
> All sample data (courts, prices, hours, tax 0%, timezone `Asia/Manila`, currency `PHP`) is a **placeholder**.

## 1. Requirements
Node.js 20+ and PostgreSQL 14+ (needs the `btree_gist` and `citext` extensions, which ship with standard PostgreSQL contrib).

## 2. Install and run
```bash
npm install
cp .env.example .env            # then edit DATABASE_URL, ADMIN_EMAIL, ADMIN_PASSWORD, CLIENT_ORIGIN
createdb skp                    # or create the database in your hosting panel
npm run migrate                 # creates all tables and constraints
npm run seed                    # roles, settings, first admin, sample courts/blog/demo customer
npm start                       # http://localhost:4000   (npm run dev to auto-reload)
```
Health check: `GET /api/health`. Sample customer: `customer@example.com` / `Customer-Demo-123`
(delete it before going live). Change `ADMIN_PASSWORD` before seeding anywhere public.

## 3. Admin setup
1. Seed creates one **super_admin** from `ADMIN_EMAIL` / `ADMIN_PASSWORD`. Sign in with `POST /api/auth/login`.
2. Create more staff with `POST /api/admin/staff` (`role`: `manager`, `staff`, `editor`, `super_admin`; password 10+ characters).
3. Replace the placeholders: `PUT /api/admin/settings` (business name, timezone, currency, tax, notice, cancellation window, policy texts, contact),
   then `/api/admin/categories`, `/courts`, `/hours`, `/pricing-rules`, `/addons`, `/faqs`.

| Role | Can do |
|---|---|
| super_admin | everything, including settings, staff, audit log |
| manager | bookings, courts, pricing, content, customers (read), CSV export |
| staff | review, approve, reject, cancel and reschedule bookings; view customers |
| editor | blog, FAQs, site content, contact inbox |
| customer | own account and own bookings only |

## 4. How booking works
`customer selects court/date/time -> server validates and prices -> saved as PENDING -> admin approves or rejects (reason required) -> customer emailed`

* **Pending requests never block a slot.** Only **approved** bookings do, so several people may request the same time and the first approval wins.
* **Double-booking protection, three layers:** (1) availability is rechecked when a request is created; (2) approval runs in a transaction that locks the court row and rechecks;
  (3) a PostgreSQL exclusion constraint, `no_overlapping_approved`, makes overlapping approved bookings on one court impossible. Conflicts return HTTP 409.
* **Buffers** (per court) extend each booking's hold. **Blocks** (maintenance or events, per court or all courts) hide slots, and any overlapping bookings are reported when the block is created.
* **Pricing** is calculated only on the server: base rate, peak and special-day rules (special beats peak; a court rule beats a global rule), percentage discounts, add-ons (per booking, player or hour) and tax. The price is snapshotted on the booking.
* **Times** are stored as UTC `timestamptz` and interpreted in the business timezone setting.
* **Cancellation:** pending bookings can always be cancelled; approved ones only if at least `cancel_window_hours` remain. Paid bookings create a pending `refunds` row. Refund processing is manual until a gateway exists.
* Every status change is stored in `booking_status_history`, and admin actions go to `audit_logs`. Approved bookings are marked completed automatically after they end.

## 5. API summary (JSON, session cookie `skp_session`)
**Public** (`/api/public`): `GET settings`, `courts`, `availability?courtId&date&duration`, `POST quote`, `GET blog?q&category&tag&page&pageSize`, `blog/:slug`, `blog-categories`, `content/:key`, `POST contact`.

**Auth** (`/api/auth`): `POST register | login | logout | change-password`, `GET me`.

**Customer** (`/api/bookings`, login required): `POST /`, `GET /`, `GET /:ref`, `POST /:ref/cancel`.
Create body: `{courtId, date:"YYYY-MM-DD", time:"HH:mm", duration, players, addons:[ids], customerName, customerEmail, customerPhone:"+63...", notes, acceptTerms:true}`.

**Admin** (`/api/admin`, permission based):
`dashboard`; `bookings` (filters `status, courtId, categoryId, from, to, q, page`); `bookings/export.csv`;
`bookings/:id` with `approve`, `reject {reason}`, `cancel {reason}`, `reschedule {date,time,duration?}`, `complete`;
CRUD for `categories, courts, hours, pricing-rules, addons, faqs, blog-categories, blocks`; `blog` (posts); `customers` (with deactivate);
`staff`, `roles`, `settings`, `content/:key`, `contact`, `audit`, `notifications`.

Errors look like `{ "error": "message", "fields": { "field": ["problem"] } }` with status 400, 401, 403, 404, 409 or 429.

## 6. Security
Passwords are hashed with scrypt. Sessions use opaque random tokens (only the SHA-256 is stored) in HttpOnly, SameSite=Lax cookies (`Secure` in production).
Writes check the request Origin (CSRF). Every input is validated with zod, SQL is parameterised, helmet sets security headers, and login, register, contact and the general API are rate limited.
Customers can only read their own bookings. CSV export neutralises spreadsheet formulas. Logs never include request bodies or secrets.
Behind a proxy or CDN set `TRUST_PROXY=1` so rate limiting sees real client IPs.
Arrange **backups** with your host (daily `pg_dump` or managed-database snapshots) and add error monitoring in production.

## 7. Email
Without `SMTP_HOST`, emails are not sent; they are recorded in `notifications` with status `logged`. Add SMTP credentials (Resend, Postmark, SES, Gmail SMTP and so on) and set `ADMIN_NOTIFY_EMAIL`.
Emails sent: request received, approved, rejected (with reason), cancelled, rescheduled, plus admin alerts for new bookings and contact messages.

## 8. Payments (not implemented)
The tables `payments` and `refunds` and the column `bookings.payment_status` exist, and `payments_enabled` is false. To add payments you need a client decision, a gateway account
(for example PayMongo or Xendit) and confirmed currency and tax. When added, mark a payment as paid **only** from a verified gateway webhook, never from the browser.

## 9. Deploy
1. Create a managed PostgreSQL database (Neon, Supabase, RDS, Railway) and set `DATABASE_URL` (add `?sslmode=require` if needed).
2. Host the API on any Node host (Render, Railway, Fly.io, or a VPS with pm2 or systemd). Set the variables from `.env.example`, with `NODE_ENV=production` and `CLIENT_ORIGIN=https://your-site`.
3. Run `npm run migrate` on every deploy and `npm run seed` once. Serve over HTTPS.
   If the site and API are on different domains, cookies need a shared registrable domain (for example `app.x.com` and `api.x.com`),
   or use a reverse proxy that serves `/api` on the site's own domain (recommended).

## 10. Testing
```bash
createdb skp_test
DATABASE_URL=postgres://user:pass@localhost:5432/skp_test NODE_ENV=test npm test
```
**Warning:** the test run drops and recreates the `public` schema. It refuses to run unless the database name ends in `_test`.

Automated (18 tests, all passing): registration, login, logout; CSRF origin check; availability slots; pricing (base, peak, add-ons); input validation; pending booking and reference;
owner-only access; pending requests not blocking; concurrent approval race (exactly one wins); database-level overlap rejection; rejection reason and email; buffers; blocks;
cancellation window; reschedule with repricing; role-based access; customer admin and deactivation; dashboard and CSV; blog publish, search, pagination and related posts; contact form.

Manual checklist before launch:
- Real SMTP email delivery.
- Mobile and desktop layouts after the front-end is connected.
- Real branding and content from the client.
- Privacy policy, terms and cancellation text reviewed by the client.
- Payment flow in the gateway's sandbox, if payments are enabled.
- A backup and restore drill.
- A load test of the booking endpoint.

## 11. Connecting the front-end
The React demo keeps its data access in one small `api`/`store` layer. Replace it with `fetch('/api/...', { credentials: 'include' })` calls to the endpoints above:
courts, availability and quote from `/api/public`, bookings from `/api/bookings`, and login from `/api/auth`.
Customers must then register and sign in with a password, and the booking page should call `/api/public/availability` instead of the demo `busy()` function.
