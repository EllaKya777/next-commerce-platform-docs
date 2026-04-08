# next-e-commerce

A production-ready, full-stack e-commerce platform built on **Next.js 15 App Router**.  
Designed for online retail with a focus on flexibility — the catalog filters, themes, translations, delivery rules, and site settings are all managed via the admin panel without touching code or redeploying.

-----

## Why Next.js 15 + MongoDB

- **Performance by default** — Server Components, streaming, edge-ready. Pages load fast out of the box without manual optimization work.
- **SEO as a first-class concern** — dynamic metadata, JSON-LD structured data, automatic sitemap, 410 redirects on deletion. Built in from day one, not bolted on.
- **Zero UI dependencies** — no Tailwind, no UI libraries. Pure React + CSS Modules.
- **Predictable architecture** — consistent patterns, no abandoned abstractions.

-----

## Stack

|Layer        |Technology                                                       |
|:------------|:----------------------------------------------------------------|
|Framework    |Next.js 15 (App Router, Server Components)                       |
|Language     |TypeScript (strict mode)                                         |
|UI           |React 19 — native CSS Modules                                    |
|State        |TanStack React Query v5                                          |
|Database     |MongoDB + Mongoose                                               |
|Auth         |NextAuth.js v5 — Google OAuth + Magic Link (passwordless)        |
|Order Engine |WooCommerce REST API (WordPress) — built-in headless order engine|
|Payments     |Stripe                                                           |
|Images       |Cloudinary (custom Next.js loader)                               |
|Email        |Nodemailer (SMTP)                                                |
|PDF          |pdf-lib + Fontkit                                                |
|i18n         |next-i18n-router + react-intl                                    |
|Testing      |Playwright                                                       |
|Monitoring   |Sentry                                                           |
|Rate Limiting|Cloudflare WAF (recommended) or alternatives (e.g. Upstash Redis)|

-----

## Quick Start

### Requirements

- Node.js ≥ 18.0.0
- MongoDB instance (local or Atlas)

### Install & Run

```bash
npm install
npm run dev
```

Open <http://localhost:3000>.

### Build

```bash
npm run build
npm run start
```

### Other scripts

```bash
npm run lint              # ESLint
npm run build:clean       # clean build
npm run build:debug       # build with debug output
npm run build:trace       # build with trace (for performance analysis)
```

-----

## Environment Variables

Create a `.env.local` file in the project root:

```env
# ── Required ──────────────────────────────────────────────────────────

MONGODB_URI=mongodb+srv://...           # MongoDB connection string (DB name: Workshop)

NEXTAUTH_SECRET=your-secret             # JWT signing secret

GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

EMAIL_SERVER_HOST=smtp.example.com
EMAIL_SERVER_PORT=587
EMAIL_SERVER_USER=user@example.com
EMAIL_SERVER_PASSWORD=...
EMAIL_FROM=noreply@example.com

NEXT_PUBLIC_SITE_URL=https://yoursite.com
NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME=your-cloud-name

STRIPE_SECRET_KEY=sk_...
NEXT_PUBLIC_STRIPE_PUBLIC_KEY=pk_...

# ── WooCommerce (WordPress) — headless order engine ───────────────────

WOO_BASE_URL=https://your-wp-store.com
WOO_CONSUMER_KEY=ck_...
WOO_CONSUMER_SECRET=cs_...

FEED_SECRET_TOKEN=...                   # Protects the Google Shopping XML feed
GOOGLE_RECAPTCHA_SECRET=...             # Contact form spam protection
SENTRY_DSN=...                          # Error monitoring
```

-----

## End-to-End Tests (Playwright)

The project includes a set of Playwright E2E tests covering the main public user flows and several optional database-level checks.  
The UI tests are **safe** and do **not** require authentication.  
Database tests **write to the database** and must be executed only against a **local or dedicated test database**.

### Setup

```bash
npm install
npx playwright install    # download browser engines (run once)
```

### Running tests

```bash
npm run dev               # start the dev server first

npm run test:e2e          # run all tests (headless)
npm run test:e2e:headed   # run with visible browser windows
npm run test:e2e:report   # open HTML report for the last run

# Run a specific file:
npx playwright test tests/<file>.spec.ts
```

### Test files

| File | Coverage |
|---|---|
| `tests/home.spec.ts` | Smoke — homepage loads and renders |
| `tests/product.spec.ts` | Product page loads and displays expected elements |
| `tests/cart.spec.ts` | Add / remove, quantity controls, persistence across reload |
| `tests/checkout.cart.spec.ts` | Cart → shipping → billing flow |
| `tests/checkout.buy_now.spec.ts` | Buy Now → shipping → billing flow |
| `tests/db/db.spec.ts` | Product CRUD — create, update, delete via Mongoose ⚠️ |
| `tests/db/delivery.spec.ts` | Delivery profile reflected on shipping page ⚠️ |

> ⚠️ Database tests write to the database. Run them only against a local or dedicated test instance — never against production.

-----

## Cart & Session Management

### Registered user

