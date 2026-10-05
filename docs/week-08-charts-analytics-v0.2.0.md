# Week 8 — Charts and analytics (`v0.2.0`)

## Overview
Goal: deliver owner-scoped spending trends and category/payment breakdowns for the same selected filters as the expense list. Release `v0.2.0` after accessible chart alternatives and consistency tests. INR-only decimal totals remain server-calculated; charts are not a second accounting engine.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Profile grouped owner/time/category/payment queries with synthetic rows; determine calendar bucket boundaries and zero-period behavior in `sql/verification/analytics.sql`. Add EF index migration only if profiling justifies it.
2. **Day 2 — API:** Add `GET /api/v1/dashboard/analytics` in `backend/Spendo.Api/Endpoints/DashboardEndpoints.cs`; accept bounded shared filter contract, return typed labels, decimal totals and `INR`; cap bucket count and date range.
3. **Day 3 — API tests:** Compare category + payment totals to filtered summary, verify owner isolation and UTC/reporting timezone bucket edges in `backend/Spendo.Tests/Dashboard/AnalyticsTests.cs`.
4. **Day 4 — Angular chart:** Verify Angular 21 compatibility, license and advisory status before selecting a maintained chart library. Add trend and category charts under `frontend/spendo/src/app/features/dashboard/`; add accessible data table alongside every chart.
5. **Day 5 — UX:** Reuse URL filters, provide touch-friendly responsive layout, clear empty/zero states and aria headings; avoid exposing raw transaction details in chart tooltips.
6. **Day 6 — Tests:** Karma test typed series mapping and empty state; Playwright change date range and confirm chart/table agree with API and remain keyboard accessible.
7. **Day 7 — Release:** Run tests/build/a11y review, write `docs/releases/v0.2.0.md`, merge PR, and tag checked `main` with `v0.2.0`.

## Commands (PowerShell)
```powershell
git switch -c feature/week-08-analytics
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/analytics.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
# After license/advisory review, install a maintained chart package in frontend/spendo:
Set-Location frontend/spendo
npm install chart.js
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/dashboard.spec.ts
npx ng build --configuration production
Set-Location ../..
dotnet test backend/Spendo.slnx
git add backend frontend sql docs
git commit -m "Add accessible owner-scoped spending analytics"
# After approval and merge to main:
git switch main
git pull --ff-only
git tag -a v0.2.0 -m "Spendo analytics and composable filtering"
```
Pin and review the resolved package version in the lockfile before merging. **Test examples:** xUnit `Assert.Equal(summary.MonthTotal, analytics.CategoryTotals.Sum(x => x.Total))`; Jasmine `expect(series.length).toBe(0)` for empty API data; Playwright select month → chart's adjacent accessible table shows synthetic INR totals.

## What you'll learn
- Server-side aggregation, chart accessibility, dependency due diligence and release validation.

## Definition of done
- [ ] Analytics agree with filtered dashboard/list and never mix owners or currencies.
- [ ] Tables make chart information available to keyboard/screen-reader users.
- [ ] Tests/build pass, release notes merged, `v0.2.0` tagged on `main`.

## Questions or decisions
- Pick chart library only after comparing supported Angular version, licensing, size and accessibility.
