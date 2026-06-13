# CuisineEase — Project

**Last updated:** 2026-06-12  
**Status:** Active development (production deployments exist)  
**Confidence:** High for technical facts; medium for business priorities

---

## What this is

CuisineEase is a **multi-restaurant food ordering and operations platform**. It connects:

- **Customers** — browse menus, order (dine-in / takeout / delivery), reserve tables, track orders, chat with restaurants and drivers
- **Restaurant staff** — manage menu, orders, tables, inventory, staff, reservations, reviews, and customer messages
- **Delivery drivers** — view assigned deliveries and chat with customers during active delivery
- **Platform admins** — approve/suspend restaurants and oversee the network

The product targets **web-first** usage with mobile-responsive UI. Payment integration emphasizes **Mobile Money (MeSomb: MTN/Orange)** and **cash on delivery**, consistent with Cameroon/West Africa market signals in the codebase.

---

## Repository layout

This workspace is a **local monorepo folder** containing two **independent Git repositories**:

| Path | Repo | Remote (primary) |
|------|------|------------------|
| `app/` | Frontend (Next.js) | `CuisineEase/frontend-web` (also `TamahJustene/CuisineEase-frontend`) |
| `backend/` | API (NestJS) | `CuisineEase/backend` |
| `ai/` | Specification docs (this folder) | Not yet tied to a remote — **decide where `ai/` should live** |
| `docs/` | Miscellaneous docs | Unknown scope |

There is **no root-level Git repo** in this workspace today.

---

## Production topology (inferred)

| Component | Host | URL pattern |
|-----------|------|-------------|
| Frontend | Vercel | `https://www.cuisineease.com` (also legacy `cuisine-ease-frontend.vercel.app` in backend config) |
| Backend API | Render | `https://api.cuisineease.com` or `https://cuisine-ease-backend.onrender.com` |
| Database | PostgreSQL | Hosted (connection via env) |
| Auth | Firebase Auth | Project `cuisine-ease` |
| File storage | Firebase Storage (default) | Bucket in Firebase config |
| Realtime | Socket.io on API HTTP server | Same host as REST API |
| Push | Firebase Cloud Messaging | Via device tokens |

---

## Users and roles

### System roles (`SystemRole`)

| Role | Portal | Path prefix |
|------|--------|-------------|
| `user` | Customer | `/customer` |
| `delivery` | Driver | `/delivery` |
| `admin` | Platform admin | `/dashboard` |

### Restaurant roles (`RestaurantRole`)

Assigned per restaurant via `UserRestaurantRole`. JWT may include `restaurantId` claim.

| Role | Portal | Path prefix |
|------|--------|-------------|
| `owner`, `manager` | Restaurant manager | `/manager` |
| `chef` | Kitchen display | `/chef` |
| `waitstaff` | Front-of-house | `/waitstaff` |
| `delivery` | Restaurant-scoped delivery staff | (backend; may overlap with system delivery) |

**Important:** Restaurant managers often retain `SystemRole.USER` while holding restaurant roles. Features must check **restaurant staff membership**, not only system role (see chat fixes in `DECISIONS.md`).

---

## Core capabilities (implemented)

### Customer
- Restaurant discovery (including near-me), menu browsing, favorites
- Cart and checkout (delivery address/time, order types)
- Order history and tracking
- Table reservations
- Notifications (inbox, WebSocket + polling)
- Chat with restaurant and driver (image attachments)
- Profile, addresses, security, notification preferences
- EN/FR i18n

### Restaurant manager
- Dashboard (stats, maps, recent activity)
- Menu CRUD (categories, ingredients, add-ons, images)
- Orders lifecycle
- Tables and sections
- Inventory
- Staff and roles
- Reservations
- Reviews
- Messages (customer chat inbox)
- Pending/suspended restaurant states

### Platform admin
- Restaurant approval workflow
- Notifications

### Delivery
- Delivery list dashboard
- Customer chat during active delivery

### Public
- Marketing site, public menu pages, contact, terms/privacy

---

## Core capabilities (partial / stub)

| Area | State | Evidence |
|------|-------|----------|
| Payment processing (MeSomb) | Partial | `payment.service.ts` TODOs |
| Kitchen display system (KDS) | Stub | `kitchen-display.service.ts` |
| Order analytics | Stub | `analytics.service.ts` |
| Role guards on all endpoints | Inconsistent | e.g. `tables.controller.ts`, `orders.controller.ts` |
| Automated tests | Minimal | Mostly boilerplate specs |
| Chef portal | Production | Full `/chef` portal K0–K12 — see `ai/features/kitchen/KITCHEN_COMPLETION_REPORT.md` |
| Delivery driver UX | Partial | List + chat; assignment flows vary |
| Email/SMS notification channels | Partial | WebSocket + FCM primary |

---

## Non-goals (for now)

Not evidenced as current priorities in code or recent work:

- Native mobile apps (React Native / Flutter)
- Multi-tenant white-label per restaurant domain
- Full POS hardware integration
- Comprehensive reporting/BI suite

Revisit when product owner confirms.

---

## Documentation conflicts (code vs docs)

| Source | Says | Reality in code |
|--------|------|-----------------|
| `backend/README.md` | Default NestJS starter | Not project-specific |
| `backend/mindmap.md` "Next Steps" | Roles, payments, notifications, admin dashboard "to implement" | Largely implemented |
| `app/.env.example` | MongoDB, Cloudinary, GitHub OAuth | Not used in active `src/` paths |
| `backend/src/config/app-urls.ts` | Production frontend = `cuisine-ease-frontend.vercel.app` | `.env.example` uses `www.cuisineease.com` |
| `app/README.md` | Generic Next.js starter | Not project-specific |

**Resolution:** Treat `ai/*.md` as source of truth going forward; deprecate or update stale files when touched.

---

## Open questions (need product owner input)

1. **Primary market launch** — Cameroon only, or multi-country from day one?
2. **`ai/` repo location** — Commit under `app/`, `backend/`, or a new meta-repo?
3. **Chef portal** — Production-critical or placeholder? (Waitstaff portal is production-complete — see `ai/features/waitstaff/`)
4. **Payment scope** — Is MeSomb live in production, or still COD-only for customers?
5. **Next quarter priority** — Reliability/ops, payments, mobile PWA, or new features?
6. **Platform admin usage** — Is `/dashboard` actively used, and by whom?

---

## Related files

- **Workflow hub:** `ai/workflow/README.md` — flows, definitions, setup, API map
- **Feature specs:** `ai/features/README.md` — customer, manager, waitstaff, kitchen
- Architecture: `ai/ARCHITECTURE.md`
- Roadmap: `ai/ROADMAP.md`
- Tasks: `ai/TASKS.md`
- Decisions: `ai/DECISIONS.md`
- Agent workflow: `ai/AGENT.md`
