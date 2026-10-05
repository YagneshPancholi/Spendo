# Week 1 — Foundations (reference plan)

## Overview
Goal: establish the **separate Spendo monorepo**, not the SHM library in which these plans are stored. This reference reconstructs Week 1 from the supplied outline; the original drafted text was not present in this repository. By Day 7, a clean Windows machine can start PostgreSQL 17, run an ASP.NET Core 10 health endpoint and an Angular 21 shell, execute starter tests, and merge a documented PR to `main`. Budget: 2 hours × 7 days. Use synthetic data only; never commit passwords, tokens, database dumps, or personal data.

The intended Spendo layout is `backend/`, `frontend/`, `sql/`, `docker/`, `build/`, `docs/`. Later plans assume `backend/Spendo.slnx`, `backend/Spendo.Api/`, `backend/Spendo.Application/`, `backend/Spendo.Infrastructure/`, `backend/Spendo.Tests/`, `frontend/spendo/` and `docker/compose.yaml`. EF Core migrations in `backend/Spendo.Infrastructure/Migrations/` become the schema source of truth; `sql/` holds reviewed SQL examples/verification, not a second competing migration system. Commands below run from the **Spendo repository root in PowerShell** unless a `Set-Location` is shown. Replace placeholders and check installed CLI versions before executing; do not copy these plans into the SHM workspace as application code.

## Daily breakdown (~2 hours/day)
1. **Day 1 — Repository:** Create the Spendo repository, `backend`, `frontend`, `sql`, `docker`, `build`, `docs`; add `.gitignore` (including `.env`, Playwright auth state and `bin/obj/node_modules`), `README.md`, and `.github/pull_request_template.md` with purpose, author, verification and rollback. Protect `main`; work on `feature/week-01-foundations`.
2. **Day 2 — Database first:** Add `docker/compose.yaml` for a local PostgreSQL 17 container with a named volume, health check and environment supplied from untracked `.env`; document a synthetic development database. Add `sql/README.md` describing EF migration ownership and a `psql` verification query.
3. **Day 3 — Backend:** Scaffold solution, API, application, infrastructure and xUnit test projects; connect project references; register health checks and a scoped service boundary. Keep credentials in user secrets or environment variables, not tracked files.
4. **Day 4 — Persistence:** Add EF Core/Npgsql packages after checking versions and advisories, `SpendoDbContext` and a first empty/baseline migration; run migration only on local development DB. Validate `SELECT current_database()` with `psql`.
5. **Day 5 — UI:** Scaffold Angular 21 standalone app with routing, SCSS and strict TypeScript under `frontend/spendo`; select the **Karma/Jasmine** test runner at generation time (or configure the supported Angular 21 Karma builder explicitly); create a health/status page and typed API base URL configuration.
6. **Day 6 — Tests and tooling:** Add one xUnit health/application test and one Jasmine shell test; install/configure Playwright in `frontend/spendo` for a smoke navigation test; document how to run the API and UI in VS Code/Visual Studio.
7. **Day 7 — Integrate:** Run local lint (if configured), tests and production builds; verify Docker restart preserves the volume. Review security and instructions, open a descriptive PR, merge into `main` after checks; no release tag yet.

## Commands (PowerShell)
```powershell
git switch -c feature/week-01-foundations
docker compose -f docker/compose.yaml up -d
docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev -c "SELECT current_database();"
dotnet new sln -n Spendo -o backend
dotnet new webapi -n Spendo.Api -o backend/Spendo.Api
dotnet new classlib -n Spendo.Application -o backend/Spendo.Application
dotnet new classlib -n Spendo.Infrastructure -o backend/Spendo.Infrastructure
dotnet new xunit -n Spendo.Tests -o backend/Spendo.Tests
dotnet sln backend/Spendo.slnx add backend/Spendo.Api backend/Spendo.Application backend/Spendo.Infrastructure backend/Spendo.Tests
dotnet add backend/Spendo.Api reference backend/Spendo.Application backend/Spendo.Infrastructure
dotnet add backend/Spendo.Infrastructure reference backend/Spendo.Application
dotnet add backend/Spendo.Tests reference backend/Spendo.Application
Set-Location frontend
ng new spendo --routing --style=scss --strict --test-runner=karma
Set-Location ..
dotnet test backend/Spendo.slnx
dotnet build backend/Spendo.slnx
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql docker build docs README.md .github .gitignore
git commit -m "Initialize Spendo development foundation"
```
Check `ng new --help` for the installed CLI's test-runner option. Install .NET 10 SDK, Angular 21 CLI, Node supported by Angular 21 and Docker Desktop before these steps; use `dotnet user-secrets` for local API secrets. `db`, `spendo`, and `spendo_dev` must match `docker/compose.yaml`.

**Starter test examples:** Jasmine `expect(component).toBeTruthy()` in `frontend/spendo/src/app/app.spec.ts`; xUnit `Assert.True(result.IsHealthy)` for the application health check in `backend/Spendo.Tests/HealthTests.cs`; Playwright `page.goto('/')` then assert the app heading is visible in `frontend/spendo/e2e/smoke.spec.ts`. Do not use actual user records.

## What you'll learn
- One-repo project boundaries, reproducible local PostgreSQL and EF migration ownership.
- Angular CLI, .NET CLI, PowerShell navigation, unit tests and PR review flow.

## Definition of done
- [ ] A fresh developer can start DB, API and UI from documented commands.
- [ ] Tests and builds pass; `.env`, credentials and test auth state are untracked.
- [ ] Baseline schema can be rebuilt from migrations; PR reviewed and merged into `main`.

## Questions or decisions for later
- Pin exact compatible patch versions, Node version and EF/Npgsql version when scaffolding; do not assume latest patches are compatible.
- Confirm which Angular Material 21 components meet accessibility needs in the independent Spendo app; SHM's SDS-only rules apply to SHM, not Spendo.
