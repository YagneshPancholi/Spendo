# Week 3 — Authentication part 1: database and API

## Overview
Goal: replace the anonymous prototype with user-owned data and production-oriented Identity authentication. ASP.NET Core Identity, short-lived JWT access tokens and **rotating, hashed, revocable refresh tokens** are required; login, registration, refresh, logout and password-reset flows get tests. Local prototype data is synthetic and must be purged before adding non-null owner IDs. Do not expose an unauthenticated expense path in any deployed environment.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Add Identity tables, owner FK/index to `Expenses`, refresh-token table with token hash, user ID, expiry, used/revoked timestamps and token-family identifier. Plan migration in `backend/Spendo.Infrastructure/Migrations/` with an explicit purge of disposable prototype rows **only in a disposable development DB**; require a reviewed data migration for any non-disposable environment. Inspect constraints using `sql/verification/auth.sql`.
2. **Day 2 — Identity:** Add Identity registration, unique normalized email handling, password policy and account lockout in `backend/Spendo.Infrastructure/Identity/`. Secrets and signing keys come from environment/user secrets or managed secret store; never log credentials.
3. **Day 3 — Tokens:** Build JWT issuance and validation in `backend/Spendo.Api/Auth/`, pin issuer/audience/signing algorithm and lifetime; use cryptographic RNG for refresh tokens, store only hashes, rotate atomically and revoke token family on reuse. Return generic failure messages to avoid account enumeration.
4. **Day 4 — Endpoints:** Implement `POST /api/v1/auth/register`, `login`, `refresh`, `logout`, `forgot-password` and `reset-password`. Deliver reset links via a pluggable mail sender (local sink for dev, not a real mailbox); single-use, time-limited reset tokens. Set refresh cookie `HttpOnly`, `Secure`, scoped path and explicit SameSite policy; protect cookie-authenticated refresh/logout against CSRF using same-site origin validation or antiforgery token.
5. **Day 5 — Ownership:** Add authenticated user context and enforce owner predicates on every expense query/mutation in `backend/Spendo.Application/Expenses/`; never accept user ID in DTOs. Make unauthorized requests 401 and cross-owner IDs 404.
6. **Day 6 — Backend tests:** Moq token store/mail sender and service boundaries; test bad credentials, reuse, revocation, lockout and cross-user isolation. Use disposable DB integration tests later for database atomicity; mocked tests cannot prove transactions.
7. **Day 7 — Verify:** Run migrations in fresh local DB, test/build; audit API logs for secrets, inspect auth cookies and expiration, merge reviewed PR to `main`. Do not enable public signups without working reset delivery and secure host/cookie policy.

## Commands (PowerShell)
```powershell
git switch -c feature/week-03-auth-api
docker compose -f docker/compose.yaml up -d
dotnet ef migrations add AddIdentityAndOwnership --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api --output-dir Migrations
dotnet ef database update --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api
Get-Content sql/verification/auth.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx
dotnet build backend/Spendo.slnx -c Release
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test
Set-Location ../..
git add backend sql
git commit -m "Require Identity and rotating tokens for expense API"
```
Week 3 UI may still show a 401 until Week 4 login UI; use a local authenticated API client to verify ownership, not production anonymous bypasses.

**Test examples:** xUnit `Assert.Equal(HttpStatusCode.NotFound, otherUserDelete.StatusCode)` and `Assert.False(await tokenStore.IsActiveAsync(reusedTokenHash))`; Jasmine `expect(error.status).toBe(401)` for an unauthenticated expense request; Playwright scenario: protected `/expenses` cannot display data without authentication (Week 4 adds redirect). Use only synthetic accounts such as `tester@example.invalid`.

## What you'll learn
- Identity schema and ownership boundaries, JWT validation, token rotation, reset/lockout and CSRF defenses.
- Why application mocks complement but cannot replace persistence-concurrency tests.

## Definition of done
- [ ] All expense operations require authentication and enforce server-derived ownership.
- [ ] Refresh rotation, logout revocation and reset token single use work; cookie flags and CSRF policy are tested.
- [ ] No real records purged or credentials logged; migrations reviewed; PR merged to `main`.

## Questions or decisions
- Confirm same-site hosting/reverse proxy before cross-origin cookie setup; defer external public deployment until cookie and CSRF design is verified.
- Confirm mail provider and production account-recovery delivery before public registration.
