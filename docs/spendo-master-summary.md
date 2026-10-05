# Spendo — Master Summary

**Single consolidated reference** compressed from: `prd.md`, `Spendo_High_Overview_Plan.txt`, `Spendo_conversation_notes.txt`, `plan/learning-plan.md`, and `plan/week-01` → `week-14`.

- **Product:** Spendo — _"Know where you spend."_
- **Owner / developer:** Yagnesh Pancholi — solo, ~2 hrs/day
- **Created / context date:** 2026-10-05
- **Scope of this doc:** product definition, requirements, architecture, data model, security, cost, 14-week execution plan, release gates, learning plan.
- **Standing data rule:** **synthetic data only.** Never commit or use real personal/patient data (PHI/PII), credentials, tokens, DB dumps, or auth state.

---

## 1. Product Definition

| Item                        | Detail                                                                                                                             |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Feature name**            | Expense Capture & Insights Dashboard (Spendo v1 — Web)                                                                             |
| **Parent epic**             | Spendo — Personal Expense Tracking Platform                                                                                        |
| **Position**                | Foundational feature. Android client, budgets/alerts, recurring expenses and bank sync build on the API + data model created here. |
| **Epic PRD / Architecture** | `/docs/ways-of-work/plan/spendo/epic-prd.md` and `epic-architecture.md` — _to be authored_                                         |

### Problem

People choose between spreadsheets (manual, no insight) and heavyweight finance apps (bank credentials, ads, feature bloat). Most abandon tracking within two weeks because logging takes too many taps and the data is never surfaced in a way that changes behaviour.

### Solution

Single-screen expense entry (≤15s), plus a dashboard answering _"how much, on what, more than last period?"_ with zero configuration. Custom categories, optional receipt photos, filters (date / category / payment method / amount), drill-down from headline number to individual transaction, and export so data is never locked in.

### UX principles

- Record an expense in under 10–15 seconds.
- Understand spending in under 10 seconds.
- Simple → Clean → Extensible → Tested. Avoid over-engineering.

### Success metrics

| Metric                                           | Target       |
| ------------------------------------------------ | ------------ |
| Time to log one expense (median, returning user) | ≤ 15 seconds |
| D14 retention (≥1 expense logged on day 14)      | ≥ 35%        |
| Expenses logged per active user per week         | ≥ 7          |
| Users applying ≥1 dashboard filter in week 1     | ≥ 50%        |
| Signup → first expense conversion                | ≥ 70%        |
| Dashboard p95 load (1 year of data)              | ≤ 1.5s       |

---

## 2. Personas

| Persona                                             | Profile                                                                         | Needs                                                                                                                        |
| --------------------------------------------------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Priya** (primary) — Budget-Conscious Professional | 28, salaried, urban India. UPI + card + cash. Desktop at work, phone elsewhere. | Knows _why_ the month ran short. Zero setup tolerance. **Will not** link a bank account.                                     |
| **Arun** (secondary) — Freelancer                   | 34, variable income.                                                            | Separate personal vs business-ish spend, photograph receipts, export for accountant at year end. Category accuracy > charts. |
| **Sana** (tertiary) — Student                       | 21, small frequent cash transactions.                                           | Simple monthly total + category breakdown. Free tier only.                                                                   |
| **Non-personas (v1)**                               | Households sharing a budget, small-business accounting, investment tracking.    |

---

## 3. User Stories (28)

**Account & Access (1–4):** sign up email+password; stay signed in on trusted browser; reset forgotten password via email; permanently delete account and all data.

**Expense Entry (5–11):** record amount/date/category; date defaults to today and last-used payment method remembered; optional note + merchant; attach receipt photo; save-and-add-another; edit/delete an incorrect expense; warn on near-duplicate logged in last few minutes.

**Categories (12–14):** sensible defaults on signup; create/rename/re-colour/archive own categories; prevented from deleting a category with expenses — offered reassign or archive instead.

**Dashboard & Filters (15–25):** current-month total vs last month; category donut chart; spend trend over time; recent transactions; filter by date range (presets + custom); filter by one or more categories; filter by payment method; filter by amount range; click a chart category to drill into transactions; filter state reflected in URL for bookmarking/sharing; clear "no data" state with CTA.

**Transactions & Export (26–28):** paginated sortable list of all matching expenses; CSV export of filtered set; PDF summary of filtered period.

---

## 4. Functional Requirements (FR-1 → FR-55)

### 4.1 Authentication & Account (FR-1 → FR-6)

- **FR-1** Register with email + password. Email verified via tokenised link; **unverified users may log expenses but cannot export.**
- **FR-2** Password min **12 characters**, checked against a common-password denylist. Stored with a memory-hard hash — **Argon2id or bcrypt cost ≥ 12**.
- **FR-3** Short-lived access token (**≤ 15 min**) + rotating refresh token in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie. **Refresh-token reuse invalidates the entire token family.**
- **FR-4** Password reset via single-use, **30-minute** token by email. Reset **invalidates all active sessions**.
- **FR-5** Login rate-limited (e.g. 5 failures / account / 15 min + per-IP limits). Generic failure message — **no account enumeration**.
- **FR-6** Permanent account deletion: expenses, categories, receipt images hard-deleted within **30 days**; access revoked immediately.

### 4.2 Expense Entry (FR-7 → FR-14)

- **FR-7** Fields — **amount** (required), **date** (required, defaults to today in device local timezone), **category** (required), **payment method** (required: Cash, Card, UPI, Bank Transfer, Other), **merchant/description** (optional, ≤120 chars), **note** (optional, ≤500 chars), **receipt image** (optional).
- **FR-8** Amount ≤ 2 decimals, **> 0**, **≤ 10,000,000**. Stored as `numeric(14,2)` — **never floating point**.
- **FR-9** Date not in the future (client local date) and not earlier than **10 years** ago.
- **FR-10** Pre-select most recently used payment method and category.
- **FR-11** "Save & Add Another" clears only amount, merchant, note, receipt — **retains** date, category, payment method.
- **FR-12** Client-generated **idempotency key** per submission; retries must not create duplicates.
- **FR-13** Identical amount + category + date by same user within **5 minutes** → **non-blocking** duplicate confirmation.
- **FR-14** Edit all fields; delete is **soft delete with 10-second in-session undo**; purge after 30 days.

### 4.3 Timezone Handling (FR-15 → FR-18)

- **FR-15** Client resolves local calendar date and sends `expenseDate` **explicitly as `YYYY-MM-DD`**. Server must **never** derive the expense date from a UTC timestamp.
- **FR-16** Every create/update includes the device **IANA timezone** (e.g. `Asia/Kolkata`), persisted as `captured_timezone`.
- **FR-17** Every dashboard/report/export request includes the IANA timezone. All relative period boundaries ("This Month", "Last 30 Days", comparison windows) resolved **server-side in that timezone**.
- **FR-18** Date field is user-editable, so no profile-level timezone setting in v1.

### 4.4 Receipt Attachments (FR-19 → FR-24)

- **FR-19** One image per expense. JPEG, PNG, WebP, HEIC. Max **10 MB** pre-compression.
- **FR-20** File type validated by **magic-byte inspection** — not extension or client MIME type.
- **FR-21** Re-encoded server-side (**EXIF/GPS stripped**), stored with a **server-generated opaque filename**. Original filenames never used on disk or in URLs.
- **FR-22** **Private Azure Blob container.** Served only via **user delegation SAS**, TTL **≤ 15 min**, scoped per request. **Storage account key never used to sign.** No publicly enumerable paths.
- **FR-23** API authenticates to Blob Storage via **managed identity**. No storage credentials in configuration.
- **FR-24** Client-side downscale to **max 2000px long edge** before upload.

### 4.5 Categories (FR-25 → FR-29)

- **FR-25** Seed **11 defaults** on signup: Food & Dining, Groceries, Transport, Utilities, Rent, Shopping, Health, Entertainment, Education, Travel, Other.
- **FR-26** Create with name (≤40 chars, **unique per user, case-insensitive**), colour, optional icon. **Max 50 active** per user.
- **FR-27** Rename / re-colour applies **retroactively** to history.
- **FR-28** Archive → hidden from entry form, still present in reports and filters.
- **FR-29** Deleting a category with expenses is **blocked**; offer reassignment or archival.

### 4.6 Dashboard (FR-30 → FR-36)

