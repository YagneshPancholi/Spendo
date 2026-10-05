# Week 14 — Documentation, polish, domain and `v1.0.0`

## Overview
Goal: a documented, accessible, secure v1 release of Spendo, subject to the hosting and operational approvals in Weeks 12–13. Configure a custom domain only after DNS/TLS ownership is resolved. Release `v1.0.0` from reviewed, checked `main`; if blockers remain, defer tag rather than call staging “production.”

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Verify production migration plan and backup/restore evidence against PostgreSQL 17; save `sql/verification/release.sql` with non-destructive checks for schema version/index presence. Require backup and change approval before live migration.
2. **Day 2 — API polish:** Verify documented endpoints, JWT/reset lifecycle, health checks, error codes and export contract; update `docs/api.md` and `docs/architecture.md` with owner boundaries and currency semantics.
3. **Day 3 — UI polish:** Audit small-screen layouts, keyboard navigation, chart alternatives, loading/empty/error states and timezone labels; update `frontend/spendo/README.md` with Angular 21/Karma/Playwright commands.
4. **Day 4 — Domain/hosting:** If approved, configure DNS, TLS, HTTPS redirects, UI/API origin policy and cookie/CSRF settings in provider-owned environment; run an external smoke test. Document rollback DNS TTL and host config without secrets. If not approved, document the blocked release condition.
5. **Day 5 — Docs:** Finalize root `README.md`, `docs/deployment/`, `docs/operations/`, `docs/testing.md`, `docs/releases/v1.0.0.md` with purpose, owner, rationale, status, setup, restore, rollback and known limitations. Explicitly defer currency conversion/INR setting behavior pending a user decision.
6. **Day 6 — Release candidate:** Run all tests/builds and manual synthetic acceptance: register, reset password, create/edit/filter/delete, chart and CSV; rehearse rollback on staging. Confirm no private data in artifacts/logs and monitor health.
7. **Day 7 — Release:** Obtain signoff, merge PR, verify deployment and migrations, tag merged `main`, publish release notes and monitor alerts; if signoff fails, fix or record blocker without tagging.

## Commands (PowerShell)
```powershell
git switch -c feature/week-14-release
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/release.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx -c Release
dotnet build backend/Spendo.slnx -c Release
Set-Location frontend/spendo
npm ci
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test
npx ng build --configuration production
Set-Location ../..
git add README.md backend frontend sql docker build docs
git commit -m "Document and verify Spendo v1 release"
# After approval, merge, deployment smoke checks and signoff:
git switch main
git pull --ff-only
git tag -a v1.0.0 -m "Spendo v1: authenticated INR expense tracking"
```
**Test examples:** xUnit `Assert.Equal(204, (int)deleteResponse.StatusCode)` in synthetic release smoke; Jasmine `expect(renderedCurrency).toContain('INR')`; Playwright register → add INR expense → filter → export → logout → cannot reopen account data.

## What you'll learn
- Production release gates, operational handoff, domain/TLS validation and traceable change control.

## Definition of done
- [ ] Docs describe how to operate, restore, test and roll back deployed components.
- [ ] Security, accessibility and isolated E2E checks pass with synthetic data; release approved.
- [ ] `v1.0.0` points to merged `main` after successful deployment smoke tests, or is explicitly deferred.

## Questions or decisions
- Future currency setting: when switching INR to GBP/USD/EUR, ask whether existing entries retain their original currency or convert at a stated historical rate; **do not** silently relabel amounts. Specify exchange-rate source, effective timestamp, rounding and audit trail in a separate future feature plan.
