# CuisineEase -- Release Gates

**Last updated:** 2026-06-13

Mandatory checks before any production deployment.

---

## Pre-Deploy Checklist

| # | Gate | Command / Check | Must Pass |
|---|------|-----------------|-----------|
| 1 | Backend build | `cd backend && npm run build && npm run check-types` | Yes |
| 2 | Frontend build | `cd app && npm run build && npm run check-types` | Yes |
| 3 | Frontend lint | `cd app && npm run lint` | Warnings OK, errors fail |
| 4 | No secrets committed | Scan for `.env`, keys, credentials | Clean |
| 5 | Migrations validated | All files ordered, no duplicates, reversible | Yes |
| 6 | Security audit clean | No NEW critical issues; accepted risks in `SECURITY_BACKLOG.md` | Yes |
| 7 | Enterprise audit compliance | Phase 0 (8/8) + Phase 1 (14/14) not regressed | Yes |
| 8 | Environment variables | All required vars set in Vercel + Render | Yes |
| 9 | Git tree clean | No unintended uncommitted changes | Yes |
| 10 | Release report generated | `release-reports/` updated | Yes |

---

## Build Requirements

### Backend

```bash
cd backend
npm run build          # NestJS compilation
npm run check-types    # TypeScript strict check
```

Both must exit 0. Warnings are acceptable; errors block release.

### Frontend

```bash
cd app
npm run build          # Next.js production build
npm run check-types    # TypeScript strict check
npm run lint           # ESLint (warnings OK, errors fail)
```

---

## Migration Rules

1. All schema changes require a TypeORM migration file
2. Migrations use `IF NOT EXISTS` / `IF EXISTS` for idempotency
3. Migration timestamps must be monotonically increasing
4. Never rely on `synchronize` in production
5. Run `migration:show` before `migration:run` on staging
6. Verify migration order against all pending migrations

### Migration Run Procedure

```bash
cd backend
npm run build
npm run migration:show     # Verify pending list
npm run migration:run      # Apply all pending
```

---

## Security Rules

### Must Pass

- Global `FirebaseGuard` active on all protected routes
- No new unguarded mutating endpoints
- Restaurant isolation enforced on all tenant-scoped APIs
- No secrets in tracked files (`.env*` gitignored)
- CORS allowlist configured (`CORS_ORIGINS` env)

### Known Accepted Risks (documented in `ai/SECURITY_BACKLOG.md`)

| ID | Risk | Severity | Mitigation |
|----|------|----------|------------|
| SEC-001 | Tables controller writes unguarded | HIGH | Waitstaff uses secured floor APIs; fix requires ADR-013 amendment |
| SEC-002 | Customer PATCH allows non-cancel transitions | MEDIUM | Low exploit value; dine-in orders are staff-created |
| SEC-003 | assignedWaitstaffId not role-validated | MEDIUM | Low exploit value |
| SEC-007 | ping handler broadcasts to all | LOW | Dev noise only |

### New Security Findings

Any new security finding discovered during development must be:
1. Added to `ai/SECURITY_BACKLOG.md` with severity
2. Assessed for deploy-blocking impact
3. Mitigated or explicitly accepted before release

---

## Test Requirements

### Minimum

- Backend enterprise test suites pass (reservation transitions, order revenue)
- Frontend Playwright auth guard specs pass
- Build succeeds on both repos (counts as smoke test)

### Target (not yet achieved)

- Integration tests: auth login + refresh
- Integration tests: create order from cart
- Integration tests: chat create + send message
- `npm test` in CI pipeline

### Known Test Debt

- Legacy NestJS Jest specs are broken/outdated (not blocking)
- Full E2E suite requires running DB + env
- Authenticated E2E needs `E2E_CHEF_TOKEN`, `E2E_MANAGER_TOKEN`, `E2E_DRIVER_TOKEN` in CI

---

## Deployment Checklist

### Step 1 -- Pre-deploy

- [ ] All gates above pass
- [ ] Pending migrations listed and validated
- [ ] Environment variables verified (`NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`, `CORS_ORIGINS`, `STORAGE_TYPE`)
- [ ] Release report generated

### Step 2 -- Deploy

- [ ] Run migrations on production DB
- [ ] Redeploy backend (Render)
- [ ] Redeploy frontend (Vercel) if env/client changes
- [ ] Verify WebSocket connectivity (same host as API, not port 3002)

### Step 3 -- Post-deploy verification

- [ ] API health check (`GET /`)
- [ ] Auth flow (login, register, Google)
- [ ] Customer: browse, cart, checkout
- [ ] Manager: dashboard loads, orders list
- [ ] Waitstaff: floor view, session create
- [ ] Chef: kitchen queue loads
- [ ] Driver: assignments list
- [ ] WebSocket: notification delivery
- [ ] Chat: message send/receive

---

## Rollback Rules

| Scenario | Action |
|----------|--------|
| Migration fails | Revert with `migration:revert`; backend stays on previous version |
| Backend crash loop | Rollback Render to previous deploy |
| Frontend build error | Vercel auto-rolls to previous deployment |
| Data corruption | Restore from DB backup; halt deploys until investigation |
| Security incident | Disable affected endpoint; patch + redeploy |

---

## Release Scoring Rubric

Based on the 74/100 baseline from `RELEASE_READINESS_REPORT.md`:

| Area | Weight | Criteria |
|------|--------|----------|
| Builds | 20 | Backend + frontend build + typecheck pass |
| Enterprise tests | 15 | Phase 0/1 spec tests pass |
| Security | 15 | No new critical issues; accepted risks documented |
| Documentation | 10 | `ai/` updated; release report generated |
| Git hygiene | 15 | Clean working trees; no unintended changes |
| Migrations | 10 | Ordered, validated, reversible |
| i18n / consistency | 10 | EN/FR keys present; no hardcoded strings in new code |
| E2E / legacy tests | 5 | Integration + Playwright pass |

**Target: 90+/100 for production release.**