- **FR-30** Defaults to **current calendar month** (device timezone), no other filters.
- **FR-31 Total Spend KPI** — sum of filtered set + absolute and % change vs immediately preceding equivalent period + transaction count + average transaction value.
- **FR-32 Category Breakdown** — donut; **top 8 individually**, remainder grouped as "Other" with tooltip listing contents; clicking a segment applies that category filter.
- **FR-33 Spend Trend** — bar/line; granularity auto-selects **daily ≤ 31 days, weekly ≤ 180 days, monthly > 180 days**; user can override.
- **FR-34 Recent Transactions** — latest **10**, showing date, merchant, category, payment method, amount, receipt indicator; rows link to full list.
- **FR-35** All four widgets respond to the **same global filter state**.
- **FR-36** Every chart has an accessible tabular equivalent via a **"view as table"** toggle.

### 4.7 Filtering (FR-37 → FR-44)

- **FR-37 Date range** presets: Today, This Week, This Month, Last Month, Last 3 Months, This Year, Custom.
- **FR-38 Category** multi-select, **including archived**.
- **FR-39 Payment method** multi-select.
- **FR-40 Amount range** min and/or max, either bound optional.
- **FR-41** **AND across dimensions, OR within a multi-select dimension.**
- **FR-42** Active filters as removable **chips**, with "Clear all" and a visible **result count**.
- **FR-43** Filter state **serialised to URL query params**; restored on load, back/forward, refresh.
- **FR-44** All filtering, sorting, aggregation, pagination execute **server-side**. Client never receives the full dataset.

### 4.8 Transaction List (FR-45 → FR-47)

- **FR-45** Paginated **25 / 50 / 100** per page, honouring active filters.
- **FR-46** Sortable by date, amount, category, merchant — asc/desc. **Default: date descending.**
- **FR-47** Inline edit and delete per row.

### 4.9 Export (FR-48 → FR-55)

- **FR-48 CSV** columns: Date, Amount, Currency, Category, Payment Method, Merchant, Note, Has Receipt.
- **FR-49 PDF** contains KPI summary, category breakdown, trend chart, full transaction table.
- **FR-50** PDF composed **server-side in .NET** (e.g. QuestPDF); charts rendered server-side to SVG/PNG and embedded. **No headless browser in v1.**
- **FR-51** All user text (merchant, note, category) escaped/sanitised before embedding in CSV or PDF.
- **FR-52** Export generation is **asynchronous** — enqueue + poll; never hold an HTTP request open.
- **FR-53** Cap **50,000 rows**; over cap → prompt to narrow filters.
- **FR-54** Export rate-limited to **10 requests / user / hour**.
- **FR-55** CSV fields starting with `=`, `+`, `-`, `@`, tab, or CR are **prefixed with a single quote** (formula-injection prevention).

---

## 5. Non-Functional Requirements (NFR-1 → NFR-45)

### Performance (NFR-1 → NFR-7)

- Dashboard **p95 ≤ 1.5s, p99 ≤ 3s** for 10,000 expenses over 3 years.
- Expense create/update API **p95 ≤ 300ms** (excluding image upload).
- Filter re-query **p95 ≤ 500ms**.
- Angular initial bundle **≤ 300 KB gzipped**; dashboard route lazy-loaded; charting library loaded on demand.
- Indexes on `(user_id, expense_date DESC)`, `(user_id, category_id)`, `(user_id, payment_method)`; verified with `EXPLAIN ANALYZE` in CI against a seeded **1M-row** dataset.
- Table partitioning deferred, but schema must not preclude future **range partitioning on `expense_date`**.
- Performance validated on the **actual production DB SKU (burstable)** — never a dev machine. Remedy = indexing and pre-aggregation **before** scaling the SKU.

### Security (NFR-8 → NFR-20)

- Every query scoped by `user_id` **derived from the token**, never from a client parameter. Cross-tenant access **structurally impossible**, enforced at the repository layer, covered by automated tests.
- **PostgreSQL Row Level Security** on `expenses`, `categories`, `receipts`; app connects as a **non-superuser, non-`BYPASSRLS`** role.
- Server-side allowlist validation; client-side validation is UX only and never trusted.
- Parameterised queries / EF Core LINQ. Dynamic sort maps to a **fixed column allowlist** — never interpolate.
- **CSP** `default-src 'self'`, no `unsafe-inline`/`unsafe-eval`, nonce-based scripts. Plus HSTS, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.
- **CORS** restricted to known origins; no wildcard with credentials.
- Secrets from **Azure Key Vault** or env config; **managed identity preferred**. Nothing in source control.
- **TLS 1.2+** in transit; encryption at rest for DB and blob.
- Structured audit logging for auth events, exports, account deletion — **never** passwords, tokens, amounts tied to identity, or full email addresses.
- Dependency vulnerability scanning in CI; **build fails on high/critical**.
- Global rate limiting per user and per IP, stricter on auth and export.
- Error responses generic; **no stack traces** in production.
- **Headless-browser PDF rendering out of scope for v1** (HTML/script injection → SSRF / local file disclosure, plus memory cost). If adopted later: sandboxed, isolated worker, egress restrictions, hard render timeouts.

### Email & Deliverability (NFR-21 → NFR-24)

- Dedicated transactional email service — **not** a consumer mailbox SMTP (ToS breach, volume caps, destroyed deliverability).
- Sending domain requires **SPF + DKIM + DMARC**.
- Transactional mail isolated on its **own subdomain**, separate from future marketing mail.
- **Bounce and complaint webhooks** consumed and logged.

### Privacy & Compliance (NFR-25 → NFR-28)

- Data minimisation — email + expense data only. No fingerprinting, no third-party analytics transmitting expense content.
- Full data export + deletion rights (GDPR-style), deletion within **30 days**.
- EXIF/GPS stripped from receipts.
- Privacy policy and terms at signup with **explicit consent capture**.

### Accessibility (NFR-29 → NFR-32)

- **WCAG 2.2 Level AA**. Full keyboard operability, visible focus, logical tab order.
- Charts not colour-only — patterns, labels, tabular equivalent. **4.5:1** text contrast, **3:1** UI components.
- Form errors programmatically associated with inputs and announced via live regions.
- Respect `prefers-reduced-motion` — chart animations disabled.

### Reliability & Operations (NFR-33 → NFR-36)

- **99.5% monthly availability** target for v1.
- **Daily automated backups** with PITR, **30-day retention**; restore tested **quarterly**.
- Health + readiness endpoints, structured JSON logging with **correlation IDs**, alerting on error-rate and latency SLO breaches.
- **Zero-downtime deployments**; migrations **backward-compatible with the previously deployed app version**.

### Compatibility & Responsiveness (NFR-37 → NFR-39)

- Latest **two major versions** of Chrome, Edge, Firefox, Safari.
- Responsive **360px → 1920px**; mobile web fully usable (Android ships later).
- Angular localisation infrastructure in place; v1 ships **`en-IN` only**. All currency/date formatting via locale-aware pipes — no hardcoded symbols.

### Data Model (NFR-40 → NFR-42)

- Single currency **INR** in v1, but every monetary record stores an explicit **ISO-4217 currency code** (avoids destructive multi-currency migration).
- `created_at` / `updated_at` as UTC `timestamptz`. **`expense_date` is a `date` column** and is the **sole source of truth** for reporting period. Aggregation must **never** derive period from `created_at`.
- Primary keys are **UUIDv7** — not sequential integers — to prevent enumeration.

### Hosting & Cost (NFR-43 → NFR-45)

- All compute on **Linux** App Service / container plans (Windows ≈ **4.6×** more for equivalent SKUs, zero benefit to this stack).
- Receipt image storage is the **primary cost-growth variable**; client-side downscaling is a cost control (~**20×** difference in stored bytes).
- Infrastructure as code (**Bicep**) — reproducible environments, auditable cost.

---

## 6. Acceptance Criteria — Key Checks

### Expense entry (US-5/6/9)

Date pre-filled with today, last category + payment method pre-selected · `450.50` / today / Food & Dining / UPI saves with confirmation **within 1s** · empty amount → inline "Enter an amount greater than 0", focus moves, screen-reader announced · `0` or negative rejected **client and server** · future date → "Date cannot be in the future" · `12.345` rejected or normalised to 2dp with visible notice · "Save & Add Another" clears amount/merchant/note/receipt, keeps date/category/payment, **focus returns to amount** · retry with same idempotency key → **exactly one record** · full keyboard operability with visible focus.

