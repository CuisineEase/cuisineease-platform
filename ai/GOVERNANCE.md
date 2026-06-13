# CuisineEase -- Governance Rules

**Effective:** 2026-06-13
**Authority:** Platform architecture + product owner

---

## 1. Source of Truth

**`ai/` is the canonical source of truth for system behavior.**

When documentation and code disagree:
1. Identify the conflict (cite file paths)
2. Determine which is correct (prefer runtime behavior + recent commits)
3. Propose update to `ai/*.md` and/or legacy doc
4. Do not silently implement against `ai/` without noting the conflict

**Superseded documents:** `backend/README.md`, `app/README.md`, `backend/mindmap.md`, parts of `.env.example`. These are stale and must not be treated as authoritative.

---

## 2. Feature Development Rules

### No feature work without updating `ai/TASKS.md`

- Every new task gets a tracked ID (e.g., `SEC-008`, `PAY-005`, `N-17`)
- Status updates happen in the **same PR** as the code change
- Completed tasks move to done; follow-ups get new entries

### Major implementation threshold

Requires spec update in `ai/` **before** coding begins:
- New domain module or database tables
- New user role or permission model change
- Payment flow changes
- Breaking API contract changes
- Cross-repo coordinated releases

### When confidence is low (<80%)

- Ask clarifying questions (see `PROJECT.md` open questions)
- Do not guess product rules (especially chat, payments, roles)
- Document assumptions in `DECISIONS.md` as Proposed until confirmed

---

## 3. Deployment Rules

### No deployment without passing release verification

See [`../docs/RELEASE_GATES.md`](../docs/RELEASE_GATES.md) for the full checklist.

Minimum requirements:
- Backend builds (`npm run build`, `npm run check-types`)
- Frontend builds (`npm run build`, `npm run check-types`)
- All pending migrations validated and ordered
- No new critical security issues (accepted risks documented in `SECURITY_BACKLOG.md`)
- No secrets committed
- Enterprise audit compliance verified

### Deploy sequence

1. Run migrations on production DB
2. Redeploy backend (Render)
3. Redeploy frontend (Vercel) if env/client changes
4. Verify environment variables

---

## 4. Security-First Enforcement

- Global `FirebaseGuard` is deny-by-default -- new endpoints are protected automatically
- `@Public()` only when intentional (browse/auth endpoints)
- Restaurant staff access via `RestaurantAccessService`, not `SystemRole` alone
- Schema changes require TypeORM migrations -- never rely on `synchronize` in production
- Security issues tracked in `ai/SECURITY_BACKLOG.md` with severity levels
- Chat/notifications emit via `NotificationsGateway.emitToUser`

---

## 5. Documentation Must Ship With Code

Update in the **same session/PR** as code changes:

| Artifact | When |
|----------|------|
| `ai/TASKS.md` | Mark tasks done; add new follow-ups |
| `ai/DECISIONS.md` | New architectural/product decisions |
| `ai/ROADMAP.md` | Phase status changes |
| `ai/ARCHITECTURE.md` | New modules, endpoints, env vars |
| `ai/PROJECT.md` | Scope or role model changes |
| `ai/SECURITY_BACKLOG.md` | New security findings or resolutions |

---

## 6. Decision Records

Architecture decisions use lightweight ADRs in `ai/DECISIONS.md`:
- Append at top; superseded decisions stay for history
- Include: context, decision, consequences, affected files
- Proposed decisions marked clearly until accepted

---

## 7. Repository Coordination

| Change type | Repository | Push target |
|-------------|-----------|-------------|
| Frontend | `app/` | `origin/main` + `personal/main` |
| Backend | `backend/` | `origin/main` |
| Specs/docs | `cuisineease-platform/` | `origin/main` |

- Never mix frontend and backend in one commit
- API contract changes require coordinated deploys (backend first)
- `ai/` syncs between workspace and platform repo

---

## 8. Anti-Patterns (Do Not)

- Implement large features without updating `ai/TASKS.md`
- Use `SystemRole.USER` alone for restaurant manager logic
- Expose WebSocket on a separate port
- Commit secrets or `.env` files
- Force-push `main` without explicit request
- Over-engineer abstractions for one-off helpers
- Trust `backend/mindmap.md` next steps as current
- Add tests unless requested or materially valuable
- Copy application source code into the platform repository

---

## 9. Enterprise Audit Alignment

All changes must be validated against enterprise audit expectations:
- Phase 0 (Critical) fixes must not regress
- Phase 1 (High) fixes must not regress
- Normal/Low backlog items tracked in `TASKS.md`
- Cross-portal consistency matrix reviewed quarterly
- Workflow verification re-run after major feature ships
