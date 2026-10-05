# Product Requirements Document — Spendo

## 1. Feature Name

**Expense Capture & Insights Dashboard (Spendo v1 — Web)**

## 2. Epic

- **Parent Epic:** Spendo — Personal Expense Tracking Platform
- **Epic PRD:** `/docs/ways-of-work/plan/spendo/epic-prd.md` *(to be authored)*
- **Epic Architecture:** `/docs/ways-of-work/plan/spendo/epic-architecture.md` *(to be authored)*
- **Position in Epic:** Foundational feature. Android client, budgets/alerts, recurring expenses, and bank sync are downstream features that build on the API and data model established here.

## 3. Goal

### Problem

Individuals who want to understand where their money goes are stuck between two bad options: spreadsheets that require manual formatting and offer no insight, or heavyweight finance apps that demand bank credentials, push ads, and bury simple expense logging under features the user never asked for. Most people abandon tracking within two weeks because logging a single expense takes too many taps and the resulting data is never surfaced in a way that changes behaviour. The core unmet need is a fast, private, low-friction way to record an expense and immediately see what it means in context of the month's spending.

### Solution

Spendo provides a single-screen expense entry form that captures an expense in under 15 seconds, backed by a dashboard that answers "how much have I spent, on what, and is that more than last period?" without configuration. Users get custom categories so the taxonomy matches their life, optional receipt photos for proof, and export so their data is never locked in. Filters on date, category, payment method, and amount let users drill from a headline number down to the individual transaction that caused it.

### Impact

| Metric | Target |
|---|---|
| Time to log one expense (median, returning user) | ≤ 15 seconds |
| D14 retention (users logging ≥1 expense on day 14) | ≥ 35% |
| Expenses logged per active user per week | ≥ 7 |
| Users who apply at least one dashboard filter in week 1 | ≥ 50% |
| Signup → first expense logged conversion | ≥ 70% |
| Dashboard p95 load time (1 year of data) | ≤ 1.5s |

## 4. User Personas

**Primary — "Priya, the Budget-Conscious Professional"**
28, salaried, urban India. Pays via UPI, card, and occasional cash. Wants to know why her month ran short. Low tolerance for setup. Will not connect her bank account to a third-party app. Uses desktop at work, phone everywhere else.

**Secondary — "Arun, the Freelancer"**
34, variable income, needs to separate personal and business-ish spending and export records for his accountant at year end. Photographs receipts. Cares about data export and category accuracy more than charts.

**Tertiary — "Sana, the Student"**
21, small frequent transactions, cash-heavy. Wants a simple monthly total and category breakdown. Price-sensitive; will only use a free tier.

**Non-persona (explicitly not served in v1):** households sharing a budget, small business accounting, investment tracking.

## 5. User Stories

### Account & Access

1. As a **new user**, I want to sign up with email and password so that my expense data is saved and available across devices.
2. As a **returning user**, I want to stay signed in on a trusted browser so that I don't re-authenticate every visit.
3. As a **user**, I want to reset a forgotten password via email so that I don't lose access to my history.
4. As a **privacy-conscious user**, I want to delete my account and all data permanently so that I retain control over my financial records.

### Expense Entry

5. As a **user**, I want to record an expense with amount, date, and category so that I can build a spending history.
6. As a **user**, I want the date to default to today and the form to remember my last-used payment method so that logging a typical expense takes minimal input.
7. As a **user**, I want to add an optional note and merchant name so that I can recall the context later.
8. As a **user**, I want to attach a photo of a receipt so that I have proof of purchase attached to the record.
9. As a **user**, I want to save an expense and immediately start another so that I can log several expenses in one sitting.
10. As a **user**, I want to edit or delete an expense I entered incorrectly so that my totals stay accurate.
11. As a **user**, I want to be warned when I submit an expense that is a near-duplicate of one logged in the last few minutes so that I don't accidentally double-count.

### Categories

12. As a **user**, I want a set of sensible default categories on signup so that I can start logging without setup.
13. As a **user**, I want to create, rename, re-colour, and archive my own categories so that the taxonomy matches how I actually think about spending.
14. As a **user**, I want to be prevented from deleting a category that has expenses, and instead be offered to reassign or archive it, so that I never silently lose data.

### Dashboard & Filters

