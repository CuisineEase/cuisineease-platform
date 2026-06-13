# CuisineEase — Roadmap

**Last updated:** 2026-06-11  
**Horizon:** Inferred from codebase maturity + recent work; **not product-owner validated**

---

## How to read this

Phases are ordered by **dependency and risk**, not calendar dates. Update when priorities are confirmed in `ai/PROJECT.md` open questions.

| Status | Meaning |
|--------|---------|
| ✅ Done | Shipped in code + deployed (best knowledge) |
| 🟡 Partial | Exists but incomplete or inconsistent |
| 🔴 Not started | No meaningful implementation |
| 📋 Planned | Agreed direction, not yet built |

---

## Phase 0 — Foundation (✅ largely complete)

- [x] NestJS API + PostgreSQL + TypeORM migrations
- [x] Firebase authentication (email + Google)
- [x] Next.js customer + manager portals
- [x] Restaurant onboarding and approval workflow
- [x] Menu, cart, orders, reservations
- [x] Role-based routing (customer, manager, admin, delivery)
- [x] EN/FR i18n (customer/manager surfaces)
- [x] Deploy frontend (Vercel) + backend (Render)

---

## Phase 1 — Operations reliability (🟡 in progress)

**Goal:** Production-stable core flows without silent failures.

| Item | Status | Notes |
|------|--------|-------|
| Global Firebase guard + CORS hardening | ✅ | Phase A security |
| Chat module (customer↔restaurant, customer↔driver) | ✅ | Migration `1772870000000` |
| Chat staff access for `USER` + restaurant roles | ✅ | Commit `914fabe` |
| Socket.io on API port (not :3002) | ✅ | Commit `dd07b89` |
| Chat image upload (Firebase storage default) | ✅ | `STORAGE_TYPE` default fix |
| Realtime chat UX (optimistic send, previews, unread) | ✅ | Commit `93926f4` |
| Notification inbox master-detail UX | ✅ | Commit `47e3fee` |
| Run migrations on all production DBs | 📋 | Manual ops step |
| `NEXT_PUBLIC_WS_URL` set in Vercel | 📋 | Should match API host |
| Consistent API response envelope | 🟡 | Some endpoints diverge |
| Role guard coverage audit | 🔴 | Tables, orders gaps |
| Notification cleanup cron | 🔴 | Commented out |
| Integration test suite (auth, orders, chat) | 🔴 | |

---

## Phase 2 — Payments & money flow (🟡 partial)

**Goal:** Customers pay digitally; restaurants reconcile payments.

| Item | Status | Notes |
|------|--------|-------|
| Cash on delivery gateway | ✅ | `cash.gateway.ts` |
| MeSomb Mobile Money integration | 🟡 | Gateway exists; service TODOs |
| Frontend payment modal (Mobile Money focus) | ✅ | Card/PayPal commented out |
| Payment notifications | ✅ | Event listener wired |
| Refunds / failed payment recovery | 🟡 | Entity exists; flow unclear |
| Reservation payments | 🟡 | Migration + DTOs exist |

**Blocked on:** Product confirmation that MeSomb is live and which currencies/operators matter.

---

## Phase 3 — Restaurant operations depth (🟡 partial)

**Goal:** Replace ad-hoc tools for day-to-day restaurant staff.

| Item | Status | Notes |
|------|--------|-------|
| Manager dashboard (stats, maps) | ✅ | |
| Order management + status updates | ✅ | |
| Table/section management | ✅ | |
| Inventory + ingredients | 🟡 | Beta ingredient CRUD; usage history deferred |
| Staff invite + roles | ✅ | |
| Kitchen dashboard (`/chef`, `/manager/kitchen`) | 🟢 | Release candidate — K0–K12 + final polish delivered |
| Driver portal (`/delivery`) | 🟢 | Release candidate — see `ai/features/driver/` |
| Waitstaff portal (`/waitstaff`) | ✅ | Phase 8 + EN/FR i18n pass complete |
| Analytics service | ✅ | Structured metrics (ENT-H11) |
| Manager global search | ✅ | Ctrl+K workspace search (ENT-H12) |
| Reservation lifecycle | ✅ | Staff + customer actions (ENT-H02–H03) |

---

## Phase 4 — Customer growth features (🟡 partial)

| Item | Status | Notes |
|------|--------|-------|
| Near-me restaurants | ✅ | |
| Delivery checkout (address, time) | ✅ | |
| Favorites (restaurants, menu items) | ✅ | |
| Order tracking maps | ✅ | |
| Customer statistics | ✅ | |
| Push notifications (FCM) | 🟡 | Backend wired; client opt-in varies |
| PWA / offline | 🟡 | manifest exists; limited offline |
| Native apps | 🔴 | Not planned in code |

---

## Phase 5 — Platform & scale (🔴 early)

| Item | Status | Notes |
|------|--------|-------|
| Platform admin dashboard | 🟡 | Overview, users, activity, health, moderation — broadcasts deferred |
| Multi-region deployment | 🔴 | Single Render instance implied |
| Observability (logs, metrics, alerts) | 🔴 | No APM in repo |
| Rate limiting / abuse protection | 🟡 | Notification rate limit in-memory |
| Admin reporting | 🔴 | |

---

## Phase 6 — Specification-driven development (📋 started)

| Item | Status | Notes |
|------|--------|-------|
| `ai/` operating system docs | ✅ | This audit |
| Spec before major features | 📋 | Policy in `AGENT.md` |
| ADR log (`DECISIONS.md`) | ✅ | Initial entries |
| Task tracking (`TASKS.md`) | ✅ | Initial backlog |
| CI test gate | 🔴 | Pre-commit builds only |

---

## Suggested priority order (agent recommendation)

Until product owner reprioritizes:

1. **Ops:** migration checklist, env var verification, role guard audit
2. **Payments:** complete MeSomb path or document COD-only production mode
3. **Quality:** integration tests for auth, orders, chat
4. **Staff tools:** Kitchen dashboard — **complete** (see `ai/features/kitchen/KITCHEN_COMPLETION_REPORT.md`)
5. **Growth:** push notification polish, PWA install prompts

---

## Out of scope (until confirmed)

- Blockchain / crypto payments
- Third-party delivery fleet API (Uber Direct, etc.)
- Franchise/multi-brand corporate hierarchy
- AI menu recommendations

---

## Roadmap update triggers

Update this file when:

- Product owner answers `PROJECT.md` open questions
- A phase completes or is deprioritized
- A production incident reveals architectural debt
- Major feature ships (add to Phase tables + move items to ✅)
