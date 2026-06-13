# CuisineEase -- Platform State

**Last updated:** 2026-06-13
**Status:** Release Candidate -- Post Phase 1 Enterprise Audit
**Release Score:** 74/100

---

## System Overview

CuisineEase is a multi-restaurant food ordering and operations platform connecting customers, restaurant staff (managers, chefs, waitstaff), delivery drivers, and platform admins. Web-first, mobile-responsive. Target market: Cameroon/West Africa (Mobile Money via MeSomb).

---

## Architecture Summary

| Layer | Technology | Host |
|-------|-----------|------|
| Frontend | Next.js 14, React 18, TypeScript, Tailwind + shadcn/ui, TanStack Query, Socket.io-client | Vercel |
| Backend | NestJS 11, TypeORM, PostgreSQL, Redis, Firebase Auth/Storage/FCM, Socket.io | Render |
| Realtime | Socket.io on API HTTP server (shared port) | Same as API |
| Payments | MeSomb (MTN/Orange Mobile Money) + Cash on Delivery | Backend gateway |
| Auth | Firebase JWT -> Postgres UUID resolution | Firebase + backend |

---

## Repository Separation Model

| Repository | Contains | Does NOT contain |
|------------|----------|-----------------|
| `frontend-web` | Next.js routes, components, services, hooks, i18n | Backend logic, DB migrations, AI specs |
| `backend` | NestJS modules, entities, migrations, WebSocket, services | Frontend code, AI specs |
| `cuisineease-platform` | AI specs, architecture, governance, release reports, business docs | Application source code |

---

## Portal Maturity

| Portal | Path | Status | Notes |
|--------|------|--------|-------|
| **Customer** | `/customer` | Production | Browse, order, track, chat, reserve, EN/FR |
| **Manager** | `/manager` | Production | Dashboard, menu, orders, staff, tables, inventory beta, Ctrl+K search |
| **Waitstaff** | `/waitstaff` | Complete | Floor sessions, orders, billing, reservations, i18n EN/FR |
| **Kitchen** | `/chef` | Release Candidate | Kanban queue, DnD, stations, analytics, offline sync, E2EE, TV mode |
| **Driver** | `/delivery` | Release Candidate | Shift, assignments, proof, chat, performance, i18n EN/FR |
| **Platform Admin** | `/dashboard` | Minimal | Restaurant approval, overview KPIs, user search, health |

---

## Enterprise Audit Completion

### Phase 0 -- Critical (8/8 complete)

| ID | Fix | Status |
|----|-----|--------|
| ENT-C01 | Delivery completion -> order status sync | Done |
| ENT-C02 | Dine-in requires active TableSession | Done |
| ENT-C03 | Manager revenue stats (delivered + completed + served) | Done |
| ENT-C04 | Auto-route paid delivery orders to kitchen | Done |
| ENT-C05 | Non-cash session payments via MeSomb | Done |
| ENT-C06 | WebSocket event name alignment + legacy aliases | Done |
| ENT-C07 | One delivery per order (unique index) | Done |
| ENT-C08 | Live restaurant RBAC endpoint + frontend hook | Done |

### Phase 1 -- High (14/14 complete)

| ID | Fix | Status |
|----|-----|--------|
| ENT-H01 | Manager driver picker dropdown | Done |
| ENT-H02-H03 | Reservation lifecycle (transitions, cancel, reschedule) | Done |
| ENT-H04 | Staff role edit after invite | Done |
| ENT-H05 | Inventory beta + ingredient CRUD | Done |
| ENT-H06 | Consume payment:recorded WS on waitstaff billing | Done |
| ENT-H07 | Driver offline sync wiring | Done |
| ENT-H08 | Waitstaff i18n EN/FR | Done |
| ENT-H09 | Chef notifications full inbox | Done |
| ENT-H10 | Expose expireSession API route | Done |
| ENT-H11 | Replace stub AnalyticsService | Done |
| ENT-H12 | Manager global search (Ctrl+K) | Done |
| ENT-H13 | Notify driver when kitchen-ready | Done |
| Release | Verification -- builds + enterprise tests pass | Done |

---

## Known Risks

### HIGH

| ID | Issue | Status |
|----|-------|--------|
| SEC-001 | `tables.controller.ts` write endpoints unguarded (guards commented out, file frozen per ADR-013) | Open -- backlog |

### MEDIUM

| ID | Issue | Status |
|----|-------|--------|
| SEC-002 | Customer `PATCH /orders/:id` allows non-cancel transitions | Open |
| SEC-003 | `assignedWaitstaffId` not validated against restaurant role | Open |
| SEC-004 | WebSocket JWT `restaurantId` claim auto-join without re-check | Mitigated |
| SEC-005 | Reservation availability endpoint lacks `restaurantId` param | Open |

### LOW

| ID | Issue | Status |
|----|-------|--------|
| SEC-006 | Broad notification socket rooms | Acceptable for MVP |
| SEC-007 | `ping` handler uses `server.emit` instead of `client.emit` | Open |

### Other Risks

- Debug endpoints `GET /test`, `GET /yes` have no auth guard (pre-existing)
- Legacy NestJS Jest specs are broken/outdated (not blocking builds)
- `husky` in backend `dependencies` instead of `devDependencies`

---

## Pending Migrations

Must run on production DB before deploy (in order):

| Migration | Purpose |
|-----------|---------|
| `1772880000000-WaitstaffFloor` | TableSession, Guest, order extensions |
| `1772890000000-TableSessionBillRequested` | `billRequestedAt` field |
| `1772900000000-TableSessionExpired` | Session `expired` status |
| `1772910000000-KitchenDashboard` | Kitchen module tables |
| `1772920000000-KitchenEnterprisePolish` | E2EE, chat extensions, analytics |
| `1772930000000-DriverDashboard` | Driver shifts, events, offline actions |
| `1772940000000-EnterprisePhase0` | Unique delivery-per-order index |
| `1772950000000-IngredientStock` | Ingredient quantity, par level, unit |

---

## Next Priorities

1. **Release** -- commit pending work, deploy, run migrations
2. **SEC-001** -- fix tables controller RBAC
3. **PAY-001** -- complete MeSomb gateway processing
4. **QA-001/002/003** -- integration tests (auth, orders, chat)
5. **N-12** -- notification WebSocket push (replace 15s polling)

---

## Related Documents

- Governance: [`GOVERNANCE.md`](./GOVERNANCE.md)
- Release gates: [`../docs/RELEASE_GATES.md`](../docs/RELEASE_GATES.md)
- Architecture: [`../docs/ARCHITECTURE_OVERVIEW.md`](../docs/ARCHITECTURE_OVERVIEW.md)
- Tasks: [`TASKS.md`](./TASKS.md)
- Security backlog: [`SECURITY_BACKLOG.md`](./SECURITY_BACKLOG.md)