15. As a **user**, I want to see my total spend for the current month and how it compares to last month so that I immediately know if I'm overspending.
16. As a **user**, I want to see spend broken down by category as a donut chart so that I can identify my largest spending areas at a glance.
17. As a **user**, I want to see a spend trend over time so that I can spot patterns and spikes.
18. As a **user**, I want to see my most recent transactions so that I can verify my latest entries were recorded.
19. As a **user**, I want to filter the entire dashboard by date range (presets and custom) so that I can analyse any period.
20. As a **user**, I want to filter by one or more categories so that I can isolate specific spending areas.
21. As a **user**, I want to filter by payment method so that I can reconcile against my card statement or track cash usage.
22. As a **user**, I want to filter by an amount range so that I can find large or small transactions.
23. As a **user**, I want to click a category in the chart to drill into its transactions so that I can see what drove the number.
24. As a **user**, I want my filter selections reflected in the URL so that I can bookmark and share a view with myself across devices.
25. As a **user**, I want a clear "no data" state with a call to action when filters return nothing so that I'm not confused by an empty screen.

### Transactions & Export

26. As a **user**, I want to browse a paginated, sortable list of all expenses matching my filters so that I can review history in detail.
27. As a **user**, I want to export the filtered expense set to CSV so that I can analyse it in a spreadsheet or send it to my accountant.
28. As a **user**, I want to export a PDF summary of the filtered period so that I have a shareable, human-readable report.

## 6. Requirements

### 6.1 Functional Requirements

#### Authentication & Account

- **FR-1:** Users register with email address and password. Email must be verified via a tokenised link before the account is fully active; unverified users may log expenses but cannot export.
- **FR-2:** Password minimum 12 characters, checked against a common-password denylist. Passwords stored using a memory-hard hash (Argon2id or bcrypt cost ≥ 12).
- **FR-3:** Authentication issues a short-lived access token (≤ 15 min) and a rotating refresh token stored in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie. Refresh token reuse must invalidate the entire token family.
- **FR-4:** Password reset via single-use, time-limited (30 min) token sent by email. Reset invalidates all active sessions.
- **FR-5:** Login endpoint is rate-limited (e.g. 5 failures per account per 15 min, plus per-IP limits) and returns a generic failure message that does not reveal whether the account exists.
- **FR-6:** Users can permanently delete their account; all expenses, categories, and receipt images are hard-deleted within 30 days, with immediate revocation of access.

#### Expense Entry

