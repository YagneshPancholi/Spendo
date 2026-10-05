# Week 9 — Error handling, logging, DTO hardening and API versioning

## Overview
Goal: predictable API errors, privacy-safe diagnostics and explicit v1 contract behavior across SQL/API/UI. Harden input and concurrency boundaries without changing legitimate expense behavior; retain `/api/v1/` compatibility.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Review constraints, cascade behavior, unique category names and concurrency strategy; document in `sql/verification/integrity.sql`. Add an EF migration only for missing invariants; test on disposable data.
2. **Day 2 — API errors:** Centralize exception handling in `backend/Spendo.Api/Errors/`; use RFC 9457 problem details with stable codes and correlation IDs, generic 500 response, appropriate 400/401/403/404/409/429. Avoid stack traces and account existence disclosure.
3. **Day 3 — Logging:** Add structured log scopes and request IDs in `backend/Spendo.Api/`; redact authorization, cookies, reset links, free-text expense descriptions and user identifiers. Document retention and access policy in `docs/operations/logging.md`.
4. **Day 4 — DTO/version:** Keep request/response types separate from EF entities, reject unknown identity/owner fields where appropriate, bound lengths/page sizes. Configure explicit API version handling for v1 with consistent deprecation policy; verify available .NET 10-compatible versioning library before adding it, or retain explicit versioned routes without another dependency.
5. **Day 5 — UI:** Add problem-details mapper and global error feedback in `frontend/spendo/src/app/core/`; retry only safe idempotent reads, show actionable validation and a generic incident ID for unexpected errors.
6. **Day 6 — Tests:** xUnit test problem details shape/secret redaction, Jasmine error mapping and no repeated unsafe POST, Playwright simulate API 500 and verify friendly retry state with no sensitive body.
7. **Day 7 — Verify:** Inspect synthetic logs, run tests/build and PR review; merge to `main`.

## Commands (PowerShell)
```powershell
git switch -c feature/week-09-error-contracts
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/integrity.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx
dotnet build backend/Spendo.slnx -c Release
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/errors.spec.ts
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql docs
git commit -m "Standardize v1 API errors and privacy-safe logs"
```
**Test examples:** xUnit `Assert.Equal("validation_error", problem.Code)` and assert logs do not contain a synthetic bearer string; Jasmine `expect(toDisplayError(problem).code).toBe('validation_error')`; Playwright mock a 500 and check the UI displays a generic incident message, not a stack trace.

## What you'll learn
- Problem details, correlation IDs, DTO boundaries and audit-friendly observability without leaking private content.

## Definition of done
- [ ] Error status/code conventions documented, with v1 behavior backward compatible.
- [ ] No credentials or expense free text in diagnostic logs; synthetic failure tests pass.
- [ ] SQL checks, frontend/backend builds and PR pass; merged into `main`.

## Questions or decisions
- Confirm retention and access control for operational logs before production deployment.
