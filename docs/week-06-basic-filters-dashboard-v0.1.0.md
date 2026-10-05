# Week 6 — Basic filters and dashboard (`v0.1.0`)

## Overview
Goal: filter by month/year, category and payment method, and show monthly/yearly, category and payment totals for the authenticated owner. Release `v0.1.0` only after end-to-end checks. Totals come from database decimal aggregation, not client floating-point math; currency remains INR only.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Examine owner/date/category/payment indexes with `EXPLAIN` in `sql/verification/dashboard.sql`; use half-open UTC ranges derived from selected calendar zone, not string formatting on indexed columns.
2. **Day 2 — API filters:** Add bounded query DTO to list endpoint; combine selected month/year, category and payment method with owner predicate, reject invalid values, retain stable pagination.
3. **Day 3 — API totals:** Add `GET /api/v1/dashboard/summary` grouped by category and payment, with month/year totals and explicit zero/empty behavior. Aggregate `numeric` in PostgreSQL; include `Currency: INR` in response.
4. **Day 4 — UI filters:** Add typed filter state in URL query parameters under `frontend/spendo/src/app/features/expenses/filters/`; preserve state on reload and reset pager when filter changes.
5. **Day 5 — Dashboard:** Add responsive, accessible totals/cards under `features/dashboard/`; include loading, empty and error states, and specify reporting time zone.
6. **Day 6 — Tests:** xUnit owner isolation and decimal totals; Jasmine query serialization and empty totals; Playwright select category/month → filtered list and dashboard total update.
7. **Day 7 — Release:** Run CI-equivalent tests/build, document release in `docs/releases/v0.1.0.md`, merge PR, then tag **merged `main`** and publish release notes. Do not tag a feature branch.

## Commands (PowerShell)
```powershell
git switch -c feature/week-06-basic-filters
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/dashboard.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/dashboard.spec.ts
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql docs
git commit -m "Add owner-scoped basic filters and dashboard totals"
# After PR approval and merge, in an up-to-date local main checkout:
git switch main
git pull --ff-only
git tag -a v0.1.0 -m "Spendo MVP: authenticated expenses, filters and totals"
# Publish tag via approved Git hosting workflow; do not tag until merged checks pass.
```
**Test examples:** xUnit `Assert.Equal(300.00m, summary.MonthTotal)` for synthetic entries and verify another owner's values excluded; Jasmine `expect(filterToParams(filter).get('categoryId')).toBe(categoryId)`; Playwright change month → URL retains selection after reload → total matches synthetic fixtures.

## What you'll learn
- SQL aggregates, URL-driven state, timezone-aware range filtering and release tagging.

## Definition of done
- [ ] Totals match filtered owner-owned rows in INR, including empty results.
- [ ] Unit/E2E/production build pass; release notes and approved PR are merged.
- [ ] `v0.1.0` points to checked, merged `main`.

## Questions or decisions
- Confirm reporting timezone (user timezone vs browser timezone) and treatment of future cross-currency totals before implementing currency settings.