### Timezone

Device `Asia/Kolkata` at 01:00 on 5 Oct → date defaults to **`2026-10-05`**, not the UTC date · request contains explicit `expenseDate` + IANA ID · record holds `captured_timezone` · expense logged in `Asia/Kolkata` viewed from `America/New_York` **does not shift months** · "This Month" from `Asia/Dubai` resolves boundaries in **`Asia/Dubai`**, not UTC or a server default · manual date edit honoured without warning.

### Receipts (US-8)

4 MB JPEG → thumbnail preview, linked to expense · 15 MB file rejected **before upload begins** · `payload.exe` renamed `receipt.jpg` rejected by magic bytes, **nothing persisted** · GPS EXIF absent after retrieval · 4000px photo transmitted ≤ 2000px · expired SAS URL denied · direct container probe without SAS denied · **no storage key or credentialed connection string** in deployed config · another user's receipt ID with my valid token → **404 (not 403)**.

### Duplicate detection (US-11)

₹200 / Transport / today logged 3 min ago → identical submit shows dismissible dialog; "Save anyway" creates the record.

### Categories (US-12–14)

New user sees **11 defaults** · "Pet Care" appears in entry form and filter · "travel" rejected as duplicate of "Travel" (case-insensitive) · renaming "Food & Dining" → "Eating Out" updates all history · deleting a category with 30 expenses blocked, reassign/archive offered · archived category absent from entry dropdown but **still selectable in dashboard filter**.

### Dashboard (US-15–18)

KPI equals exact sum to 2dp · ₹10,000 vs ₹8,000 → "**+₹2,000 (+25%) vs last month**" with directional indicator · zero previous period → "**No data for comparison**", not ∞% or div-by-zero · 12 categories → top 8 individual + 4 grouped "Other", segments **sum to 100%** · 7-day range → daily granularity; 12-month → monthly · "view as table" shows identical values accessibly · 50 expenses → exactly the **10 most recent, newest first** · 10,000 expenses / 3 years on the **production SKU** → **p95 ≤ 1.5s over 100 runs**.

### Filtering (US-19–25)

"Last Month" updates **all four widgets** and result count · Groceries + Transport returns either, excludes others · Cash + ₹500–₹2000 **inclusive** · min > max → validation error, **no query issued** · custom range end before start rejected · each filter a removable chip, "Clear all" resets to default current-month · copied URL restores identical view in a new tab · browser **Back** restores previous filter state · clicking "Shopping" donut segment adds the chip and updates all widgets · zero matches → empty state explaining why + "Clear filters" · another user's ID in the query string is **ignored**, only own data returned.

### Transaction list (US-26/27)

120 matches at 25/page → **5 pages** with total count displayed · sort by amount desc → highest in the **entire filtered set** first, not just the page · arbitrary sort column via API rejected against the **allowlist**, no SQL constructed from input.

### Export (US-28)

300 matches → **300 data rows + header**, Amount column sums to the **Total Spend KPI** · note `=cmd|'/c calc'!A1` renders as **inert text** in Excel · commas/quotes/newlines correctly quoted, parses without corruption · 60,000 matches → prompted to narrow, **not truncated** · PDF contains KPI summary, both charts, transaction table · HTML/script in merchant name appears as **literal escaped text** · large export returns **immediately with a job reference**, polled · deployed container image contains **no headless browser runtime** · 11th export in an hour → rate-limit message with retry time.

### Authentication (US-1–4)

Valid email → verification email sent, told to check inbox · 8-char password rejected with policy explained · wrong password → "**Invalid email or password**", identical to non-existent account · 5 failures then 6th → temporary lockout with retry-after · reset link used twice → second rejected as consumed · completed reset **invalidates all other sessions** · DNS shows valid **SPF, DKIM, DMARC** · verification mail to Gmail/Outlook passes SPF/DKIM alignment and **lands in inbox** · hard bounce recorded and observable via webhook · account deletion confirmed with password → access revoked immediately, all records + receipts purged within 30 days.

### Cross-cutting

axe-core reports **zero WCAG 2.2 AA violations** on entry form, dashboard, transaction list · all pages fully operable at **360px with no horizontal scrolling** · automated suite confirms **no endpoint returns another user's data under any parameter manipulation** · CI blocks merge on any high/critical dependency vulnerability · all IaC-provisioned compute is **Linux**.

---

## 7. Architecture & Tech Stack

```
Angular Web / PWA  ──HTTPS/REST──▶  ASP.NET Core Web API  ──EF Core──▶  PostgreSQL
Future Android app ──────────────▶  (same API)
```

**Frontend:** Angular (21) · Angular Material · responsive PWA first · Capacitor can package the PWA for Android later · charts later (Chart.js / Apache ECharts) · strict TypeScript · SCSS · Karma/Jasmine unit tests · Playwright E2E.

**Backend:** ASP.NET Core (.NET 10) Web API · REST · versioned `/api/v1/...` · EF Core + Npgsql · Dapper may be added later for specific complex/high-performance queries · **DTOs instead of exposing EF entities** · EF Core migrations · Clean-Architecture separation (API / Application / Domain / Infrastructure) · CQRS/MediatR used **selectively, not forced**.

**Cloud:** Azure. Object storage = **Azure Blob** (chosen over S3 to avoid cross-cloud egress, a second IAM model, and long-lived static access keys). PDF = server-side .NET composition. Timezone = device-level, client-resolved.

### Repository & structure

Separate repos: **`spendo-frontend`**, **`spendo-backend`** (optional future `spendo-infrastructure`). The week plans assume a single Spendo repo layout:

```
backend/   Spendo.slnx, Spendo.Api, Spendo.Application, Spendo.Infrastructure, Spendo.Tests
frontend/  spendo/ (Angular app, e2e/)
sql/       README.md + verification/*.sql  (reviewed SQL checks, NOT a second migration system)
docker/    compose.yaml, api.Dockerfile
build/     production build + deploy config
docs/      architecture, api, deployment, operations, testing, releases
.github/   workflows/, pull_request_template.md
```

**EF Core migrations in `backend/Spendo.Infrastructure/Migrations/` are the schema source of truth.**

Conceptual reference layout from notes: `frontend/`, `backend/{API,Application,Domain,Infrastructure}`, `database/migrations/`, `tests/{UnitTests,IntegrationTests}`, `docs/{architecture,api,releases}`, `README.md`.

### Branching & releases

- **main-only** strategy initially; no `develop` branch.
- Feature branches: `feature/add-expense`, `feature/authentication`, `feature/week-NN-...`.
- Flow: feature branch → **Pull Request** → `main` → CI/CD → production.
- `main` protected. PR template: purpose, author, verification, rollback.
- **Semantic versioning.** Tag **merged `main` only, never a feature branch**:
  ```
  git switch main; git pull --ff-only
  git tag -a v0.1.0 -m "First MVP release"
  ```

---

## 8. Database Design

**Core tables:** `Users`, `Categories`, `Expenses`. **Future:** `ExpenseAttachments`, `RecurringExpenses`, `PaymentMethods`, `Budgets`, `SharedExpenses`.

**Expense fields:** `Id (UUID)`, `UserId`, `CategoryId`, `Amount`, `Description`, `PaymentMethod`, `ExpenseAt`, `Notes`, `CreatedAt`, `CreatedBy`, `UpdatedAt`, `UpdatedBy`, plus `Currency` (ISO-4217, default `INR`) and `captured_timezone`.

**Audit:** business entities carry `Id, CreatedAt, CreatedBy, UpdatedAt, UpdatedBy`. `CreatedBy/UpdatedBy` use stable **`Users.Id` UUIDs, not email**. `Users` itself needs no `CreatedBy` initially. EF `SaveChanges` populates audit fields automatically.

**Time & IDs:** UTC is the standard. Exact moments stored in UTC `timestamptz`; **date-only concepts stay date-only** to avoid timezone shifting. UUIDs preferred, partly to enable future offline/mobile sync.

**Design principles:**

