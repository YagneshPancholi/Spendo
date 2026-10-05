# Week 7 — Composable filters and search

## Overview
Goal: combine week/date/range, amount bounds and text search with Week 6 filters while keeping stable pagination and owner isolation. A single typed filter contract drives SQL, API and Angular URL state. Two hours per day × seven; no release tag this week.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Design parameterized, owner-scoped predicates and indexes for date/amount. Inspect `EXPLAIN (ANALYZE, BUFFERS)` on synthetic local records; decide whether simple `ILIKE` is adequate before adding full-text indexing. Document findings in `sql/verification/search.sql`.
2. **Day 2 — API contract:** Extend `ExpenseFilterRequest` with UTC half-open start/end, min/max decimal, normalized search term and existing fields; set search length and page-size bounds. Reject contradictory range/min>max and invalid dates with 400.
3. **Day 3 — API query:** Compose filters through parameterized EF LINQ; escape literal `%`/`_` when searching for user-entered text, never concatenate SQL. Apply owner predicate before aggregates and paging; use deterministic order.
4. **Day 4 — UI filter model:** Add `ExpenseFilter` and pure `filterToParams`/`paramsToFilter` in `frontend/spendo/src/app/features/expenses/filters/`; preserve shareable URL and clear invalid/stale keys.
5. **Day 5 — UI controls:** Add accessible period/date/amount/search controls; debounce search, cancel superseded HTTP requests, reset page and show active-filter chips. Avoid sending requests on every keystroke.
6. **Day 6 — Tests:** xUnit combinations and injection-like literal input, Jasmine round-trip/query normalization, Playwright combining category/date/search then clearing filters.
7. **Day 7 — Verify:** Run checks and inspect query plan with synthetic volume; ensure dashboard filters use identical semantics; merge PR into `main`.

## Commands (PowerShell)
```powershell
git switch -c feature/week-07-composable-filters
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/search.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/filters.spec.ts
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql
git commit -m "Compose expense date amount and text filters"
```
If profiling shows a new index is needed, add an EF migration and run `dotnet ef database update --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api` before re-running SQL. **Test examples:** xUnit `Assert.Single(await query.ForOwner(ownerA).WithSearch("%").ExecuteAsync())` for a literal `%` fixture; Jasmine `expect(paramsToFilter(filterToParams(filter))).toEqual(filter)`; Playwright search + category + range → one synthetic row, then clear → all owner rows.

## What you'll learn
- Query composition and parameterization, search escaping, index selectivity and cancellable UI search.

## Definition of done
- [ ] Arbitrary supported filter combinations preserve owner scope and page behavior.
- [ ] Search wildcards behave literally and invalid ranges produce clear 400 responses.
- [ ] SQL profiling and tests documented; PR merged into `main`.

## Questions or decisions
- Confirm whether week means ISO Monday–Sunday, and whether search includes notes as well as description.