- **FR-7:** The expense form captures: **amount** (required), **date** (required, defaults to today in the device's local timezone), **category** (required), **payment method** (required — Cash, Card, UPI, Bank Transfer, Other), **merchant/description** (optional, ≤ 120 chars), **note** (optional, ≤ 500 chars), **receipt image** (optional).
- **FR-8:** Amount accepts up to 2 decimal places, must be > 0, and must be ≤ 10,000,000. Stored as `numeric(14,2)` — never as a floating-point type.
- **FR-9:** Date must not be in the future (relative to the client's local date) and must not be earlier than 10 years before today.
- **FR-10:** The form pre-selects the user's most recently used payment method and most recently used category.
- **FR-11:** "Save & Add Another" persists the expense and resets only amount, merchant, note, and receipt — retaining date, category, and payment method.
- **FR-12:** Each submission includes a client-generated idempotency key; the API must not create duplicate records for retried requests with the same key.
- **FR-13:** If an expense with identical amount, category, and date was created by the same user within the last 5 minutes, the UI presents a non-blocking duplicate confirmation before saving.
- **FR-14:** Expenses can be edited (all fields) or deleted. Deletion is a soft delete with a 10-second in-session undo; records are purged after 30 days.

#### Timezone Handling

- **FR-15:** The client resolves the user's local calendar date from the device and transmits `expenseDate` **explicitly as a calendar date** (`YYYY-MM-DD`). The server must never derive the expense date by converting a UTC timestamp.
- **FR-16:** Every expense-create and expense-update request includes the device's IANA timezone identifier (e.g. `Asia/Kolkata`), persisted on the record as `captured_timezone`.
- **FR-17:** Every dashboard, report, and export request includes the device's IANA timezone identifier. All relative period boundaries ("This Month", "Last 30 Days", period-over-period comparison windows) are resolved server-side in that timezone.
- **FR-18:** Because the date field is user-editable, a user whose device timezone produces an unexpected date can correct it manually. No profile-level timezone setting is required in v1.

#### Receipt Attachments

- **FR-19:** One image per expense. Accepted types: JPEG, PNG, WebP, HEIC. Max 10 MB pre-compression.
- **FR-20:** File type validated by magic-byte inspection on the server, not by extension or client-supplied MIME type.
- **FR-21:** Images are re-encoded server-side (stripping EXIF, including GPS) and stored with a server-generated opaque filename. Original filenames are never used on disk or in URLs.
- **FR-22:** Images are stored in a **private Azure Blob Storage container**. They are served only via **user delegation SAS** URLs with a short TTL (≤ 15 min), scoped per request. The storage account key is never used to sign URLs, and no publicly enumerable paths exist.
- **FR-23:** The API authenticates to Blob Storage using a **managed identity**. No storage credentials are present in application configuration.
- **FR-24:** Client-side downscaling to max 2000px on the long edge before upload, to reduce transfer size and storage cost.

#### Categories

- **FR-25:** On signup, the account is seeded with default categories: Food & Dining, Groceries, Transport, Utilities, Rent, Shopping, Health, Entertainment, Education, Travel, Other.
- **FR-26:** Users can create categories with a name (≤ 40 chars, unique per user, case-insensitive), colour, and optional icon. Maximum 50 active categories per user.
- **FR-27:** Categories can be renamed and re-coloured; changes apply retroactively to all historical expenses.
- **FR-28:** Categories can be archived — hidden from the entry form but still rendered in historical reports and filters.
- **FR-29:** Deleting a category with existing expenses is blocked; the user is offered reassignment to another category or archival instead.

#### Dashboard

- **FR-30:** Dashboard defaults to the **current calendar month** (in the device timezone) with no other filters applied.
- **FR-31:** **Total Spend KPI** — sum of the filtered set, with absolute and percentage change against the immediately preceding equivalent period. Also displays transaction count and average transaction value.
- **FR-32:** **Category Breakdown** — donut chart of spend by category for the filtered set. Top 8 categories shown individually, remainder grouped as "Other" with a tooltip listing contents. Clicking a segment applies that category as a filter.
- **FR-33:** **Spend Trend** — bar/line chart over the filtered range. Granularity auto-selects: daily (range ≤ 31 days), weekly (≤ 180 days), monthly (> 180 days). User can override granularity.
- **FR-34:** **Recent Transactions** — latest 10 expenses in the filtered set, showing date, merchant, category, payment method, amount, and a receipt indicator. Each row links to the full transaction list.
- **FR-35:** All four widgets respond to the same global filter state.
- **FR-36:** Every chart has an accessible tabular equivalent reachable via a "view as table" toggle.

#### Filtering

- **FR-37:** **Date range** — presets (Today, This Week, This Month, Last Month, Last 3 Months, This Year, Custom). Custom allows any start/end within the user's data history.
- **FR-38:** **Category** — multi-select, including archived categories.
- **FR-39:** **Payment method** — multi-select.
- **FR-40:** **Amount range** — min and/or max, either bound optional.
- **FR-41:** Filters combine with AND logic across dimensions and OR within a multi-select dimension.
- **FR-42:** Active filters render as removable chips with a "Clear all" action and a visible result count.
- **FR-43:** Filter state is serialised to URL query parameters and restored on load, back/forward navigation, and refresh.
- **FR-44:** All filtering, sorting, aggregation, and pagination execute server-side. The client must never receive the full dataset to filter locally.

#### Transaction List

- **FR-45:** Paginated list (25/50/100 per page) honouring the active filters.
- **FR-46:** Sortable by date, amount, category, or merchant, ascending or descending. Default: date descending.
- **FR-47:** Inline edit and delete actions per row.

#### Export

- **FR-48:** CSV export of the currently filtered set with columns: Date, Amount, Currency, Category, Payment Method, Merchant, Note, Has Receipt.
- **FR-49:** PDF export containing the KPI summary, category breakdown, trend chart, and full transaction table for the filtered period.
- **FR-50:** PDF documents are composed **server-side using a .NET document library** (e.g. QuestPDF). Charts are rendered server-side to SVG/PNG and embedded. A headless browser is explicitly not used in v1 — see NFR-20.
- **FR-51:** All user-supplied text (merchant, note, category name) is escaped/sanitised before being embedded in CSV or PDF output.
- **FR-52:** Export generation is **asynchronous** — the client enqueues a request and polls for completion, so a slow render never holds an HTTP request open.
- **FR-53:** Exports are capped at 50,000 rows; requests exceeding the cap prompt the user to narrow their filters.
- **FR-54:** Export is rate-limited to 10 requests per user per hour.
- **FR-55:** CSV fields beginning with `=`, `+`, `-`, `@`, tab, or carriage return are prefixed with a single quote to prevent formula injection in spreadsheet applications.

### 6.2 Non-Functional Requirements

#### Performance

- **NFR-1:** Dashboard p95 end-to-end load ≤ 1.5s and p99 ≤ 3s for an account with 10,000 expenses spanning 3 years.
- **NFR-2:** Expense create/update API p95 ≤ 300ms excluding image upload.
- **NFR-3:** Filter re-query p95 ≤ 500ms.
- **NFR-4:** Angular initial bundle ≤ 300 KB gzipped; dashboard route lazy-loaded; charting library loaded on demand.
- **NFR-5:** Database indexed on `(user_id, expense_date DESC)`, `(user_id, category_id)`, and `(user_id, payment_method)`. Aggregation queries must use indexes, verified via `EXPLAIN ANALYZE` in CI against a seeded 1M-row dataset.
- **NFR-6:** PostgreSQL table partitioning deferred, but the schema must not preclude future range partitioning on `expense_date`.
- **NFR-7:** Performance NFRs are validated against the **actual production database SKU** (burstable tier) under a seeded dataset — never against a developer machine. If targets fail, the remedy is indexing and pre-aggregation before scaling the SKU.

#### Security

- **NFR-8:** Every data-access query is scoped by the authenticated `user_id` derived from the token — never from a client-supplied parameter. Cross-tenant access must be structurally impossible, enforced at the repository layer and covered by automated tests.
- **NFR-9:** PostgreSQL Row Level Security enabled on `expenses`, `categories`, and `receipts` as defence in depth, with the application connecting under a non-superuser, non-`BYPASSRLS` role.
- **NFR-10:** All inputs validated server-side against an allowlist. Client-side validation is a UX affordance only and is never trusted.
- **NFR-11:** All database access via parameterised queries / EF Core LINQ. Dynamic SQL for sorting must map user input to a fixed allowlist of column names — never interpolate.
- **NFR-12:** Content Security Policy with `default-src 'self'`, no `unsafe-inline`/`unsafe-eval`, nonce-based scripts. Plus `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.
- **NFR-13:** CORS restricted to known origins. No wildcard origins with credentials.
- **NFR-14:** Secrets sourced from Azure Key Vault or environment configuration. No credentials, connection strings, or keys in source control. Managed identity preferred over any stored secret.
- **NFR-15:** TLS 1.2+ in transit; encryption at rest for database and blob storage.
- **NFR-16:** Structured audit logging for auth events, exports, and account deletion. Logs must never contain passwords, tokens, raw amounts tied to identity, or full email addresses.
- **NFR-17:** Dependency vulnerability scanning in CI; build fails on high/critical severity findings.
- **NFR-18:** Global API rate limiting per authenticated user and per IP, with stricter limits on auth and export endpoints.
- **NFR-19:** Error responses are generic to the client; stack traces and framework details are never returned in production.
- **NFR-20:** Headless-browser PDF rendering is **out of scope for v1** due to its attack surface (HTML/script injection into the render context enabling SSRF or local file disclosure) and its memory/image-size cost. If adopted later, it must run sandboxed, in an isolated worker, with egress restrictions and hard render timeouts.

#### Email & Deliverability

- **NFR-21:** Transactional email (verification, password reset) is sent via a dedicated email service — **not** a consumer mailbox provider's SMTP, which breaches ToS, caps volume, and destroys deliverability.
- **NFR-22:** The sending domain must have **SPF, DKIM, and DMARC** records correctly configured.
- **NFR-23:** Transactional mail is isolated on its own subdomain, separate from any future marketing mail, so marketing complaint rates cannot degrade password-reset deliverability.
- **NFR-24:** Bounce and complaint webhooks are consumed and logged so delivery failures are observable.

#### Privacy & Compliance

- **NFR-25:** Data minimisation — collect only email and expense data. No device fingerprinting, no third-party analytics that transmit expense content.
- **NFR-26:** Users can export all their data and request full deletion (GDPR-style rights), with deletion completed within 30 days.
- **NFR-27:** Receipt images are stripped of EXIF/GPS metadata before persistence.
- **NFR-28:** Privacy policy and terms presented at signup with explicit consent capture.

#### Accessibility

- **NFR-29:** WCAG 2.2 Level AA conformance. Full keyboard operability, visible focus indicators, logical tab order.
- **NFR-30:** Charts are not colour-only — patterns, labels, and the tabular equivalent convey the same information. Minimum 4.5:1 contrast for text, 3:1 for UI components.
- **NFR-31:** Form errors are programmatically associated with inputs and announced to screen readers via live regions.
- **NFR-32:** Respects `prefers-reduced-motion`; chart animations disabled when set.

#### Reliability & Operations

- **NFR-33:** 99.5% monthly availability target for v1.
- **NFR-34:** Daily automated database backups with point-in-time recovery, retained 30 days. Restore procedure tested quarterly.
- **NFR-35:** Health and readiness endpoints, structured JSON logging with correlation IDs, and alerting on error rate and latency SLO breaches.
- **NFR-36:** Zero-downtime deployments; all database migrations are backward-compatible with the previously deployed application version.

#### Compatibility & Responsiveness

- **NFR-37:** Supports latest two major versions of Chrome, Edge, Firefox, and Safari.
- **NFR-38:** Fully responsive from 360px to 1920px. Mobile web must be fully usable, since Android ships later.
- **NFR-39:** Angular localisation infrastructure in place; v1 ships `en-IN` only. All currency and date formatting goes through locale-aware pipes — no hardcoded symbols or formats.

#### Data Model

- **NFR-40:** Single currency (INR) in v1, but every monetary record stores an explicit ISO-4217 currency code to avoid a destructive migration when multi-currency arrives.
- **NFR-41:** System timestamps (`created_at`, `updated_at`) stored as UTC `timestamptz`. The **`expense_date` is a `date` column** and is the sole source of truth for which reporting period an expense belongs to. Aggregation must never derive the period from `created_at`.
- **NFR-42:** Primary keys are UUIDv7 — not sequential integers — to prevent enumeration via API responses.

#### Hosting & Cost

- **NFR-43:** All compute runs on **Linux** App Service / container plans. Windows plans cost roughly 4.6× more for equivalent SKUs in the target region and provide no benefit to this stack.
- **NFR-44:** Receipt image storage is the primary cost-growth variable. Client-side downscaling (FR-24) is a cost control, not only a UX affordance — it represents an approximately 20× difference in stored bytes per receipt.
- **NFR-45:** Infrastructure defined as code (Bicep) so environments are reproducible and cost is auditable.

## 7. Acceptance Criteria

### US-5 / US-6 / US-9 — Expense Entry

- [ ] **Given** I am signed in, **when** I open the expense form, **then** the date field is pre-filled with today and my last-used category and payment method are pre-selected.
- [ ] **Given** I enter amount `450.50`, today's date, category "Food & Dining", and payment method "UPI", **when** I click Save, **then** the expense persists and a success confirmation appears within 1 second.
- [ ] **Given** I submit with an empty amount, **when** validation runs, **then** an inline error "Enter an amount greater than 0" is shown, focus moves to the amount field, and the error is announced by screen readers.
- [ ] **Given** I enter `0` or a negative amount, **when** I submit, **then** submission is rejected both client-side and server-side.
- [ ] **Given** I enter a date in the future, **when** I submit, **then** submission is rejected with "Date cannot be in the future".
- [ ] **Given** I enter `12.345`, **when** the field loses focus, **then** it is rejected or normalised to 2 decimal places with a visible notice.
- [ ] **Given** I click "Save & Add Another", **when** the expense saves, **then** amount, merchant, note, and receipt clear while date, category, and payment method persist, and focus returns to the amount field.
- [ ] **Given** the network drops and my client retries with the same idempotency key, **when** both requests reach the server, **then** exactly one expense record exists.
- [ ] **Given** I complete the form using only the keyboard, **when** I tab through it, **then** every control is reachable in logical order with a visible focus indicator.

### US-5 (Timezone) — Device-Local Dates

- [ ] **Given** my device is set to `Asia/Kolkata` and local time is 01:00 on 5 October, **when** I open the form, **then** the date defaults to `2026-10-05` — not `2026-10-04` (the UTC date).
- [ ] **Given** I submit an expense, **when** the request is inspected, **then** it contains an explicit `expenseDate` of `YYYY-MM-DD` and an IANA timezone identifier.
- [ ] **Given** an expense is stored, **when** the record is inspected, **then** `captured_timezone` holds the IANA identifier the expense was logged in.
- [ ] **Given** I log an expense in `Asia/Kolkata` and later open the dashboard from a device set to `America/New_York`, **when** the monthly total renders, **then** the expense remains in its originally recorded date and does not shift between months.
- [ ] **Given** I request "This Month" from a device in `Asia/Dubai`, **when** the server resolves the period, **then** boundaries are computed in `Asia/Dubai`, not UTC and not a server-default timezone.
- [ ] **Given** the auto-filled date is not what I intended, **when** I change it manually, **then** the edited date is honoured without warning.

### US-8 — Receipt Attachment

- [ ] **Given** I attach a 4 MB JPEG, **when** it uploads, **then** a thumbnail preview appears and the image is linked to the expense.
- [ ] **Given** I attach a 15 MB file, **when** I submit, **then** it is rejected with "Image must be under 10 MB" before upload begins.
- [ ] **Given** I rename `payload.exe` to `receipt.jpg` and upload it, **when** the server inspects the file, **then** it is rejected based on magic-byte analysis and nothing is persisted.
- [ ] **Given** I upload a photo containing GPS EXIF data, **when** I later retrieve the stored image, **then** no EXIF or GPS metadata is present.
- [ ] **Given** I upload a 4000px-wide photo, **when** it is transmitted, **then** the client has downscaled it to ≤ 2000px on the long edge.
- [ ] **Given** I copy a receipt SAS URL and open it after its TTL expires, **then** access is denied.
- [ ] **Given** the blob container is probed directly without a SAS token, **then** access is denied (container is private).
- [ ] **Given** the deployed application configuration is inspected, **then** no storage account key or connection string with credentials is present — access is via managed identity.
- [ ] **Given** another user's receipt ID, **when** I request it with my own valid token, **then** the API returns 404 (not 403, to avoid confirming existence).

### US-11 — Duplicate Detection

- [ ] **Given** I logged ₹200 / Transport / today three minutes ago, **when** I submit an identical expense, **then** a dismissible dialog asks me to confirm, and choosing "Save anyway" creates the record.

### US-12 to US-14 — Categories

- [ ] **Given** I am a newly registered user, **when** I first open the expense form, **then** 11 default categories are available.
- [ ] **Given** I create a category named "Pet Care", **when** I save, **then** it appears in the entry form and in the category filter.
- [ ] **Given** I already have a category "Travel", **when** I try to create "travel", **then** it is rejected as a duplicate (case-insensitive).
- [ ] **Given** I rename "Food & Dining" to "Eating Out", **when** I view the dashboard, **then** all historical expenses display under the new name.
- [ ] **Given** a category has 30 linked expenses, **when** I attempt to delete it, **then** deletion is blocked and I am offered reassign or archive.
- [ ] **Given** I archive a category, **when** I open the entry form, **then** it is absent from the dropdown, but **when** I open the dashboard filter, **then** it is still selectable.

### US-15 to US-18 — Dashboard

- [ ] **Given** I have expenses in the current month, **when** the dashboard loads, **then** the Total Spend KPI equals the exact sum of those expenses to 2 decimal places.
- [ ] **Given** I spent ₹10,000 this month and ₹8,000 last month, **when** the dashboard loads, **then** it displays "+₹2,000 (+25%) vs last month" with an appropriate directional indicator.
- [ ] **Given** the previous period had zero spend, **when** the comparison renders, **then** it displays "No data for comparison" rather than a division-by-zero error or "∞%".
- [ ] **Given** I have expenses across 12 categories, **when** the donut chart renders, **then** the top 8 appear individually and the remaining 4 are grouped as "Other", with chart segment percentages summing to 100%.
- [ ] **Given** I select a 7-day range, **when** the trend chart renders, **then** granularity is daily; **given** a 12-month range, **then** granularity is monthly.
- [ ] **Given** I click "view as table" on any chart, **when** the toggle activates, **then** an accessible table with identical values is displayed.
- [ ] **Given** I have 50 expenses this month, **when** the Recent Transactions widget renders, **then** exactly the 10 most recent are shown, newest first.
- [ ] **Given** an account with 10,000 expenses over 3 years **on the production database SKU**, **when** I load the dashboard, **then** p95 load time is ≤ 1.5s measured over 100 runs.

### US-19 to US-25 — Filtering

- [ ] **Given** I select the "Last Month" preset, **when** the filter applies, **then** all four widgets update to that period and the result count reflects the filtered set.
- [ ] **Given** I select categories "Groceries" and "Transport", **when** the filter applies, **then** results include expenses in either category and exclude all others.
- [ ] **Given** I set payment method to "Cash" and amount range ₹500–₹2000, **when** both apply, **then** only cash expenses between ₹500 and ₹2000 inclusive are returned.
- [ ] **Given** I set a minimum amount greater than the maximum, **when** I apply, **then** a validation error is shown and no query is issued.
- [ ] **Given** a custom date range with end before start, **when** I apply, **then** it is rejected with a clear message.
- [ ] **Given** active filters, **when** I view the filter bar, **then** each is a removable chip and "Clear all" resets to the default current-month view.
- [ ] **Given** filters are applied, **when** I copy the URL and open it in a new tab, **then** the identical filtered view is restored.
- [ ] **Given** filters are applied, **when** I press browser Back, **then** the previous filter state is restored.
- [ ] **Given** I click the "Shopping" segment of the donut chart, **when** the drill-down occurs, **then** "Shopping" is added as an active filter chip and all widgets update.
- [ ] **Given** filters that match zero expenses, **when** results render, **then** an empty state explains why and offers "Clear filters".
- [ ] **Given** I craft an API request with another user's ID in the query string, **when** the server processes it, **then** the parameter is ignored and only my own data is returned.

### US-26 / US-27 — Transaction List

- [ ] **Given** 120 expenses match my filters, **when** I view the list at 25 per page, **then** 5 pages are available and the total count is displayed.
- [ ] **Given** I sort by amount descending, **when** the list refreshes, **then** the highest amount in the entire filtered set appears first — not merely the highest on the current page.
- [ ] **Given** I supply an arbitrary string as the sort column via the API, **when** the server validates it, **then** the request is rejected against the column allowlist and no SQL is constructed from the input.

### US-28 — Export

- [ ] **Given** filters matching 300 expenses, **when** I export CSV, **then** the file contains exactly 300 data rows plus a header, and the sum of the Amount column equals the Total Spend KPI.
- [ ] **Given** an expense note of `=cmd|'/c calc'!A1`, **when** I open the exported CSV in Excel, **then** the value is rendered as inert text and no formula executes.
- [ ] **Given** a note containing commas, quotes, and newlines, **when** exported, **then** the CSV is correctly quoted and parses without corruption.
- [ ] **Given** filters matching 60,000 expenses, **when** I request an export, **then** I am prompted to narrow the range rather than receiving a truncated file.
- [ ] **Given** I export a PDF, **when** it generates, **then** it contains the KPI summary, both charts, and the transaction table for the filtered period.
- [ ] **Given** a merchant name containing HTML or script markup, **when** the PDF is produced, **then** the markup appears as literal escaped text and is not interpreted.
- [ ] **Given** I request a large export, **when** the request is issued, **then** it returns immediately with a job reference and the file becomes available via polling — the HTTP request is not held open.
- [ ] **Given** the deployed container image is inspected, **then** no headless browser runtime is present.
- [ ] **Given** I have requested 10 exports in the past hour, **when** I request an 11th, **then** I receive a rate-limit message with a retry time.

### US-1 to US-4 — Authentication

- [ ] **Given** I register with a valid email, **when** I submit, **then** a verification email is sent and I am told to check my inbox.
- [ ] **Given** I register with a password of 8 characters, **when** I submit, **then** it is rejected with the policy explained.
- [ ] **Given** I enter a valid email with the wrong password, **when** I submit, **then** the error reads "Invalid email or password" — identical to the error for a non-existent account.
- [ ] **Given** 5 consecutive failed attempts, **when** I try a 6th, **then** I am temporarily locked out with a retry-after message.
- [ ] **Given** a password reset link, **when** I use it twice, **then** the second attempt is rejected as already consumed.
- [ ] **Given** I complete a password reset, **when** I check my other logged-in sessions, **then** all have been invalidated.
- [ ] **Given** the sending domain is queried via DNS, **then** valid SPF, DKIM, and DMARC records are present.
- [ ] **Given** a verification email is sent to a mainstream provider (Gmail, Outlook), **when** it arrives, **then** it passes SPF/DKIM alignment and lands in the inbox, not spam.
- [ ] **Given** an email hard-bounces, **when** the webhook fires, **then** the bounce is recorded and observable.
- [ ] **Given** I request account deletion, **when** I confirm with my password, **then** access is revoked immediately and all records including receipt images are purged within 30 days.

### Cross-Cutting

- [ ] Automated accessibility scan (axe-core) reports zero WCAG 2.2 AA violations on the entry form, dashboard, and transaction list.
- [ ] All pages are fully operable at 360px width with no horizontal scrolling.
- [ ] An automated test suite confirms that no API endpoint returns another user's data under any parameter manipulation.
- [ ] CI blocks merge on any high or critical dependency vulnerability.
- [ ] All compute resources provisioned by the IaC templates are Linux-based.

## 8. Out of Scope

| Item | Rationale |
|---|---|
| **Android native app** | Web-first decision. The API and data model built here are the foundation for it. |
| **Offline entry & sync** | Tied to the Android release. Requires a conflict-resolution design (client-generated IDs, last-write-wins vs. merge) that should not be rushed into v1. |
| **Budgets, limits, and overspend alerts** | Separate feature with its own notification infrastructure. |
| **Recurring / scheduled expenses** | Adds scheduling and backfill complexity. |
| **Income tracking and net cash flow** | v1 is expense-only. |
| **Bank / UPI account linking and auto-import** | Requires aggregator partnerships, regulatory review, and a far higher security bar. |
| **Shared, household, or group expenses** | Changes the entire authorisation model from single-tenant-per-user. |
| **Multi-currency and FX conversion** | Schema accommodates it (NFR-40), but no UI, rates, or conversion logic in v1. |
| **Profile-level timezone override** | Device timezone is authoritative; the editable date field covers the edge cases. |
| **Headless-browser PDF rendering** | Security and operational cost not justified for one report layout (NFR-20). |
| **Receipt OCR / auto-extraction of amount and merchant** | Image is stored as an attachment only; no parsing. |
| **Multiple receipts per expense** | One image per expense in v1. |
| **Split transactions across categories** | One category per expense. |
| **Tags and free-text search on notes** | Not selected for v1 filters. |
| **Custom/saved dashboard layouts or saved filter presets** | Fixed four-widget layout; URL bookmarking covers the common case. |
| **Social login (Google / Apple)** | Email + password only in v1. |
| **Two-factor authentication** | Session security hardened (FR-3), but 2FA deferred. |
| **Bulk CSV import of historical expenses** | Export only, not import. |
| **Email digests or scheduled reports** | Transactional email only (verification, reset). |
| **Localisation beyond `en-IN`** | Infrastructure ready, content not translated. |
| **Dark mode** | Cosmetic; deferred. |
| **Admin console, subscriptions, or billing** | Free product in v1. |

## 9. Technical Context

Decisions confirmed during discovery. These are inputs to the architecture spec, not a substitute for it.

| Area | Decision |
|---|---|
| **API** | .NET REST API |
| **Database** | PostgreSQL (Azure Database for PostgreSQL — Flexible Server) |
| **Web client** | Angular |
| **Cloud** | Azure |
| **Object storage** | Azure Blob Storage — chosen over S3 to avoid cross-cloud egress, a second IAM model, and long-lived static access keys. Managed identity + user delegation SAS satisfy FR-22/FR-23. |
| **Email** | Dedicated transactional email service over an owned domain. Azure Communication Services Email is the natural fit; Brevo/Resend/MailerSend are viable free-tier alternatives. Free-tier quotas change — verify at implementation time. |
| **PDF** | Server-side .NET composition library with server-rendered chart images. |
| **Timezone** | Device-level, client-resolved. |

### Indicative Monthly Cost — Azure, Central India

Retail pay-as-you-go, USD, verified from the Azure Retail Prices API on 2026-10-05. Monthly figures assume 730 hours.

**Component rates**

| Service | SKU | Unit price | Per month |
|---|---|---|---|
| App Service (Linux) | B1 | $0.018/hr | $13.14 |
| App Service (Linux) | B2 | $0.036/hr | $26.28 |
| App Service (Linux) | P0v3 | $0.082/hr | $59.86 |
| PostgreSQL Flexible | Burstable B1ms | $0.0245/hr | $17.89 |
| PostgreSQL Flexible | Burstable B2s | $0.098/hr | $71.54 |
| PostgreSQL storage | — | $0.131/GB/mo | — |
| PostgreSQL backup (beyond included) | LRS | $0.095/GB/mo | — |
| Blob Storage | Hot LRS | $0.02/GB/mo | — |
| Static Web Apps | Free | $0 | $0 |

> App Service on **Windows** is materially more expensive for identical capacity (P0v3 Windows $0.162/hr vs Linux $0.082/hr). See NFR-43.

**Scenario A — MVP / early users**

| Item | Cost |
|---|---|
| Angular on Static Web Apps (Free) | $0.00 |
| .NET API on App Service Linux B1 | $13.14 |
| PostgreSQL B1ms + 32 GB storage | $22.08 |
| Blob Hot, 10 GB receipts | $0.20 |
| **Total** | **≈ $35/month** |

**Scenario B — small production load**

| Item | Cost |
|---|---|
| Static Web Apps (Free) | $0.00 |
| App Service Linux B2 | $26.28 |
| PostgreSQL B2s + 64 GB storage | $79.92 |
| Blob Hot, 100 GB receipts | $2.00 |
| **Total** | **≈ $108/month** |

INR equivalents are indicative at roughly ₹88–90/USD; Azure bills INR at its own published rate.

**Cost levers**

- Azure free account provides 12 months of PostgreSQL B1ms (750 hrs/month) and App Service F1. A `Compute - Free` PostgreSQL Flexible Server SKU is also published at $0 in the target region — confirm eligibility. This can reduce Scenario A to near zero for year one.
- Stop the PostgreSQL Flexible Server when not in use during development (supported up to 7 days); only storage is billed while stopped.
- Azure Container Apps with scale-to-zero is cheaper than App Service for genuinely spiky traffic, and more expensive under steady load.
- Receipt storage is the dominant growth variable — see NFR-44.

## 10. Open Questions

1. **Hosting shape** — App Service vs. Container Apps. Drives both the NFR-33 availability design and the cost profile.
2. **Email provider selection** — Azure Communication Services Email vs. a third-party free tier. Confirm current quotas and domain-verification requirements.
3. **PDF library licensing** — QuestPDF's Community licence has a revenue threshold; confirm applicability before committing.
4. **Export artefact lifetime** — how long generated CSV/PDF files remain downloadable before cleanup.
5. **Soft-delete vs. export interaction** — should soft-deleted expenses (within the 30-day window) appear in exports? Current assumption: no.