- No arbitrary `Field1`/`Field2` columns. No JSON for everything.
- New known requirements → normal columns via EF migrations.
- Truly dynamic user-defined fields → separate extension model or PostgreSQL JSONB, **only if genuinely needed**.
- **Derive** year/month/week/day from the expense date; do **not** store redundant Year/Month/Week columns.
- Index from **actual query patterns**: `UserId + date`, `UserId + CategoryId + date`, `UserId + PaymentMethod + date` — not blanket indexing.
- Additional week-plan indexes: `(OwnerId, ExpenseAtUtc, Id)` for stable pagination; amount positive check constraint; per-user unique normalized category name; owner FK + index.

### Composable filter API

```
GET /api/v1/expenses?fromDate=2026-09-01&toDate=2026-09-30&categoryId=2&paymentMethod=UPI&minAmount=100&maxAmount=1000
```

Use **composable query/filter objects**, not a separate method per combination. A single typed filter contract drives SQL, API and Angular URL state.

### Endpoint inventory (from week plans)

`POST/GET/PUT/DELETE /api/v1/expenses` · `GET /api/v1/expenses/export` · `GET /api/v1/categories`, `/categories/{id}` · `GET /api/v1/dashboard/summary` · `GET /api/v1/dashboard/analytics` · `POST /api/v1/auth/{register,login,refresh,logout,forgot-password,reset-password}`.

**Status conventions:** 201 + Location on create · 204 on delete · 400 invalid · 401 unauthenticated · **404 for another owner's resource** (never 403, to avoid confirming existence) · 409 conflict · 429 rate limit · generic 500.

### Authentication model

- Auth included from V1 because future Android/cloud sync requires user ownership.
- V1: email/password. Future: Google Sign-In, potentially Apple — multiple providers map to the **same internal User entity**.
- **API derives current user from auth context; never trust a client-supplied `UserId`.**
- Access token in **memory only**; refresh token in **HttpOnly cookie, never exposed to JavaScript**.
- Refresh tokens: cryptographic RNG, **store only hashes**, rotate atomically, **revoke token family on reuse**, token-family identifier + expiry + used/revoked timestamps.
- Angular guards are **UX only** — server-side owner checks remain authoritative.

---

## 9. Roadmap & Releases

| Release      | Phase                      | Content                                                                                                                                                                                                                                                                                                                                                                    | Est.    |
| ------------ | -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| —            | **Prototype**              | Angular + API + PostgreSQL, `Expenses` table, add/get/delete, basic list, basic dashboard total, Git repo                                                                                                                                                                                                                                                                  | 1–2 wks |
| **`v0.1.0`** | **MVP**                    | Signup/login, JWT, user↔expense relationship, categories, payment methods, add/edit/delete, date/time handling, validation, pagination, month/year + category + payment + amount filtering, search, combined filters, monthly/yearly/category/payment dashboard, responsive UI                                                                                             | 4–6 wks |
| **`v0.2.0`** | **Dashboard & experience** | Week/month/year + custom date range, all filters combined, **charts**, category analytics, payment analytics, better mobile UX                                                                                                                                                                                                                                             | 2–3 wks |
| **`v0.3.0`** | **Production engineering** | Global exception handling, proper status codes, DTOs, validation, authN/authZ, logging, config management, migrations, security checks, API versioning, unit + integration tests; frontend error/loading/empty states, form validation, auth guards, responsive, **PWA**, accessibility, **CSV export**; GitHub repo, PRs, CI, build, automated tests, deployment pipeline | 2–3 wks |
| **`v1.0.0`** | **Production v1**          | Production deployment, security hardening, monitoring, logging, DB backups, documentation, API docs, final UI polish, performance checks, production config, custom domain, stable release process                                                                                                                                                                         | 3–5 wks |

**Total: ~3–4 months ≈ 120–220 focused hours** at ~2 hrs/day.

**Future (post-v1):** Google Sign-In · Android app · recurring expenses · receipt attachments/scanning · budgets · multiple payment accounts · shared expenses · offline-first sync · natural-language entry · AI insights.

**Offline support:** not required for V1, but architecture must not prevent it. UUIDs, clear API boundaries and user ownership make future local storage + sync feasible. Conflict handling designed when actually implemented.

### Core development rule

> **Build V1 simply, but never build V1 in a way that prevents V2.**

```
Prototype → v0.1 MVP → v0.2 Analytics → v0.3 Product Quality → v1.0 Production
```

Prefer **Simple → Clean → Extensible → Tested** over **Complex → "Enterprise" → Unfinished.**

### First target (the "alive" milestone)

```
Angular → POST /api/v1/expenses → ASP.NET Core API → EF Core → PostgreSQL
PostgreSQL → GET /api/v1/expenses → API → Angular → Expense List
```

Concretely: _open Spendo → enter ₹250 → select UPI → Food → Save → see ₹250 in the list._

---

## 10. 14-Week Execution Plan (2 hrs/day × 7 days per week)

Every week follows the same shape: **Day 1 SQL → Days 2–3 API → Days 4–5 Angular → Day 6 tests → Day 7 verify/merge.** Every week ends in a reviewed PR merged to `main`.

### Week 1 — Foundations

Create the Spendo repo with `backend`, `frontend`, `sql`, `docker`, `build`, `docs`; `.gitignore` (`.env`, Playwright auth state, `bin/obj/node_modules`), `README.md`, PR template; protect `main`. Day 2: `docker/compose.yaml` for local **PostgreSQL 17** with named volume, health check, env from untracked `.env`; `sql/README.md` documenting EF migration ownership. Day 3: scaffold solution + Api/Application/Infrastructure/xUnit projects, project references, health checks. Day 4: EF Core/Npgsql (check versions + advisories), `SpendoDbContext`, baseline migration. Day 5: Angular 21 standalone app with routing, SCSS, strict TS under `frontend/spendo`, **Karma/Jasmine chosen at generation time**, health/status page, typed API base URL. Day 6: one xUnit test, one Jasmine shell test, Playwright smoke nav test. Day 7: lint/tests/production builds, verify Docker restart preserves the volume, PR merged. **No tag.**
**DoD:** fresh developer can start DB/API/UI from documented commands; tests and builds pass; `.env`/credentials/auth state untracked; baseline schema rebuildable from migrations.

### Week 2 — Local expense prototype

Add/list/delete an INR expense end to end. **Local-development-only; must not be deployed publicly or released before Week 3 ownership enforcement.**
`Expenses` with UUID ID, `numeric(12,2)` amount + positive check, 3-char `Currency` default `INR`, description length, payment method, category label, UTC `timestamptz`, optional notes. DTOs in `Spendo.Api/Contracts/Expenses/`, service in `Spendo.Application/Expenses/`, endpoints in `ExpenseEndpoints.cs` under `/api/v1/expenses`, deterministic UTC/ID order. 201+Location / 400 / 404 / 204. xUnit with **Moq at the repository boundary** (no live data, no EF InMemory as SQL proof). Angular: strict models + `ExpenseApiService` in `core/api/`, environment base URL, **never direct HTTP from a component**, `HttpTestingController` tests. Reactive add form, accessible list, delete confirmation, loading/error/empty states. Playwright: add synthetic ₹250 Food/UPI → see row → cancel delete → delete → empty state.
**DoD:** invalid amount rejected by **both** API and DB; `Currency` persists as INR; no client-provided identity accepted; no publicly reachable anonymous expense API.

### Week 3 — Authentication part 1 (DB + API)

Replace the anonymous prototype with user-owned data. ASP.NET Core Identity tables, owner FK/index on `Expenses`, refresh-token table (token hash, user ID, expiry, used/revoked timestamps, token-family ID). Purge disposable prototype rows **only in a disposable dev DB**; any non-disposable environment requires a reviewed data migration. Identity registration, unique normalized email, password policy, **account lockout**. JWT issuance/validation in `Spendo.Api/Auth/` — pin issuer, audience, signing algorithm, lifetime; cryptographic RNG refresh tokens, hashes only, atomic rotation, family revocation on reuse, **generic failure messages**. Endpoints: register/login/refresh/logout/forgot-password/reset-password; reset links via a **pluggable mail sender (local sink in dev, not a real mailbox)**; single-use, time-limited tokens. Refresh cookie `HttpOnly`, `Secure`, scoped path, explicit SameSite; **CSRF protection on cookie-authenticated refresh/logout** via same-site origin validation or antiforgery token. Enforce owner predicates on **every** expense query/mutation; never accept user ID in DTOs; 401 unauthenticated, **404 cross-owner**.
**DoD:** all expense operations authenticated with server-derived ownership; rotation, logout revocation, reset single-use, cookie flags and CSRF all tested; no credentials logged.

