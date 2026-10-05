# Week 2 — Local expense prototype

## Overview
Goal: add, list and delete an INR expense end to end, with initial Karma/Jasmine, xUnit/Moq and Playwright coverage. **Prototype is local-development-only and must not be deployed publicly or released before Week 3 ownership enforcement.** Two hours per day, seven days. An expense stores a `Currency` ISO code (`INR` by default) for future multi-currency support; do not perform conversion. Paths are relative to the separate Spendo repository described in Week 1.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Design `Expenses` with UUID ID, decimal amount `numeric(12,2)` and positive check, 3-character currency default `INR`, description length, payment method, category label, date/time as UTC `timestamptz`, and optional notes. Add EF entity/configuration in `backend/Spendo.Infrastructure/Persistence/` and migration; record schema checks in `sql/verification/expenses.sql`. Decide Week 3 migration of prototype rows: delete synthetic-only rows before production; no orphaned records.
2. **Day 2 — API create/list:** Add request/response DTOs to `backend/Spendo.Api/Contracts/Expenses/` and application service to `backend/Spendo.Application/Expenses/`; validate required fields, currency `INR`, decimal precision and pagination bounds. Map endpoints in `backend/Spendo.Api/Endpoints/ExpenseEndpoints.cs` under `/api/v1/expenses`; list in deterministic UTC/ID order.
3. **Day 3 — API delete and tests:** Return 201 with location, 400 for invalid request, 404 for missing delete and 204 for success. Add `backend/Spendo.Tests/Expenses/ExpenseServiceTests.cs` using Moq for the repository boundary; no live data or EF InMemory as SQL-behavior proof.
4. **Day 4 — Angular service:** Add strict models and `ExpenseApiService` in `frontend/spendo/src/app/core/api/`; use environment base URL and `HttpClient`, never direct HTTP from a component. Add request and response unit tests with `HttpTestingController`.
5. **Day 5 — Angular UI:** Implement a reactive add form, accessible list and delete confirmation under `frontend/spendo/src/app/features/expenses/`, with loading/error/empty states. Use Angular Material 21 appropriate to this **separate Spendo app**, verify installed component APIs and imports; no assumed SDS dependency.
6. **Day 6 — E2E:** Add `frontend/spendo/e2e/expenses.spec.ts`: add synthetic ₹250 Food/UPI, see row, cancel deletion, delete, see empty state; run against local ephemeral data and isolate tests. Never use real patient or employee data.
7. **Day 7 — Verify:** Confirm SQL constraints and API error behavior; run frontend and backend tests/build; merge PR to `main`. Do not tag or expose anonymous endpoints; Week 3 replaces prototype access.

## Commands (PowerShell)
```powershell
git switch -c feature/week-02-prototype
docker compose -f docker/compose.yaml up -d
dotnet ef migrations add AddExpenses --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api --output-dir Migrations
dotnet ef database update --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api
Get-Content sql/verification/expenses.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/expenses.spec.ts
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql
git commit -m "Add local expense prototype with tests"
```
Use the PowerShell `Get-Content -Raw` pipeline for SQL verification, not shell input redirection.

**Test examples:** Jasmine `expect(req.request.method).toBe('POST')` and `req.flush(syntheticExpense)`; xUnit `Assert.ThrowsAsync<ValidationException>(() => service.CreateAsync(nonPositiveAmount))`; Playwright add → see `Food` and `INR` → delete → row absent. Test DB constraint separately with `psql` on a disposable development DB.

## What you'll learn
- Vertical SQL → API → UI delivery; decimal money representation, validation at two boundaries and deterministic lists.
- Angular HTTP test controller, Moq service seams and browser-driven acceptance tests.

## Definition of done
- [ ] Add/list/delete works locally and invalid amount is rejected by API and DB.
- [ ] Currency persists as `INR`; no client-provided identity is accepted.
- [ ] Unit and E2E tests pass; synthetic prototype records are marked disposable.
- [ ] PR merged to `main`; no publicly reachable anonymous expense API.

## Questions or decisions
- Confirm initial payment methods and temporary category labels; Week 4 replaces labels with user-owned category IDs.
