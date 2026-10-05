# Week 4 — Authentication part 2: UI and default categories

## Overview
Goal: registration/login/refresh/logout and guarded expenses work end to end. Signup creates **a default set of user-owned categories exactly once** within the account-creation transaction; category names remain a decision and examples are synthetic placeholders. Access token stays in memory; the rotated refresh token is an HttpOnly cookie, never exposed to JavaScript. Budget: 14 hours.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Add `Categories` with owner FK, normalized per-user unique name and soft-delete policy decision; add `CategoryId` FK to `Expenses`. Backfill only synthetic local prototype data or reset disposable DB; add category ownership index and verify in `sql/verification/categories.sql`.
2. **Day 2 — API:** Add transactional registration/category seeding to `backend/Spendo.Application/Auth/`; idempotency prevents duplicates on retry. `GET /api/v1/categories` returns only caller-owned categories; expense create checks category owner server-side.
3. **Day 3 — Angular auth:** Build strict login/register forms in `frontend/spendo/src/app/features/auth/`, with field and server validation, accessible error messages and generic reset response. Add `AuthApiService` in `core/api/` and memory-only session state in `core/auth/`.
4. **Day 4 — Interceptors:** Add auth header and coordinated refresh-on-401 interceptor in `core/interceptors/`; allowlist API origin, avoid sending bearer to arbitrary hosts and avoid refresh recursion. With cookie credentials, follow explicit backend origin/CSRF configuration; clear memory and redirect on failed refresh.
5. **Day 5 — Guards:** Guard `expenses` and `dashboard` routes; restore session with a single refresh request on reload; implement logout. Replace prototype static category options with owner-scoped API response; show empty/default loading state.
6. **Day 6 — Tests:** Cover signup transaction and category ownership in xUnit; Karma tests for guard, interceptor concurrency and forms; Playwright synthetic register → login → create expense → logout → blocked navigation.
7. **Day 7 — Verify:** Run DB migration and tests; manually check Chrome cookie settings, CSRF rejection, no bearer in storage, no stale user data after logout; merge PR.

## Commands (PowerShell)
```powershell
git switch -c feature/week-04-auth-ui
docker compose -f docker/compose.yaml up -d
dotnet ef migrations add AddUserCategories --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api --output-dir Migrations
dotnet ef database update --project backend/Spendo.Infrastructure --startup-project backend/Spendo.Api
Get-Content sql/verification/categories.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
dotnet test backend/Spendo.slnx
Set-Location frontend/spendo
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/auth.spec.ts
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql
git commit -m "Add authenticated UI and signup categories"
```
**Test examples:** xUnit `Assert.Equal(expectedDefaultCount, categories.Count(c => c.OwnerId == newUser.Id))` after retry; Jasmine `expect(req.request.headers.has('Authorization')).toBeTrue()` only for the configured API origin; Playwright logout → navigate to `/expenses` → login shown, with no cached expense rows.

## What you'll learn
- Transactional provisioning, cookie/session coordination and secure interceptors.
- Route guards as UX only: server-side owner checks remain authoritative.

## Definition of done
- [ ] Signup seeds exactly one category set, with no category ownership bypass.
- [ ] Reload refreshes session safely; failed refresh/logouts clear user data.
- [ ] Unit/E2E tests pass; PR merged into `main`.

## Questions or decisions
- Agree on default category names and whether users can rename/delete defaults before implementing those mutations.