### Week 4 — Authentication part 2 (UI + default categories)

`Categories` with owner FK, per-user normalized unique name, soft-delete policy decision; `CategoryId` FK on `Expenses`; ownership index. **Transactional signup seeding — exactly one default category set, idempotent on retry.** `GET /api/v1/categories` returns only caller-owned rows; expense create validates category owner **server-side**. Angular: strict login/register forms with field + server validation, accessible errors, generic reset response; `AuthApiService`; **memory-only session state**. Interceptors: auth header + coordinated refresh-on-401, **API origin allowlist** (never send bearer to arbitrary hosts), no refresh recursion, clear memory and redirect on failed refresh. Guards on `expenses` and `dashboard`; session restored with a **single** refresh request on reload; logout. Replace static category options with owner-scoped API data. Playwright: register → login → create expense → logout → blocked navigation. Manually verify Chrome cookie settings, CSRF rejection, **no bearer in storage**, no stale user data after logout.

### Week 5 — Expense management

Review `Expenses` `timestamptz`, nullable notes, category FK, `(OwnerId, ExpenseAtUtc, Id)` index, amount constraint. `PUT /api/v1/expenses/{id}` with DTO validation, owner-filtered lookup, owner-validated category ID, **server-generated** created/updated UTC timestamps, 404 for another user. Bounded `page`/`pageSize`, deterministic sort **with tie-breaker**, caller-scoped total count. Reuse the add form for edit; validate required description, 2-decimal amount, category selection; preserve server validation messages **without trusting HTML**. Centralise local ↔ UTC conversion in `shared/date-time/`; define **date-only values separately from instants**; test the daylight-saving boundary. Accessible pager + query-parameter navigation. Playwright: edit ₹250 → ₹300, paginate, verify after reload.
**DoD:** amount never calculated with binary floating-point totals; UTC round-trip and boundary tests pass.

### Week 6 — Basic filters & dashboard → **`v0.1.0`**

`EXPLAIN` owner/date/category/payment indexes; use **half-open UTC ranges** derived from the selected calendar zone — never string formatting on indexed columns. Bounded query DTO combining month/year + category + payment with the owner predicate; reject invalid values; keep stable pagination. `GET /api/v1/dashboard/summary` grouped by category and payment with month/year totals and explicit zero/empty behaviour; **aggregate `numeric` in PostgreSQL**, return `Currency: INR`. Typed filter state in **URL query parameters**, preserved on reload, pager reset on filter change. Responsive accessible totals/cards with loading/empty/error states and a stated **reporting time zone**. Day 7: CI-equivalent tests/build, `docs/releases/v0.1.0.md`, merge PR, **then tag merged `main`** and publish notes.
**DoD:** totals match filtered owner rows in INR including empty results; `v0.1.0` points to checked, merged `main`.

### Week 7 — Composable filters & search

Parameterised owner-scoped predicates and date/amount indexes; `EXPLAIN (ANALYZE, BUFFERS)` on synthetic rows; **decide whether simple `ILIKE` suffices before adding full-text indexing**. Extend `ExpenseFilterRequest`: UTC half-open start/end, min/max decimal, normalized search term, search-length and page-size bounds; **400 on contradictory ranges / min > max / invalid dates**. Compose filters via parameterised EF LINQ; **escape literal `%` and `_`**; never concatenate SQL; apply the owner predicate **before** aggregates and paging. Pure `filterToParams` / `paramsToFilter` for a shareable URL, clearing invalid/stale keys. Accessible period/date/amount/search controls — **debounce search, cancel superseded HTTP requests**, reset page, active-filter chips; no request per keystroke. Ensure dashboard filters use **identical semantics**. **No tag this week.**

### Week 8 — Charts & analytics → **`v0.2.0`**

Profile grouped owner/time/category/payment queries; determine calendar bucket boundaries and zero-period behaviour; add an index migration **only if profiling justifies it**. `GET /api/v1/dashboard/analytics` in `DashboardEndpoints.cs` — bounded shared filter contract, typed labels, decimal totals, `INR`, **capped bucket count and date range**. Verify category + payment totals equal the filtered summary, owner isolation, and UTC/reporting-timezone bucket edges. Angular: **verify Angular 21 compatibility, licence and advisory status before selecting a chart library**; trend + category charts; **an accessible data table alongside every chart**; reuse URL filters; touch-friendly responsive layout; clear empty/zero states; aria headings; **no raw transaction detail in chart tooltips**. Pin and review the resolved lockfile version before merging. Day 7: tests/build/a11y review, `docs/releases/v0.2.0.md`, tag `v0.2.0` on merged `main`.
**Rule:** charts are **not a second accounting engine** — totals stay server-calculated.

### Week 9 — Error handling, logging, DTO hardening, API versioning

Review constraints, cascade behaviour, unique category names, concurrency strategy; EF migration only for missing invariants. Centralise exception handling in `Spendo.Api/Errors/` with **RFC 9457 problem details**, stable codes, correlation IDs, generic 500, and correct 400/401/403/404/409/429 — **no stack traces, no account-existence disclosure**. Structured log scopes + request IDs; **redact authorization headers, cookies, reset links, free-text expense descriptions and user identifiers**; document retention and access in `docs/operations/logging.md`. Keep request/response types separate from EF entities, reject unknown identity/owner fields, bound lengths and page sizes; configure explicit v1 versioning — **verify a .NET 10-compatible versioning library exists, or keep explicit versioned routes without another dependency**. Angular problem-details mapper + global error feedback; **retry only safe idempotent reads**; show actionable validation and a generic **incident ID** for unexpected errors.
**DoD:** v1 behaviour backward compatible; no credentials or expense free text in logs.

### Week 10 — PWA, accessibility, CSV → **`v0.3.0`**

Export query uses owner + shared filters with deterministic order; **no separate unbounded search path**. `GET /api/v1/expenses/export`: owner scoping, row/time bounds, UTF-8 CSV, fixed columns (date, amount, currency, category, payment, description), **neutralise formula prefixes `=` `+` `-` `@` tab CR/LF**, escape quotes/commas/newlines, **never export notes by accident**, prefer numeric amount formatting over user-controlled strings; file name without private identifiers. `ng add @angular/pwa` after checking the Angular 21 version; inspect `ngsw-config.json` and **restrict caching to public app-shell assets — never cache authenticated API responses, tokens or expense data; never store auth state. Offline expense editing is out of scope.** Accessibility audit: keyboard/focus order, labels, contrast, **modal focus return**, table alternatives, narrow viewport. Filtered export action with download error state. Day 7: manual offline-shell vs API check, `docs/releases/v0.3.0.md`, tag merged `main`.

### Week 11 — Test hardening

**No new product behaviour, no release tag.** `sql/verification/regression.sql` for owner FK, positive amount, unique category checks on a **disposable** local DB; reruns must not destroy shared data. Expand xUnit for amount bounds, date filters, cross-owner mutations, **refresh-token reuse**, category seeding, CSV injection — Moq + synthetic fixtures. Boundary tests for 400/401/404/problem-details and authorization through a test host; isolated DB only for EF transaction/SQL-specific assertions, **never live data**. Karma coverage for form, guard, interceptor, UTC/date-only helpers, filter round-trip, error/empty states — `HttpTestingController` and spies, **not a live API**. Playwright suite covering register/login, add/edit/delete, filter/dashboard, CSV, logout — **isolated synthetic account and data per test, role/label locators, wait for responses not sleeps, no committed auth state**. Flake review: **fix isolation instead of adding retries/sleeps**; CI artifacts without sensitive response bodies. Document commands, coverage baseline, known gaps in `docs/testing.md`.
**Principle:** mocks exercise application decisions, **not PostgreSQL semantics** — keep migration/constraint verification separate. Coverage baselines must not be inflated by deleting unrelated tests.

### Week 12 — CI/CD and deployment research/setup

