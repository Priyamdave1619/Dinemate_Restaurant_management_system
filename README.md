# DineMate — Restaurant Ordering & Management Platform

A multi-tenant restaurant platform: one shared backend + database, with a
separate frontend for every role — platform owner, restaurant owner/manager,
kitchen waiter, and the dining customer. Web portals are Next.js; the
restaurant and waiter apps also ship as native Flutter apps.

---

## 1. Problem Statement

Small and mid-size restaurants and food courts typically stitch together
several disconnected tools to run day-to-day operations:

- **Order-taking is manual or fragmented** — paper KOTs (kitchen order
  tickets), verbal shouting between waiter and kitchen, or a generic
  QR-menu tool that only shows a static menu with no live order/status
  flow back to the table.
- **No single source of truth** — the menu a customer sees, the menu the
  waiter sells, and the menu the kitchen prepares from can drift out of
  sync when they live in separate spreadsheets or apps.
- **No real-time visibility** for the owner/manager into what's
  happening on the floor right now — which tables are occupied, which
  orders are pending/preparing/ready, and how staff are performing.
- **Staff and role management is an afterthought** — most low-cost
  QR-ordering tools give one shared login instead of proper
  owner / manager / waiter roles with scoped permissions.
- **Multi-restaurant/franchise operators** have no central place to
  onboard new outlets, monitor them, or enforce subscription/billing —
  each outlet ends up as an isolated silo.
- **Billing, reporting, and payments** are handled outside the ordering
  system entirely, so reconciling a day's sales means manually
  cross-referencing multiple tools.

## 2. Solution

DineMate is a **single backend, multi-frontend** platform that models the
whole dine-in journey — from a diner scanning a table QR code, to the
kitchen fulfilling the order, to the owner reconciling the day's bills —
as one connected system with proper roles and tenant isolation.

| Problem | How DineMate solves it |
|---|---|
| Fragmented order flow | One order object flows: Customer places → Waiter/Kitchen sees KOT → status updates (accept → prep → ready → served) sync live back to the customer |
| Menu drift | Menu is owned centrally per restaurant (in the backend) and consumed identically by the customer portal, waiter app, and restaurant portal |
| No floor visibility | Restaurant portal (owner/manager) gets live dashboards: tables, active orders, staff, finance, reports |
| No role separation | Every login goes through one auth endpoint that resolves role (super_admin / owner / manager / waiter) from the username, then scopes what that user can see and do |
| Multi-outlet operators | A **Super Admin** portal manages all restaurants on the platform: onboarding, subscriptions, load balancing, platform-wide reporting |
| Disconnected billing | Orders auto-archive into bills; optional Razorpay integration for online payment, and QR codes are generated per table for frictionless entry |

### Who uses which app

- **Diner** → Customer Portal (web) — scans the table's QR code, browses
  the menu, places an order, and tracks its status live. No app install,
  no login required.
- **Waiter** → Waiter Portal (web) or Waiter Flutter app — sees the live
  floor/table grid, takes orders on behalf of a table, tracks KOTs, and
  generates bills.
- **Restaurant Owner / Manager** → Restaurant Portal (web) or Restaurant
  Flutter app — manages menu, staff, tables & QR codes, finance, offers,
  reports, and notifications for their own restaurant.
- **Platform Super Admin** → Super Admin Portal (web) — onboards and
  manages every restaurant tenant on the platform, subscriptions, and
  platform-wide load balancing/reporting.

---

## 3. Architecture

```
                         ┌────────────────────────┐
                         │   Backend (Next.js)     │
                         │  Auth · Orders · Menu    │
                         │  Billing · Tenants       │  :3000
                         │  Neon Postgres (Drizzle) │
                         └───────────┬─────────────┘
             REST API (Bearer JWT + httpOnly cookies)
        ┌───────────────┬───────────┼───────────────┬───────────────┐
        │                │           │               │               │
  ┌─────▼─────┐   ┌──────▼──────┐ ┌──▼───────┐  ┌────▼─────┐  ┌──────▼──────┐
  │Super Admin│   │ Restaurant  │ │ Customer │  │  Waiter  │  │  Flutter    │
  │  Portal   │   │   Portal    │ │  Portal  │  │  Portal  │  │ Apps (2)    │
  │  :3001    │   │   :3002     │ │  :3003   │  │  :3004   │  │ Rest./Wtr.  │
  └───────────┘   └─────────────┘ └──────────┘  └──────────┘  └─────────────┘
```

All five web apps are independent Next.js projects that talk to the
**same** backend over its REST API — none of them run their own database
or auth logic. The two Flutter apps are alternate, native front ends for
the Restaurant and Waiter roles that hit the same API.

## 4. Repository / Package Map

