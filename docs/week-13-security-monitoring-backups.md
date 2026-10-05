# Week 13 — Security hardening, monitoring and backups

## Overview
Goal: verify auth boundaries, availability, restore capability and privacy-safe monitoring ahead of v1. Security and backup controls must be **tested**, not merely configured. Never log or export real personal data; even though Spendo is an expense tracker, apply least-privilege and HIPAA-aware handling if integrated with clinical environments.

## Daily breakdown (~2 hours/day)
1. **Day 1 — SQL:** Review least-privilege DB roles, TLS, schema grants, indexes and backup retention in `sql/verification/security.sql`; create a **synthetic disposable** database for restore rehearsal. Encrypt backups at rest and restrict access.
2. **Day 2 — API auth:** Check JWT issuer/audience/expiry, refresh reuse/revocation, password reset, rate limits and per-owner predicates for all list, dashboard and export endpoints. Test CSRF defenses for cookie-authenticated refresh/logout and configure precise CORS origins.
3. **Day 3 — Web security:** Configure HTTPS-only, secure headers and CSP for the actual host in `build/` or provider config; avoid `unsafe-inline` without documented exception. Review dependency advisories and lockfile, no raw user HTML in UI.
4. **Day 4 — Monitoring:** Add privacy-safe health/readiness probes, structured metrics and alert thresholds in `backend/Spendo.Api/` and `docs/operations/`; never include tokens, account email or expense text in telemetry. Distinguish auth failure metrics from user identity.
5. **Day 5 — Backups:** Document scheduled encrypted `pg_dump`/provider backups, retention and restore RPO/RTO in `docs/operations/backups.md`; perform local isolated restore and verify row counts/checksums without overwriting live DB.
6. **Day 6 — Tests:** xUnit security boundary/rate limit, Jasmine no session persistence or external bearer, Playwright cross-user access/CSRF and expired-session behavior on synthetic accounts.
7. **Day 7 — Review:** Run scans already in CI and tests/build; record threat model, unresolved risks and restore evidence; merge PR only when critical issues have owners and resolution.

## Commands (PowerShell)
```powershell
git switch -c feature/week-13-security-operations
docker compose -f docker/compose.yaml up -d
Get-Content sql/verification/security.sql -Raw | docker compose -f docker/compose.yaml exec -T db psql -U spendo -d spendo_dev
# Local synthetic-only rehearsal; temporary dump stays inside the local DB container:
docker compose -f docker/compose.yaml exec -T db pg_dump -U spendo -Fc -f /tmp/spendo-synthetic.dump spendo_dev
# Restore only into a disposable spendo_restore database, never a live database:
docker compose -f docker/compose.yaml exec -T db createdb -U spendo spendo_restore
docker compose -f docker/compose.yaml exec -T db pg_restore -U spendo -d spendo_restore --no-owner /tmp/spendo-synthetic.dump
docker compose -f docker/compose.yaml exec -T db rm /tmp/spendo-synthetic.dump
dotnet test backend/Spendo.slnx
Set-Location frontend/spendo
npm audit --audit-level=high
npm run test -- --watch=false --browsers=ChromeHeadless
npx playwright test
npx ng build --configuration production
Set-Location ../..
git add backend frontend sql build docs
git commit -m "Verify Spendo security monitoring and restore procedures"
```
For binary backup restore on Windows PowerShell, avoid text pipelines that can corrupt bytes. On a real environment use an encrypted, access-controlled provider backup rather than this temporary local container dump. **Test examples:** xUnit verify another owner's rows are absent from CSV export (404 applies only to ID-based foreign record lookups); Jasmine `expect(sessionStorage.getItem('accessToken')).toBeNull()`; Playwright logout → refresh cookie no longer authenticates.

## What you'll learn
- Threat modeling, least privilege, privacy-safe telemetry and recovery validation.

## Definition of done
- [ ] Ownership, CSRF, CORS, headers and rate limits verified; no critical unmitigated findings.
- [ ] Encrypted backup policy and **tested** isolated restore documented; no dump tracked in Git.
- [ ] Alerts have an owner, checks pass and PR merged into `main`.

## Questions or decisions
- Approve incident response owner, retention, RPO/RTO and on-call coverage before launch.