Research first, then a **safe, repeatable GitHub Actions pipeline and only a non-production deployment if hosting/budget is approved.** Compare Azure Container Apps vs App Service for the API, Static Web Apps for Angular, Azure Database for PostgreSQL; document pricing, free quotas, TLS/domain, region, backup, email, **cookie same-site topology** and ongoing cost in `docs/deployment/options.md`. **Verify official pricing and current SKU limits — free tiers are not guaranteed.** Fallback (Cloudflare Pages + Render + Supabase) only after evaluating security, cross-site cookies, backups and cost. Document EF migration promotion/rollback and **backup-before-migrate**; **never run PR-branch migrations against production**. Multi-stage `docker/api.Dockerfile` with health checks and **non-root runtime**; signing key/DB connection via a managed secret provider, **not image or repo**. Angular production build in `build/` with injected public API origin; **no private secrets in compiled JavaScript**; document SPA fallback. `.github/workflows/spendo-ci.yml` — **pinned trusted actions, least-privilege permissions**, dependency caching, `dotnet test`, Karma ChromeHeadless, Playwright against **isolated ephemeral services**, builds, artifact retention; **no production secrets on untrusted PRs**. CD gated by environment approvals and **OIDC federated credentials rather than long-lived cloud keys**, staging health probe and documented rollback; validate cookie/CORS/CSRF across actual UI/API origins. **Do not auto-deploy to production.**

### Week 13 — Security hardening, monitoring, backups

**Controls must be tested, not merely configured.** Least-privilege DB roles, TLS, schema grants, indexes, backup retention; **synthetic disposable** DB for restore rehearsal; backups encrypted at rest with restricted access. Verify JWT issuer/audience/expiry, refresh reuse/revocation, password reset, rate limits, **per-owner predicates on all list, dashboard and export endpoints**; CSRF defences for cookie-authenticated refresh/logout; precise CORS origins. HTTPS-only, secure headers and CSP for the **actual host**; avoid `unsafe-inline` without a documented exception; review dependency advisories and lockfile; **no raw user HTML in the UI**. Privacy-safe health/readiness probes, structured metrics, alert thresholds — **never tokens, account email or expense text in telemetry**; distinguish auth-failure metrics from user identity. Document scheduled encrypted `pg_dump`/provider backups, retention, **RPO/RTO**; perform an isolated local restore and verify **row counts/checksums without overwriting the live DB**. Merge only when critical issues have owners and resolution; record the **threat model and unresolved risks**.
**Note:** restore only into a disposable `spendo_restore` DB; on Windows PowerShell avoid text pipelines that corrupt binary dumps; **never track a dump in Git**.

### Week 14 — Documentation, polish, domain → **`v1.0.0`**

Subject to Weeks 12–13 approvals. Verify production migration plan and backup/restore evidence against PostgreSQL 17; `sql/verification/release.sql` with **non-destructive** checks for schema version and index presence; require backup + change approval before a live migration. Verify documented endpoints, JWT/reset lifecycle, health checks, error codes, export contract; update `docs/api.md` and `docs/architecture.md` with owner boundaries and currency semantics. Audit small-screen layouts, keyboard navigation, chart alternatives, loading/empty/error states, timezone labels; update `frontend/spendo/README.md` with Angular 21/Karma/Playwright commands. **Configure a custom domain only after DNS/TLS ownership is resolved** — DNS, TLS, HTTPS redirects, UI/API origin policy, cookie/CSRF settings, external smoke test, documented rollback DNS TTL **without secrets**; if not approved, document the blocked release condition. Finalise root `README.md`, `docs/deployment/`, `docs/operations/`, `docs/testing.md`, `docs/releases/v1.0.0.md` with purpose, owner, rationale, status, setup, restore, rollback, known limitations. Release candidate: all tests/builds + manual synthetic acceptance (register, reset password, create/edit/filter/delete, chart, CSV) + **rehearse rollback on staging**. Day 7: signoff → merge → verify deployment and migrations → tag merged `main` → publish notes → monitor alerts.
**Rule:** if blockers remain, **defer the tag rather than call staging "production."**

---

## 11. Standard Commands (PowerShell, from repo root)

```powershell
# Branch + local database
git switch -c feature/week-NN-topic
docker compose -f docker/compose.yaml up -d
docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev -c "SELECT current_database();"

# Scaffold (Week 1 only)
dotnet new sln -n Spendo -o backend
dotnet new webapi    -n Spendo.Api            -o backend/Spendo.Api
dotnet new classlib  -n Spendo.Application    -o backend/Spendo.Application
dotnet new classlib  -n Spendo.Infrastructure -o backend/Spendo.Infrastructure
dotnet new xunit     -n Spendo.Tests          -o backend/Spendo.Tests
dotnet sln backend/Spendo.slnx add backend/Spendo.Api backend/Spendo.Application backend/Spendo.Infrastructure backend/Spendo.Tests
dotnet add backend/Spendo.Api reference backend/Spendo.Application backend/Spendo.Infrastructure
dotnet add backend/Spendo.Infrastructure reference backend/Spendo.Application
dotnet add backend/Spendo.Tests reference backend/Spendo.Application
Set-Location frontend; ng new spendo --routing --style=scss --strict --test-runner=karma; Set-Location ..

# Migrations (only when the EF model actually changed — never create an empty migration)
dotnet ef migrations add <Name> --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api --output-dir Migrations
dotnet ef database update      --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api

# SQL verification — use Get-Content -Raw, NOT shell input redirection
Get-Content sql/verification/<file>.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev

# Tests and builds
dotnet test  backend/Spendo.slnx                      # add --collect:"XPlat Code Coverage" in Week 11
dotnet build backend/Spendo.slnx -c Release
Set-Location frontend/spendo
npm ci
npm run test -- --watch=false --browsers=ChromeHeadless   # add --code-coverage in Week 11
npx playwright test                                    # npx playwright install chromium if missing
npm audit --audit-level=high
npx ng build --configuration production
Set-Location ../..

# Packaging / PWA
docker build -f docker/api.Dockerfile -t spendo-api:local .
ng add @angular/pwa        # run inside frontend/spendo

# Backup rehearsal — synthetic, disposable DB only
docker compose -f docker/compose.yaml exec -T db pg_dump -U spendo -Fc -f /tmp/spendo-synthetic.dump spendo_dev
docker compose -f docker/compose.yaml exec -T db createdb -U spendo spendo_restore
docker compose -f docker/compose.yaml exec -T db pg_restore -U spendo -d spendo_restore --no-owner /tmp/spendo-synthetic.dump
docker compose -f docker/compose.yaml exec -T db rm /tmp/spendo-synthetic.dump

# Release — tag merged main only
git switch main; git pull --ff-only
git tag -a v1.0.0 -m "Spendo v1: authenticated INR expense tracking"
```

**Prerequisites:** .NET 10 SDK, Angular 21 CLI, a Node version supported by Angular 21, Docker Desktop. Use `dotnet user-secrets` for local API secrets. `db`, `spendo`, `spendo_dev` must match `docker/compose.yaml`. **Pin exact compatible patch versions** — do not assume latest patches are compatible. Check `ng new --help` for the installed CLI's test-runner option.

### Representative test assertions

| Layer      | Example                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| xUnit      | `Assert.True(result.IsHealthy)` · `Assert.ThrowsAsync<ValidationException>(() => service.CreateAsync(nonPositiveAmount))` · `Assert.Equal(HttpStatusCode.NotFound, foreignEdit.StatusCode)` · `Assert.False(await tokenStore.IsActiveAsync(reusedTokenHash))` · `Assert.Equal(expectedDefaultCount, categories.Count(c => c.OwnerId == newUser.Id))` · `Assert.Equal(300.00m, summary.MonthTotal)` · `Assert.Equal(summary.MonthTotal, analytics.CategoryTotals.Sum(x => x.Total))` · `Assert.Equal("validation_error", problem.Code)` · `Assert.DoesNotContain("=1+1", unescapedCsvCell)` · `Assert.Equal(HttpStatusCode.Unauthorized, anonymousList.StatusCode)` |
| Jasmine    | `expect(component).toBeTruthy()` · `expect(req.request.method).toBe('POST')` · `expect(req.request.headers.has('Authorization')).toBeTrue()` only for the configured API origin · `expect(toUtc(localDate).toISOString()).toBe(expectedInstant)` · `expect(paramsToFilter(filterToParams(filter))).toEqual(filter)` · `expect(series.length).toBe(0)` · `expect(sessionStorage.getItem('accessToken')).toBeNull()` · `expect(apiBaseUrl).toMatch(/^https:/)`                                                                                                                                                                                                       |
| Playwright | smoke `page.goto('/')` · add ₹250 Food/UPI → delete → empty state · register → login → expense → logout → blocked navigation · edit ₹250→₹300 → reload → correct page · filter + search + range → clear · chart's adjacent table shows INR totals · mocked 500 → generic incident message, no stack trace · export → inspect CSV headers, **no other owner's rows** · two synthetic accounts: A's expense never visible to B in list, dashboard or export                                                                                                                                                                                                          |