| Folder you uploaded | App | Role | Port | Stack |
|---|---|---|---|---|
| `DineMate_Backend_Restaurants_OR_Ordering_system-main` | **Backend API** | Core platform | 3000 | Next.js 16, Drizzle ORM, Neon Postgres, JWT auth |
| `DinMate_Comp_supadmin_frontend-main` | **Super Admin Portal** | Platform owner | 3001 | Next.js 15, Tailwind, shadcn/Radix, TanStack Query |
| `DineMate_Restaurant_portal_Supaadmin_Frontend-main` | **Restaurant Portal** | Owner / Manager | 3002 | Next.js 15, Tailwind |
| `DineMate_Restraurant_customer_portal-main` | **Customer Portal** | Diner | 3003 | Next.js 15, Tailwind, Framer Motion, Zustand |
| `DineMate_Waitportal-main` | **Waiter Portal** | Waiter / Manager / Owner | 3004 | Next.js 15 |
| `dinemate_restaurant_flutter` | **Restaurant App (Flutter)** | Owner / Manager / Waiter | — | Flutter, Riverpod, Dio |
| `dinemate_waiter_Flutter` | **Waiter App (Flutter)** | Waiter / Manager / Owner | — | Flutter, Riverpod, go_router, Dio |

> Note: two of the uploaded zips (`DinMate_Comp_supadmin_frontend-main` and
> `DineMate_Restaurant_portal_Supaadmin_Frontend-main`) and the customer/
> waiter portals share the same underlying Next.js template as the backend
> — each is a distinct deployable app, just started fresh from the same
> boilerplate.

---

## 5. Prerequisites

