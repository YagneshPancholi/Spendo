# Week 5 — Expense management

## Overview
Goal: edit user-owned expenses, browse owner-owned categories, paginate lists and handle UTC/date-only semantics and validation consistently. Each layer delivers a working edit flow. Amount remains decimal `INR`; `Currency` is persisted per expense, but conversion/settings are explicitly out of scope.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Review `Expenses` UTC timestamp (`timestamptz`), nullable optional notes, category FK, `(OwnerId, ExpenseAtUtc, Id)` index and amount constraint in EF configuration. Add migration only if needed; use `sql/verification/expense-management.sql` to inspect page order and isolation.
2. **Day 2 — API editing:** Add `PUT /api/v1/expenses/{id}` with DTO validation, owner-filtered lookup and owner-validated category ID. Keep created/updated UTC timestamps server generated; return 404 for another user's ID.
3. **Day 3 — API list:** Add bounded `page`/`pageSize` and deterministic sort with tie-breaker; return total count scoped to caller. Add read-only `GET /api/v1/categories/{id}` if necessary; no global category leakage.
4. **Day 4 — Angular form:** Reuse add form for edit in `frontend/spendo/src/app/features/expenses/`; validate required description, decimal two-digit amount and category selection. Preserve server validation messages without trusting HTML.
5. **Day 5 — Time + pagination:** Centralize local input ↔ UTC conversion in `frontend/spendo/src/app/shared/date-time/`; define date-only values separately from instants and test daylight-saving boundary. Add accessible pager, loading state and query-parameter navigation.
6. **Day 6 — Tests:** xUnit cross-owner edits and invalid category; Jasmine pagination/UTC conversion and form; Playwright edit synthetic ₹250 to ₹300, paginate, then verify after reload.
7. **Day 7 — Verify:** Run checks, inspect SQL migration and offset performance, check keyboard navigation, merge PR.

## Commands (PowerShell)
```powershell
git switch -c feature/week-05-expense-management
docker compose -f docker/compose.yaml up -d
# Only if EF model changed:
dotnet ef migrations add ImproveExpenseManagement --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api --output-dir Migrations
dotnet ef database update --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api
Get-Content sql/verification/expense-management.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/expenses.spec.ts
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql
git commit -m "Support owner-scoped expense editing and pagination"
```
If no EF model changed, skip both EF commands and do not create an empty migration. **Test examples:** xUnit `Assert.Equal(HttpStatusCode.NotFound, foreignEdit.StatusCode)`; Jasmine `expect(toUtc(localDate).toISOString()).toBe(expectedInstant)`; Playwright edit → reload → updated amount visible on the correct page.

## What you'll learn
- Composite indexes, stable pagination, date-only vs instant semantics and validation across UI/API/DB.

## Definition of done
- [ ] Update, list and category lookups are owner-scoped; pagination has stable order and bounded size.
- [ ] UTC round-trip and boundary tests pass; amount never calculated with binary floating-point totals.
- [ ] Checks pass and PR merged into `main`.

## Questions or decisions
- Confirm business meaning of “expense date”: local calendar date, exact instant, or both. Keep explicit mapping until decided.