Use only synthetic accounts such as `tester@example.invalid`.

---

## 12. Cost — Azure, Central India

Retail pay-as-you-go USD, verified from the Azure Retail Prices API on **2026-10-05**. Monthly figures assume **730 hours**.

| Service                                  | SKU            | Unit price   | Per month |
| ---------------------------------------- | -------------- | ------------ | --------- |
| App Service (Linux)                      | B1             | $0.018/hr    | $13.14    |
| App Service (Linux)                      | B2             | $0.036/hr    | $26.28    |
| App Service (Linux)                      | P0v3           | $0.082/hr    | $59.86    |
| PostgreSQL Flexible                      | Burstable B1ms | $0.0245/hr   | $17.89    |
| PostgreSQL Flexible                      | Burstable B2s  | $0.098/hr    | $71.54    |
| PostgreSQL storage                       | —              | $0.131/GB/mo | —         |
| PostgreSQL backup (beyond included), LRS | —              | $0.095/GB/mo | —         |
| Blob Storage                             | Hot LRS        | $0.02/GB/mo  | —         |
| Static Web Apps                          | Free           | $0           | $0        |

> **Windows App Service P0v3 is $0.162/hr vs Linux $0.082/hr** — see NFR-43.

**Scenario A — MVP / early users ≈ $35/month:** Static Web Apps Free $0.00 + App Service Linux B1 $13.14 + PostgreSQL B1ms & 32 GB $22.08 + Blob Hot 10 GB $0.20.

**Scenario B — small production load ≈ $108/month:** Static Web Apps Free $0.00 + App Service Linux B2 $26.28 + PostgreSQL B2s & 64 GB $79.92 + Blob Hot 100 GB $2.00.

INR equivalents indicative at ₹88–90/USD; Azure bills INR at its own published rate.

**Cost levers:**

- Azure free account gives **12 months of PostgreSQL B1ms (750 hrs/mo)** and App Service F1. A `Compute - Free` PostgreSQL Flexible SKU is published at $0 in the target region — **confirm eligibility**. Can reduce Scenario A to near zero for year one.
- **Stop the PostgreSQL Flexible Server** when not developing (up to 7 days) — only storage is billed while stopped.
- Container Apps with scale-to-zero is cheaper for genuinely spiky traffic, **more expensive under steady load**.
- **Receipt storage is the dominant growth variable** (NFR-44).

**Deployment alternative (from notes, low-cost direction):** Angular → Cloudflare Pages, API → Render, PostgreSQL → Supabase, custom domain later. **Re-check provider pricing/limits when deployment begins.** The database must **not** be directly public-facing: users → frontend → API → PostgreSQL. A public domain is primarily needed for the user-facing frontend.

---

## 13. Out of Scope (v1)

| Item                                     | Rationale                                                                    |
| ---------------------------------------- | ---------------------------------------------------------------------------- |
| Android native app                       | Web-first; the API and data model here are its foundation                    |
| Offline entry & sync                     | Tied to Android; needs a conflict-resolution design that shouldn't be rushed |
| Budgets, limits, overspend alerts        | Separate feature with its own notification infrastructure                    |
| Recurring / scheduled expenses           | Adds scheduling and backfill complexity                                      |
| Income tracking & net cash flow          | v1 is expense-only                                                           |
| Bank / UPI linking & auto-import         | Aggregator partnerships, regulatory review, far higher security bar          |
| Shared / household / group expenses      | Changes the whole authorisation model                                        |
| Multi-currency & FX conversion           | Schema accommodates it (NFR-40); no UI, rates or conversion logic            |
| Profile-level timezone override          | Device timezone authoritative; editable date covers edge cases               |
| Headless-browser PDF rendering           | Security and operational cost unjustified for one layout (NFR-20)            |
| Receipt OCR / auto-extraction            | Image stored as attachment only; no parsing                                  |
| Multiple receipts per expense            | One image per expense                                                        |
| Split transactions across categories     | One category per expense                                                     |
| Tags and free-text search on notes       | Not selected for v1 filters                                                  |
| Saved dashboard layouts / filter presets | Fixed four-widget layout; URL bookmarking covers the common case             |
| Social login (Google / Apple)            | Email + password only                                                        |
| Two-factor authentication                | Session security hardened (FR-3); 2FA deferred                               |
| Bulk CSV import                          | Export only, not import                                                      |
| Email digests / scheduled reports        | Transactional email only                                                     |
| Localisation beyond `en-IN`              | Infrastructure ready, content not translated                                 |
| Dark mode                                | Cosmetic; deferred                                                           |
| Admin console, subscriptions, billing    | Free product in v1                                                           |

---

## 14. Open Questions & Decisions

### Product / architecture (PRD)

1. **Hosting shape** — App Service vs Container Apps. Drives NFR-33 availability design and cost profile.
2. **Email provider** — Azure Communication Services Email vs a third-party free tier. Confirm current quotas and domain-verification requirements.
3. **PDF library licensing** — QuestPDF Community licence has a revenue threshold; confirm applicability before committing.
4. **Export artefact lifetime** — how long generated CSV/PDF files remain downloadable before cleanup.
5. **Soft-delete vs export** — should soft-deleted expenses (within the 30-day window) appear in exports? **Current assumption: no.**

### Per-week decisions

| Week | Decision needed                                                                                                                                                                                                                                                                                                           |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Pin exact compatible patch versions, Node version, EF/Npgsql version. Confirm which Angular Material 21 components meet accessibility needs.                                                                                                                                                                              |
| 2    | Confirm initial payment methods and temporary category labels (Week 4 replaces labels with user-owned category IDs).                                                                                                                                                                                                      |
| 3    | Confirm same-site hosting / reverse proxy before cross-origin cookie setup; defer public deployment until cookie + CSRF design is verified. Confirm mail provider and production account-recovery delivery before public registration.                                                                                    |
| 4    | Agree default category names and whether users can rename/delete defaults before implementing those mutations.                                                                                                                                                                                                            |
| 5    | Confirm the business meaning of "expense date": local calendar date, exact instant, or both. Keep explicit mapping until decided.                                                                                                                                                                                         |
| 6    | Confirm reporting timezone (user vs browser) and treatment of future cross-currency totals.                                                                                                                                                                                                                               |
| 7    | Confirm whether "week" means ISO Monday–Sunday, and whether search includes notes as well as description.                                                                                                                                                                                                                 |
| 8    | Pick the chart library **only after** comparing supported Angular version, licensing, size and accessibility.                                                                                                                                                                                                             |
| 9    | Confirm retention and access control for operational logs before production deployment.                                                                                                                                                                                                                                   |
| 10   | Choose the export row limit and whether asynchronous large exports are needed in a future release.                                                                                                                                                                                                                        |
| 11   | Agree meaningful branch/line coverage thresholds **from the observed baseline** before making CI blocking.                                                                                                                                                                                                                |
| 12   | Approved monthly cost, Azure subscription/region, public domain, staging/production boundary. Who owns DNS, mail delivery, backups and operational alerts? If unresolved, keep CD as a **gated design, not an active deploy**.                                                                                            |
| 13   | Approve incident-response owner, retention, RPO/RTO and on-call coverage before launch.                                                                                                                                                                                                                                   |
| 14   | **Future currency setting:** when switching INR → GBP/USD/EUR, decide whether existing entries retain their original currency or convert at a stated historical rate. **Do not silently relabel amounts.** Specify exchange-rate source, effective timestamp, rounding and audit trail in a separate future feature plan. |

---

## 15. Daily Routine & Learning Plan

### Recommended daily routine (2 hours)

| Time   | Activity                          |
| ------ | --------------------------------- |
| 15 min | Plan what to accomplish today     |
| 75 min | Actual development                |
| 20 min | Testing / debugging / refactoring |
| 10 min | Commit and document what changed  |

Keep each day's work small enough to finish a meaningful piece.

### Fundamentals learning plan (parallel track)

**Purpose:** understand, review, debug and take responsibility for AI-assisted code; stay resilient in the job market. **Status:** active, reviewed monthly. **Created 2026-10-01.**

