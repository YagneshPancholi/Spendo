# Week 12 — CI/CD and deployment research/setup

## Overview
Goal: research Azure feasibility/cost, then set up a safe, repeatable GitHub Actions pipeline and **only** a non-production deployment if hosting/budget decisions are approved. Azure free/trial quotas, container hosting, managed PostgreSQL and networking change: verify official pricing and current SKU limits before selecting. Fallback to Cloudflare Pages + Render + Supabase only after evaluating security, cross-site cookies, backups and operating cost; do not imply free tiers are guaranteed.

## Daily breakdown (~2 hours/day)
1. **Day 1 — Research/decision:** Compare Azure Container Apps or App Service for API, Static Web Apps for Angular and Azure Database for PostgreSQL; document pricing, free quotas, TLS/domain, region, backup, email, cookie same-site topology and ongoing cost in `docs/deployment/options.md`. Obtain budget and environment approval before provisioning.
2. **Day 2 — SQL:** Document EF migration promotion/rollback and backup-before-migrate in `sql/README.md`; run migration on disposable Postgres 17 and verify with `psql`; never run PR branch migrations against production.
3. **Day 3 — API packaging:** Add multi-stage `docker/api.Dockerfile`, health checks and non-root runtime; pass signing key/DB connection via managed environment secret provider, not image or repo.
4. **Day 4 — UI packaging:** Add production Angular build process in `build/` with injected public API origin configuration; no private secrets in compiled JavaScript. If Azure static hosting is chosen, document deployment adapter and SPA fallback; otherwise evaluate static container hosting.
5. **Day 5 — CI:** Add reviewed GitHub Actions workflow at `.github/workflows/spendo-ci.yml`: pinned trusted actions, least-privilege permissions, dependency caching, `dotnet test`, Karma ChromeHeadless, Playwright against isolated ephemeral services, builds and artifact retention policy. No production secrets on untrusted PRs.
6. **Day 6 — CD (approval-gated):** Add environment approvals, OIDC federated credentials rather than long-lived cloud keys, staging health probe and rollback procedure in `build/` and `docs/deployment/`; only implement provider-specific deploy job after choice is documented. Validate cookie/CORS/CSRF across actual UI/API origins.
7. **Day 7 — Verify:** Run CI on PR, confirm artifact hygiene and staging smoke test if provisioned, record actual cost/decision and merge to `main`; do not auto-deploy to production.

## Commands (PowerShell)
```powershell
git switch -c feature/week-12-ci-deployment
docker compose -f docker/compose.yaml up -d
dotnet ef database update --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api
docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev -c "SELECT version();"
docker build -f docker/api.Dockerfile -t spendo-api:local .
dotnet test backend/Spendo.slnx
Set-Location frontend/spendo
npm ci
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test
npx ng build --configuration production
Set-Location ../..
git add .github/workflows/spendo-ci.yml backend frontend sql docker build docs
git commit -m "Document hosting choice and add safe Spendo CI"
```
If Playwright browsers are missing locally, run `npx playwright install chromium` in `frontend/spendo` before its suite. **Test examples:** xUnit `Assert.True(health.IsHealthy)` in staging smoke checks; Jasmine `expect(apiBaseUrl).toMatch(/^https:/)` for production config; Playwright staging scenario log in with an isolated synthetic account, add then delete expense, verify no other user's data.

## What you'll learn
- Hosting cost trade-offs, reproducible packaging, OIDC trust and gated deployment.

## Definition of done
- [ ] Hosting decision or explicit deferral documented with costs and cookie/networking considerations.
- [ ] CI verifies unit/E2E/build with isolated data, short-lived credentials and least privilege.
- [ ] Staging deployed only after approval; rollback/migration ownership documented; PR merged.

## Questions or decisions
- What is the approved monthly cost, Azure subscription/region, public domain, and staging/production boundary?
- Who owns DNS, mail delivery, backups and operational alerts? If unresolved, keep CD as a gated design, not an active deploy.
