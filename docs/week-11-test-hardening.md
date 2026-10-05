# Week 11 — Test hardening

## Overview
Goal: make the existing vertical slices regression-resistant with deterministic Playwright E2E, Karma/Jasmine coverage and xUnit/Moq API tests. Mocks exercise application decisions, not PostgreSQL semantics; retain migration/constraint verification separately. No new product behavior or release tag.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Inventory constraints and migrations; add `sql/verification/regression.sql` for owner FK, positive amount and unique category checks against a disposable local database; ensure reruns do not destroy shared data.
2. **Day 2 — API unit tests:** Expand `backend/Spendo.Tests/` cases for amount bounds, date filters, cross-owner mutations, refresh-token reuse, category seeding and CSV injection; mock service dependencies using Moq with synthetic fixtures.
3. **Day 3 — API boundary tests:** Test HTTP 400/401/404/problem-details and authorization behavior through a test host; use an isolated DB only for EF transaction/SQL-specific assertions if one is introduced, never live production data.
4. **Day 4 — Angular unit tests:** Increase Karma coverage for form, guard, interceptor, UTC/date-only helpers, filter round-trip and error/empty UI states; use `HttpTestingController` and Jasmine spies, not a live API.
5. **Day 5 — Playwright suite:** Cover register/login, add/edit/delete, filter/dashboard, CSV and logout in `frontend/spendo/e2e/`; isolate synthetic accounts and data per test, prefer role/label locators, wait for responses rather than sleeps; no committed auth state.
6. **Day 6 — Flake review:** Run tests repeatedly and inspect failures; fix test isolation instead of adding retries/sleeps. Save reports as CI artifacts without sensitive response bodies.
7. **Day 7 — Gate:** Document commands, coverage baseline and known gaps in `docs/testing.md`; run local checks and merge PR.

## Commands (PowerShell)
```powershell
git switch -c feature/week-11-test-hardening
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/regression.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx --collect:"XPlat Code Coverage"
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless --code-coverage
npx playwright test
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql docs
git commit -m "Harden synthetic regression tests across Spendo"
```
**Test examples:** xUnit `Assert.Equal(HttpStatusCode.Unauthorized, anonymousList.StatusCode)`; Jasmine `expect(authInterceptorRequest.headers.has('Authorization')).toBeFalse()` for an external origin; Playwright two synthetic accounts → expense created by A never visible to B in list, dashboard or export.

## What you'll learn
- Test pyramid boundaries, isolation, auth fixture hygiene and reproducible coverage.

## Definition of done
- [ ] Critical flows pass consistently on a clean machine without arbitrary waits.
- [ ] Backend and frontend coverage baselines are documented, not inflated by removing unrelated tests.
- [ ] No secrets/PII in fixtures or artifacts; PR merged into `main`.

## Questions or decisions
- Agree on meaningful branch/line coverage thresholds from the observed baseline before making CI blocking.