**Goals:** understand the code shipped, not just generate it · master C#, .NET Web API, SQL, Angular fundamentals · one portfolio capstone (API + SQL + Angular, synthetic data, CI) · add healthcare interop (FHIR/HL7) and security as differentiators · **park Python, AI and gaming until the core is solid.**

**Weekly rhythm (8–10 hrs/week):** Mon–Thu 45 min each — learn the topic, **no AI for the first 20 min** · Fri 30 min — rewrite one small piece by hand, then compare with AI · Sat 2–3 hrs practice/project · Sun 30 min — update learned log, review Anki, plan next week. Daily 10–15 min Anki. Weekly: read one senior dev's PR and write down why they made each choice; 1 hour light SQL practice from Week 1; write xUnit tests even for tiny exercises.

**Using AI correctly:** try first (15–20 min) → ask AI to **explain**, not just write → explain the answer back in your own words → **if you can't explain it, don't ship it** → ask AI to quiz you every Friday.

**Tracks (~16–18 weeks total):**

| #   | Track                                       | Duration | Core topics                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Check question                                                                                                                                                                                       |
| --- | ------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | C# fundamentals (Weeks 0A–0C)               | 3 wks    | **0A:** value vs reference types, stack/heap, nullability (`int?`, NRT, `?.`, `??`, `??=`), strings/`StringBuilder`/culture, `var`/`const`/`readonly`/`static`, switch expressions + pattern matching, `record`, `record struct`, `enum`, tuples, `init`. **0B:** encapsulation/inheritance/polymorphism/abstraction, `abstract` vs `interface`, `virtual`/`override`/`sealed`, access modifiers, properties vs fields, constructors, **composition over inheritance**, `Equals`/`GetHashCode`/`IEquatable<T>`/`ToString`. **0C:** generics + constraints, delegates/`Func`/`Action`/`Predicate`/lambdas/closures/events, collection complexity, `IEnumerable`/`yield return`/`IDisposable`/`using`, exceptions (custom, throw vs result, **never swallow**), GC + finalizers vs `Dispose`, extension methods, `Span<T>` awareness | Why does a copied struct not affect the original but a copied class reference does? · When pick an interface over an abstract class? · What does a closure capture and why can that cause loop bugs? |
| 2   | Async, LINQ, EF Core                        | 3 wks    | Task model, `async`/`await`, `ConfigureAwait`, cancellation tokens · deferred execution, `IEnumerable` vs `IQueryable`, grouping, joins · change tracking, `AsNoTracking`, **N+1**, eager vs lazy loading, migrations                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Why can `.Result` deadlock? · When does a LINQ query actually run? · Can you predict the generated SQL?                                                                                              |
| 3   | .NET Web API (Weeks 4–7)                    | 3–4 wks  | DI + service lifetimes, **captive dependencies**, middleware pipeline, global exception handling, structured logging (**never log PHI or secrets**), REST design/validation/status codes/versioning, authN vs authZ, OAuth2/OIDC, JWT, **OWASP Top 10 — review your own API for 3 risks and fix them**, audit logging                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Explain a captive-dependency bug, and authN vs authZ                                                                                                                                                 |
| 4   | SQL (Weeks 8–10)                            | 3 wks    | Execution plans, scans vs seeks, clustered/non-clustered/covering indexes, statistics, ACID, isolation levels (`READ COMMITTED` vs `SNAPSHOT`), locking, **blocking vs deadlocks**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Why was an index used or not? Justify an isolation level                                                                                                                                             |
| 5   | Angular (Weeks 11–13)                       | 2–3 wks  | RxJS `switchMap`/`mergeMap`/`concatMap`, unsubscribing, change detection, `OnPush`, signals, state management, component design, accessibility, performance                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Why did you choose `switchMap`?                                                                                                                                                                      |
| 6   | Testing & architecture (Weeks 14–15)        | 2 wks    | xUnit, Moq/NSubstitute, unit vs integration, Arrange-Act-Assert, TDD for one small feature, SOLID, repository/service patterns, clean architecture                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Do tests fail when you break the code on purpose? Can you justify each design choice in one sentence?                                                                                                |
| 7   | Capstone (start ~Week 8, finish Week 16–18) | ongoing  | Small FHIR-style .NET API + Angular front end, **synthetic data only**: EF Core + SQL with proper indexes, unit + integration tests, authentication, input validation, audit logging, synthetic FHIR Patient endpoint, GitHub Actions CI, README explaining design decisions                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | —                                                                                                                                                                                                    |

**Practice exercises:** model a synthetic `Appointment` as class/struct/record and compare equality + copy behaviour · synthetic `Notification` hierarchy (email/SMS) behind an interface, then refactor to composition · generic `Result<T>` + in-memory generic repository with a custom exception · convert a sync call to async and explain what the thread does at each `await` · 10 LINQ problems, translate 5 to SQL · enable SQL logging and fix one N+1 · small API with custom middleware, global error handler, auth and validation · tune 3 slow queries, reproduce and fix a deadlock with two sessions · search-as-you-type with debounce + cancellation · refactor a messy class into small testable parts.

**Healthcare & security, small doses:** FHIR resources (Patient, Encounter, Observation) · HL7 v2 structure (MSH, PID, OBX) · HIPAA basics, audit logging, data privacy.

**Habits for "I forget things":** keep a markdown **learned log** (what, why, tiny example) · one **Anki card** per concept · **predict output in a comment before running** · one repo, one folder per week, short README in each · rewrite one piece by hand each week without AI.

**Career-protection tasks:** Week 1 update resume + LinkedIn and start a brag document · Week 4 reach out to 3 colleagues/former teammates · Week 8 two mock interviews (SQL + system design basics) · End: publish the capstone and write a short post about it.

**Progress tracking:** each Sunday rate yourself 1–5 on that week's Check question. **Below 4 = repeat before moving on.**

**Parked until after the capstone:** Python basics + a deterministic automation script · practical AI tooling (LLM APIs, RAG, evaluation, limits) · optional gaming side-track (Unity + C#, or a leaderboard API with .NET + SQL).

**Resources:** Microsoft Learn C# path and official C# docs · _C# in Depth_ (Jon Skeet, later) · Exercism C# track · official Angular and RxJS docs.

---

## 16. Non-Negotiable Rules — Quick Reference

1. **Synthetic data only.** No real personal/patient data in code, tests, logs, fixtures, artifacts or notes.
2. **Never commit** passwords, tokens, connection strings, `.env`, DB dumps, or Playwright auth state.
3. **Ownership is server-derived** from the auth token. Never accept a `UserId` from a client. Cross-owner lookups return **404**, not 403.
4. **Money is decimal.** `numeric` in PostgreSQL, `decimal` in C#. Never binary floating point. Aggregate in the database.
5. **`expense_date` is a `date`** and is the sole source of truth for reporting period. Never derive it from `created_at` or a UTC conversion.
6. **Parameterised queries only.** Escape `%`/`_` in search. Sort columns from a fixed allowlist.
7. **Filtering, sorting, aggregation and pagination are server-side.** Stable order requires a tie-breaker.
8. **Access token in memory; refresh token in an HttpOnly cookie.** Rotate, hash, revoke the family on reuse.
9. **Never cache authenticated API data, tokens or expense data in the service worker.**
10. **Neutralise CSV formula prefixes.** Escape all user text in CSV and PDF.
11. **Every chart needs an accessible table equivalent.** WCAG 2.2 AA, keyboard-first, 360px-safe.
12. **No stack traces or account-existence disclosure** in responses. RFC 9457 problem details with correlation IDs.
13. **Redact** authorization headers, cookies, reset links, expense free text and user identifiers from logs and telemetry.
14. **Migrations are the schema source of truth**; `sql/` holds verification only. Never run PR-branch migrations against production. Back up before migrating.
15. **Tag merged `main` only**, after checks pass. Never tag a feature branch. If blockers remain, **defer the tag**.
16. **Verify before adopting:** library licence, Angular/.NET compatibility, advisories, size, accessibility, and current cloud pricing/quotas.
17. **Fix test isolation, not flakiness** — no arbitrary sleeps or retries.
18. **Linux compute only.** Infrastructure as code (Bicep).
19. **The database is never public-facing.** Users → frontend → API → PostgreSQL.
20. **Build V1 simply, but never in a way that prevents V2.** Avoid over-engineering.
