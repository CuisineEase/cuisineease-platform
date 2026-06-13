# CuisineEase -- Repository Catalog

**Last updated:** 2026-06-13

---

## Repository Map

| # | Repository | URL | Branch | Role |
|---|-----------|-----|--------|------|
| 1 | **Frontend** | `git@github.com:CuisineEase/frontend-web.git` | `main` | Next.js 14 web application -- customer, manager, waitstaff, chef, driver, admin portals |
| 2 | **Backend** | `git@github.com:CuisineEase/backend.git` | `main` | NestJS 11 REST API + Socket.io -- domain modules, migrations, realtime |
| 3 | **Platform** (this) | `git@github.com:CuisineEase/cuisineease-platform.git` | `main` | System intelligence, architecture, governance, release control |

### Frontend remotes

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `git@github.com:CuisineEase/frontend-web.git` | Primary |
| `personal` | `git@github.com:TamahJustene/CuisineEase-frontend.git` | Developer fork |

---

## Ownership Boundaries

### Frontend (`app/`)

**Contains:** Next.js routes, React components, services, hooks, i18n, styles

**Does NOT contain:** Backend logic, database migrations, AI specs

**Pre-commit:** format, lint, types, build

### Backend (`backend/`)

**Contains:** NestJS modules, TypeORM entities, migrations, WebSocket gateway, services, controllers

**Does NOT contain:** Frontend code, AI specs

**Pre-commit:** format, lint, types, build

### Platform (`cuisineease-platform/`)

**Contains:** AI specs, architecture docs, release reports, governance, business docs

**Does NOT contain:** Application source code (`.ts`, `.tsx`, `.js`, `.jsx`), `node_modules`, `.env` files, build artifacts

---

## Coordination Rules

1. **API contract changes** require coordinated deploy: backend first, then frontend
2. **Migration deploys** run before backend redeploy
3. **`ai/` updates** ship in the same PR as the code change (in frontend or backend repo), then sync to platform repo
4. **No cross-repo commits** -- never mix frontend and backend changes in one commit
5. **Release verification** must pass in both `app/` and `backend/` before production deploy

---

## Deploy Sequence

```
1. Run migrations on production DB
2. Redeploy backend (Render)
3. Redeploy frontend (Vercel) if env/client changes
4. Verify: NEXT_PUBLIC_API_URL, NEXT_PUBLIC_WS_URL, CORS_ORIGINS
```

---

## Where `ai/` Lives

The `ai/` directory exists in **both** the platform repo (canonical) and the workspace root. When updating specs:

1. Update in platform repo (or workspace `ai/`)
2. Ensure both locations stay synchronized
3. Reference `ai/TASKS.md` and `ai/DECISIONS.md` in PRs to frontend/backend