- **Node.js** 20+ and npm
- **Flutter SDK** 3.8+ (only needed if you're building the mobile apps)
- A **Neon Postgres** database (or any Postgres instance — Neon is what
  the backend's `.env` is pre-configured for)
- Optional, for full functionality:
  - **Cloudflare R2** (or S3-compatible bucket) for logo/QR/document storage
  - **SMTP** credentials for email
  - **Razorpay** account for online payments
  - **Twilio** account for SMS

---

## 6. Setup

### 6.1 Backend (start here — every other app depends on it)

```bash
cd DineMate_Backend_Restaurants_OR_Ordering_system-main
npm install
```

Create `.env` (a template already exists in the folder) with at minimum:

```bash
# Postgres connection string (Neon or any Postgres)
DATABASE_URL=postgresql://<user>:<password>@<host>/<db>?sslmode=require

# Random long secret used to sign auth tokens
AUTH_SECRET=<generate a long random string>

# Bootstraps exactly one super_admin account on first start
SUPER_ADMIN_USERNAME=<choose a username>
SUPER_ADMIN_PASSWORD=<choose a strong password>

# Comma-separated origins allowed to call this API (one per frontend port)
ALLOWED_ORIGINS=http://localhost:3001,http://localhost:3002,http://localhost:3003,http://localhost:3004

# Public URL of the customer portal (encoded into table QR codes)
NEXT_PUBLIC_BASE_URL=http://localhost:3003

# Shared secret for signing QR payloads — MUST match the customer portal's
# NEXT_PUBLIC_QR_SECRET exactly
QR_SECRET=<generate a long random string>
NEXT_PUBLIC_QR_SECRET=<same value as QR_SECRET>

# Optional integrations — uncomment/fill in as needed
# R2_ACCOUNT_ID=
# R2_ACCESS_KEY_ID=
# R2_SECRET_ACCESS_KEY=
# R2_BUCKET_NAME=
# R2_PUBLIC_URL=
# SMTP_HOST=
# SMTP_PORT=
# SMTP_USER=
# SMTP_PASS=
# EMAIL_FROM="DineMate <noreply@example.com>"
# RAZORPAY_KEY_ID=
# RAZORPAY_KEY_SECRET=
# RAZORPAY_WEBHOOK_SECRET=
# TWILIO_ACCOUNT_SID=
# TWILIO_AUTH_TOKEN=
# TWILIO_FROM_NUMBER=
```

⚠️ **Security note:** the `.env` files that came with these zips contain
real-looking database credentials, an `AUTH_SECRET`, a `QR_SECRET`, and a
default super-admin password. Treat those as compromised — rotate/replace
every one of them before deploying anywhere beyond your own machine, and
never commit `.env` to a public repo.

Then set up the database and start the server:

```bash
npm run db:push          # push the Drizzle schema to your database
npm run create-superadmin  # bootstraps the SUPER_ADMIN_* account (or it auto-runs on first boot)
npm run dev               # starts the API on http://localhost:3000
```

Useful backend scripts:

```bash
npm run db:generate   # generate a new Drizzle migration from schema changes
npm run db:migrate    # apply pending migrations
npm run db:studio     # open Drizzle Studio (DB browser)
npm test               # run the Vitest suite (89 tests: auth, tenancy, pricing, billing)
```

Once running, interactive API docs are at:
`http://localhost:3000/swagger-ui/index.html`

### 6.2 Super Admin Portal (port 3001)

```bash
cd DinMate_Comp_supadmin_frontend-main
npm install
```

`.env`:
```bash
NEXT_PUBLIC_API_URL=http://localhost:3000
NEXT_PUBLIC_APP_NAME=DineMate Admin
```

```bash
npm run dev   # http://localhost:3001
```

Log in with the `SUPER_ADMIN_USERNAME` / `SUPER_ADMIN_PASSWORD` you set
in the backend `.env`.

### 6.3 Restaurant Portal (port 3002)

```bash
cd DineMate_Restaurant_portal_Supaadmin_Frontend-main
npm install
```

`.env`:
```bash
NEXT_PUBLIC_API_URL=http://localhost:3000
NEXT_PUBLIC_APP_NAME=DineMate Restaurant
```

```bash
npm run dev   # http://localhost:3002
```

Owners log in with their **Restaurant ID** (e.g. `REST000021`) as the
username — create a restaurant first from the Super Admin portal to get one.

### 6.4 Customer Portal (port 3003)

```bash
cd DineMate_Restraurant_customer_portal-main
npm install
```

`.env`:
```bash
NEXT_PUBLIC_API_URL=http://localhost:3000
QR_SECRET=<same value as backend QR_SECRET>
NEXT_PUBLIC_QR_SECRET=<same value as backend QR_SECRET>
```

```bash
npm run dev   # http://localhost:3003
```

No login — diners land directly on `/[restaurantId]/table/[tableNumber]`,
normally by scanning the table's QR code (generated from the Restaurant
Portal).

### 6.5 Waiter Portal (port 3004)

```bash
cd DineMate_Waitportal-main
npm install
```

`.env`:
```bash
NEXT_PUBLIC_API_URL=http://localhost:3000
```

```bash
npm run dev   # http://localhost:3004
```

Waiters log in with the username their restaurant owner assigned them.

### 6.6 Flutter apps (Restaurant & Waiter — optional, for mobile)

```bash
cd dinemate_restaurant_flutter        # or dinemate_waiter_Flutter/dinemate_waiter
flutter pub get
flutter run
```

Point each app's API base URL (in its networking/config layer, via Dio)
at your running backend — `http://localhost:3000` for local development,
or your deployed backend URL otherwise. If you're testing on a physical
device or emulator, replace `localhost` with your machine's LAN IP or a
tunneled URL (e.g. ngrok), since the device can't reach your computer's
`localhost` directly.

---

## 7. Running Everything Together (local dev)

Start each app in its own terminal, in this order:

```bash
# 1. Backend
cd DineMate_Backend_Restaurants_OR_Ordering_system-main && npm run dev     # :3000

# 2. Super Admin
cd DinMate_Comp_supadmin_frontend-main && npm run dev                     # :3001

# 3. Restaurant Portal
cd DineMate_Restaurant_portal_Supaadmin_Frontend-main && npm run dev      # :3002

# 4. Customer Portal
cd DineMate_Restraurant_customer_portal-main && npm run dev               # :3003

# 5. Waiter Portal
cd DineMate_Waitportal-main && npm run dev                                # :3004
```

Typical first-run flow:
1. Log into the **Super Admin Portal** (3001) with your bootstrap
   super-admin account and create a restaurant → note the generated
   Restaurant ID.
2. Log into the **Restaurant Portal** (3002) using that Restaurant ID,
   set up the menu, tables, and staff (managers/waiters).
3. Generate a table QR code from the Restaurant Portal — it encodes a
   link into the **Customer Portal** (3003).
4. Open that link (or scan the QR) to place a test order as a diner.
5. Log into the **Waiter Portal** (3004) with a staff account to see the
   order come in and manage it through to billing.

---

## 8. Authentication Model

- One login endpoint (`POST /api/auth/login`) serves every role; the
  backend infers the role from the username itself (platform username →
  `super_admin`, Restaurant ID → `owner`, assigned username →
  `manager`/`waiter`).
- Returns a short-lived `accessToken` (15 min) and a `refreshToken`;
  send the access token as `Authorization: Bearer <token>` on every
  request, and call `POST /api/auth/refresh` to renew it.
- Same-origin web clients can instead rely on the httpOnly cookies the
  login endpoint sets (`dinemate_session` / `dinemate_refresh`) as a
  convenience — Bearer token takes priority when both are present.

---

## 9. Tech Stack Summary

- **Backend:** Next.js (API routes), Drizzle ORM, Neon Postgres, JWT
  (`jose`), Zod validation, Vitest, Swagger/OpenAPI docs, optional
  Cloudflare R2 / SMTP / Razorpay / Twilio integrations
- **Web portals:** Next.js 15/16, Tailwind CSS, shadcn/ui + Radix,
  TanStack Query/Table, Framer Motion, Zustand
- **Mobile apps:** Flutter, Riverpod, Dio, go_router, flutter_secure_storage

---

## 10. Known Gaps / Things to Configure Before Production

- Rotate every secret shipped in the sample `.env` files
  (`DATABASE_URL`, `AUTH_SECRET`, `QR_SECRET`, super-admin password).
- Storage falls back to local `./public` if R2 isn't configured — fine
  for dev, not for a real deployment.
- Payments (Razorpay), SMS (Twilio), and email (SMTP) are all optional
  and disabled until their env vars are filled in.
- `ALLOWED_ORIGINS` and `NEXT_PUBLIC_BASE_URL` need to be updated to your
  real domains before deploying.