- After login, `useSyncCartOnAuth` runs automatically
- Cart is looked up by `ownerEmail` — the email used for login
- If the device `sessionId` differs from the stored one:
  - Guest cart is deleted
  - Stored `sessionId` is applied to the device cookie
  - Cache and React Query state are updated
- If no saved cart exists for the user, the current guest cart is attached to their `ownerEmail` during checkout

### Guest user

- On entering an email, `usePersonalDataProtection` runs
- If `email ≠ cart.email` — prompt: log out and continue with new email, or restore cart from DB
- If email matches but `sessionId` differs — prompt: continue with the old order or start fresh

### General rules

- One active cart per device at any time
- `sessionId` is stored in cookies (TTL: 5 days)
- Cart TTL — **5 days** (MongoDB TTL index on `lastApplied`)
- All cart operations are centralized through `PreOrderProvider` (React Query)
- Guest cart lookup always uses `{ ownerEmail: { $exists: false } }` — guest data can never bleed into authenticated accounts

### Order email ownership

Order visibility in the customer profile is determined by the **Billing email** — not the login email and not the Shipping email. If a user completes checkout with a different billing email than their login, the order will appear only under the billing email account.

-----

## Order Persistence

The platform uses a **dual-store architecture**:

- **MongoDB** — primary application datastore
- **WooCommerce (WordPress)** — headless operational order engine

After a successful order placement, the PreOrder document is **not deleted** — it is moved into a dedicated `orders` collection in MongoDB and retained for **6 months**.

### MongoDB

|Store      |Role / Retention                                           |
|:----------|:----------------------------------------------------------|
|`preorders`|Active carts only — TTL 5 days; removed on order completion|
|`orders`   |Completed orders — full payload, retained 6 months         |

MongoDB is a fully self-sufficient order store.

-----

### WooCommerce (WordPress) Integration

WooCommerce is a **built-in part of the architecture**.

- **Checkout → MongoDB** — full order payload saved to `orders`
- **MongoDB → WooCommerce** — order created in WooCommerce via REST API
- **Profile ← WooCommerce** — order history and statuses fetched from WooCommerce

-----

## Admin Panel

Access: `/admin` — requires `role: admin`.  
Admin role is assigned **manually via direct MongoDB edit only** — there is no UI for role promotion.

|Section     |URL                       |Purpose                                                       |
|:-----------|:-------------------------|:-------------------------------------------------------------|
|Dashboard   |`/admin`                  |Welcome screen + introduction                                 |
|Guide       |`/admin/guide`            |Built-in documentation                                        |
|Collections |`/admin/collection/[name]`|Products, Categories, Posts — CRUD + WooCommerce sync         |
|Translations|`/admin/translations`     |Edit language packs without rebuild                           |
|Themes      |`/admin/styles`           |Visual theme editor — active theme applied site-wide instantly|
|Pages       |`/admin/pages`            |Static page content (Markdown)                                |
|Settings    |`/admin/settings`         |Languages, currencies, brand, filters (Enums), integrations   |
|Delivery    |`/admin/deliveryProfile`  |Per-country delivery rules                                    |
|Orders      |External link             |WooCommerce order management                                  |

-----

## Product Import

Products, categories, and posts can be imported via **XLSX or JSON**.

The import pipeline uses a **differential algorithm**:

- create
- update
- delete

Imports are safe to retry — WooCommerce IDs are stored per variant.

### MongoDB

MongoDB is the primary catalog store. Each product contains an array of variants.

### WooCommerce sync

After updating MongoDB:

- new items are created in WooCommerce
- updated items are synchronized
- deleted items are removed

WooCommerce receives only essential fields.

### Redirects

Redirects for products, categories, and posts:

- stored in MongoDB
- processed via middleware
- automatically removed when an item is restored

-----

## Key Architecture Decisions

- **Performance & SEO by default** — Next.js 15 Server Components minimize client-side JS. Dynamic metadata, JSON-LD, sitemap, and 410 redirects are built in from day one.
- **Zero-dependency UI** — no Tailwind, no MUI/AntD. Native CSS Modules + CSS Variables. Full control over bundle and rendering.
- **Consistent codebase** — no third-party custom packages, no mixed paradigms. Every part of the application is written in the same style and structure. Easy to extend, easy to hand off.
- **Admin-first configurability** — filters, themes, translations, delivery rules managed from the panel. No redeploy needed.
- **Defense in depth security** — Zod validation + DOMPurify + CSP nonce + Rate Limiting + Cloudflare WAF (recommended).
- **Passwordless auth only** — Google OAuth and Magic Link. No password storage.
- **MongoDB is AI-ready** — Vector Search available natively. Ready to add semantic product discovery and an AI shopping assistant without structural changes.
- **Multi-tenant ready** — the architecture already supports isolated storefronts with separate admin panels and data. Extending to a SaaS model requires no structural changes.

-----

## Documentation

Full technical documentation: `Technical_Documentation.docx`

-----

## Roadmap

- MongoDB Vector Search — semantic product discovery
- AI chatbot — powered by vector embeddings of the catalog
- Full order migration to MongoDB — remove WooCommerce dependency entirely
- Multi-tenancy — manage multiple storefronts from one admin panel (groundwork already in place)
- Real-time cart counter in header
