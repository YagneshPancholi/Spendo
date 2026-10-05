# Week 10 — PWA, accessibility and CSV (`v0.3.0`)

## Overview
Goal: installable Angular shell, accessible keyboard/screen-reader expense flows and **authenticated, filtered** CSV export. Never cache authenticated API responses, tokens or expense data in a service worker; offline expense editing is out of scope. Release `v0.3.0` after verifying CSV formula-injection prevention and data isolation.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Confirm export query uses owner + shared filters and a deterministic order; profile a bounded synthetic export using `sql/verification/export.sql`. Do not add a separate unbounded search path.
2. **Day 2 — API export:** Implement `GET /api/v1/expenses/export` in `backend/Spendo.Api/Endpoints/ExpenseEndpoints.cs` with owner scoping, export row/time bounds, UTF-8 CSV, fixed columns (date, amount, currency, category, payment, description) and neutralization of spreadsheet formula prefixes (`=`, `+`, `-`, `@`, tab, CR/LF) in text cells. Escape quotes/commas/newlines; never export notes by accident. Consider safe amount formatting (numeric values rather than user-controlled strings).
3. **Day 3 — API tests:** xUnit verifies cross-owner exclusion, filter agreement, formula-like synthetic description (`=1+1`) treated as text, and quoted/comma content. Define file name without private identifiers.
4. **Day 4 — PWA:** In `frontend/spendo`, use `ng add @angular/pwa` after checking Angular 21 version; inspect `ngsw-config.json` and restrict caching to public app-shell assets. Do not enable offline API/data cache or store auth state.
5. **Day 5 — Accessibility:** Audit keyboard/focus order, labels, contrast, modal focus return and table alternatives in `features/expenses/` and `features/dashboard/`; resolve confirmed issues, test at narrow viewport.
6. **Day 6 — UI export/tests:** Add a filtered export action with download error state; Jasmine request parameters/filename handling and Playwright authenticated export, keyboard navigation and synthetic CSV parse check.
7. **Day 7 — Release:** Test/build, manual offline shell vs API behavior, update `docs/releases/v0.3.0.md`; merge reviewed PR and tag merged `main`.

## Commands (PowerShell)
```powershell
git switch -c feature/week-10-pwa-csv
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/export.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
Set-Location frontend/spendo
ng add @angular/pwa
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test e2e/export.spec.ts
npx ng build --configuration production
Set-Location ../..
dotnet test backend/Spendo.slnx
git add backend frontend sql docs
git commit -m "Add safe CSV export accessible UI and offline shell"
# After approval and merge:
git switch main
git pull --ff-only
git tag -a v0.3.0 -m "Spendo PWA shell accessibility and CSV export"
```
**Test examples:** xUnit `Assert.DoesNotContain("=1+1", unescapedCsvCell)` for formula-like synthetic data (assert parsed cell is prefixed with a protective apostrophe); Jasmine `expect(req.request.params.get('categoryId')).toBe(categoryId)`; Playwright click export → inspect downloaded CSV headers and synthetic rows, assert no other owner's records.

## What you'll learn
- PWA cache boundaries, CSV injection prevention and accessible data visualizations.

## Definition of done
- [ ] No private API data or tokens cached by PWA; offline shell is clearly read-only.
- [ ] Filtered CSV is bounded, owner-scoped and safe to open in common spreadsheets.
- [ ] Keyboard/screen-reader checks and tests pass; `v0.3.0` tagged on merged `main`.

## Questions or decisions
- Choose export row limit and whether users need asynchronous large exports in a future release.
