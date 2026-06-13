# CuisineEase — Agent Operating Instructions

**Last updated:** 2026-06-12  
**Audience:** AI coding agents (Cursor, CI bots, contributors using LLMs)

---

## Mission

Help build CuisineEase safely by treating **`ai/` as the source of truth** for intent, architecture, and priorities — not stale READMEs or outdated mindmaps.

---

## Before writing code

1. **Read** (in order, skim if familiar):
   - `ai/PROJECT.md` — what we're building
   - `ai/workflow/README.md` — flows, definitions, setup (start here for portal work)
   - `ai/features/README.md` — portal-specific specs (customer, manager, waitstaff)
   - `ai/ARCHITECTURE.md` — how it's built
   - `ai/TASKS.md` — what's in flight
   - `ai/DECISIONS.md` — constraints you must not violate
   - `ai/ROADMAP.md` — priority context

2. **Classify the request:**
   - **Question only** → answer from code + `ai/`; minimal or no code changes
   - **Bug fix** → reproduce, fix root cause, update `TASKS.md` if tracking an ID
   - **Small change** → implement with minimal diff; match existing conventions
   - **Major feature** → **stop** if spec is missing; update `TASKS.md` + propose design in `DECISIONS.md` first

3. **Major implementation threshold** — requires spec update first:
   - New domain module or DB tables
   - New user role or permission model change
   - Payment flow changes
   - Breaking API contract changes
   - Cross-repo coordinated releases

4. **When confidence is low** (<80% on requirements):
   - Ask clarifying questions (see `PROJECT.md` open questions)
   - Do not guess product rules (especially chat, payments, roles)
   - Document assumptions in `DECISIONS.md` as Proposed until confirmed

---

## Repository rules

### Two repos

| Change type | Path | Push target |
|-------------|------|-------------|
| Frontend | `app/` | `origin/main` + `personal/main` (user preference) |
| Backend | `backend/` | `origin/main` |

Never mix frontend and backend in one commit across repos.

### Pre-commit hooks

Both repos run **format, lint, types, build** on commit. Fix failures; do not skip hooks unless user explicitly requests.

### Commits

Only commit when user asks. Use complete-sentence commit messages focused on **why**.

---

## Code conventions

### Backend (`backend/`)

- NestJS modules: entity → dto → service → controller
- Use `apiSuccess()` for new endpoints unless matching legacy pattern nearby
- New routes: assume `FirebaseGuard` applies; use `@Public()` only when intentional
- Restaurant staff: check `UserRestaurantRole`, not only `SystemRole.USER`
- Schema changes: add TypeORM migration; never rely on prod synchronize
- Chat/notifications: emit via `NotificationsGateway.emitToUser`
- Storage: default Firebase; set path prefix per domain (`chat/{id}/`, etc.)

### Frontend (`app/`)

- API calls in `src/services/`; React Query in `src/hooks/`
- Use `getUuidBackedUserId()` for Postgres UUID APIs
- Unwrap `{ data }` from API responses
- i18n: add keys to `en.json` and `fr.json`
- Mobile-first for inbox patterns (WhatsApp-style list → detail)
- Match shadcn/Tailwind patterns in surrounding files
- Do not add tests unless requested or materially valuable

---

## When documentation and code disagree

1. **Identify** the conflict (cite file paths)
2. **Explain** which is likely correct (prefer runtime behavior + recent commits)
3. **Propose** update to `ai/*.md` and/or legacy doc
4. **Do not** silently implement against `ai/` without noting the conflict

Known stale docs: `backend/README.md`, `app/README.md`, `backend/mindmap.md`, parts of `.env.example`.

---

## After shipping

Update in the **same session/PR**:

| Artifact | When |
|----------|------|
| `ai/TASKS.md` | Mark tasks done; add new follow-ups |
| `ai/DECISIONS.md` | New architectural/product decisions |
| `ai/ROADMAP.md` | Phase status changes |
| `ai/ARCHITECTURE.md` | New modules, endpoints, env vars |
| `ai/PROJECT.md` | Scope or role model changes |

---

## Domain-specific guardrails

### Chat

- One `customer_restaurant` thread per customer+restaurant pair
- Driver chat only during active delivery; auto-close on terminal status
- Staff with `SystemRole.USER` must use restaurant staff code paths
- Image upload goes through backend storage (not direct Cloudinary from browser)

### Auth & roles

- Firebase JWT required for protected routes
- Manager routes: may need `restaurantId` in JWT or localStorage (`getRestaurantId()`)
- Do not trust client role alone for authorization (backend must enforce)

### Payments

- MeSomb is partial — verify before assuming digital payments work end-to-end
- Frontend currently emphasizes Mobile Money in modal

### Realtime

- WebSocket URL = API host (not port 3002)
- Invalidate or patch `chat-conversations`, `chat-messages`, `chat-unread-count` query keys

---

## Deployment reminders

After backend changes affecting schema or sockets:

1. Run migrations on production DB
2. Redeploy backend (Render)
3. Redeploy frontend if env or client changes (Vercel)
4. Verify: `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`, `STORAGE_TYPE`, `CORS_ORIGINS`

---

## Questions to ask the product owner

When starting significant work, confirm if unknown:

1. Target market and payment methods in production today
2. Priority: reliability vs features vs payments
3. Whether chef/waitstaff portals are in active use
4. Where to version `ai/` documentation
5. Acceptable breaking API changes vs backward compatibility

---

## Anti-patterns (do not)

- Implement large features without updating `ai/TASKS.md`
- Use `SystemRole.USER` alone for restaurant manager logic
- Expose WebSocket on a separate port
- Commit secrets or `.env` files
- Force-push `main` without explicit user request
- Over-engineer abstractions for one-off helpers
- Trust `backend/mindmap.md` next steps as current

---

## Quick file index

| Need | Location |
|------|----------|
| Workflow & flows | `ai/workflow/` |
| Customer / manager specs | `ai/features/customer/`, `ai/features/manager/` |
| Kitchen / waitstaff specs | `ai/features/kitchen/`, `ai/features/waitstaff/` |
| Order status UI layers | `app/src/lib/orderStatusViews.ts` |
| Customer nav | `app/src/config/customerNav.ts` |
| Manager nav | `app/src/config/managerNav.ts` |
| API client | `app/src/lib/api.ts` |
| Auth session | `app/src/services/auth.ts` |
| Role routing | `app/src/middleware.ts`, `app/src/lib/authRouting.ts` |
| Chat service (BE) | `backend/src/chat/chat.service.ts` |
| Chat UI | `app/src/components/Chat/` |
| Notifications gateway | `backend/src/notifications/notifications.gateway.ts` |
| Migrations | `backend/src/lib/database/migrations/` |
| CORS/URLs | `backend/src/config/app-urls.ts` |
| Env samples | `backend/.env_sample`, `app/.env.example` |

---

## Version

Spec OS v1.0 — initial audit 2026-06-12
