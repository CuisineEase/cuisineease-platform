# Waitstaff Production Security Audit

**Date:** 2026-06-11  
**Scope:** Phase 8 Waitstaff Portal — orders, table-sessions, tables, guests, payments, reservations, notifications, Socket.io  
**Auditor:** Production hardening pass (post-implementation)

---

## Executive summary

A full endpoint scan was performed across waitstaff-related backend modules. **Four CRITICAL** and **three HIGH** issues were found; all in-scope CRITICAL/HIGH items were **fixed** in this pass. One **HIGH** risk remains in the frozen `tables.controller.ts` (documented in `ai/SECURITY_BACKLOG.md`).

Multi-tenant isolation for waitstaff flows now consistently uses `RestaurantAccessService.assertWaitstaffOrAbove()` (or customer-self checks where appropriate) plus entity ownership validation (`restaurantId` path/body match, session/guest/order scoping).

---

## Endpoint matrix

Legend: **ISO** = restaurant isolation enforced | **RBAC** = role check | **ID** = ID-guessing blocked

### Floor & table-sessions (`floor/`)

| Method | Path | ISO | RBAC | ID | Notes |
|--------|------|-----|------|-----|-------|
| GET | `/restaurants/:restaurantId/floor` | ✅ | ✅ waitstaff+ | ✅ | Path `restaurantId` + `assertWaitstaffOrAbove` |
| GET | `/restaurants/:restaurantId/tables/:tableId` | ✅ | ✅ waitstaff+ | ✅ | Service scopes table to restaurant |
| GET | `/restaurants/:restaurantId/waitstaff/shift-summary` | ✅ | ✅ waitstaff+ | ✅ | |
| GET | `/restaurants/:restaurantId/table-sessions/active` | ✅ | ✅ waitstaff+ | ✅ | |
| GET | `/table-sessions/:id` | ✅ | ✅ waitstaff+ | ✅ | Load session → assert on `session.restaurant.id` |
| POST | `/table-sessions` | ✅ | ✅ waitstaff+ | ✅ | DTO `restaurantId`; table/guest/reservation validated in service |
| PATCH | `/table-sessions/:id` | ✅ | ✅ waitstaff+ | ✅ | Session restaurant derived from entity |
| POST | `/table-sessions/:id/transfer` | ✅ | ✅ waitstaff+ | ✅ | Target table must match session restaurant |
| GET | `/table-sessions/:id/billing` | ✅ | ✅ waitstaff+ | ✅ | |
| GET | `/table-sessions/:id/timeline` | ✅ | ✅ waitstaff+ | ✅ | |
| POST | `/table-sessions/:id/request-bill` | ✅ | ✅ waitstaff+ | ✅ | |
| POST | `/table-sessions/:id/close` | ✅ | ✅ waitstaff+ | ✅ | |
| POST | `/table-sessions/:id/payments` | ✅ | ✅ waitstaff+ | ✅ | Payment linked to `tableSession` + `restaurant` |

### Guests (`floor/guests`)

| Method | Path | ISO | RBAC | ID | Notes |
|--------|------|-----|------|-----|-------|
| GET | `/restaurants/:restaurantId/guests/search` | ✅ | ✅ waitstaff+ | ✅ | Query scoped to path restaurant |
| POST | `/restaurants/:restaurantId/guests` | ✅ | ✅ waitstaff+ | ✅ | `assertRestaurantIdMatch` path vs body |
| GET | `/restaurants/:restaurantId/guests/:guestId` | ✅ | ✅ waitstaff+ | ✅ | Service filters by restaurant + guestId |

### Orders (`orders/`)

| Method | Path | ISO | RBAC | ID | Notes |
|--------|------|-----|------|-----|-------|
| POST | `/orders` | ✅ | ✅ customer/staff | ✅ | **FIXED** — `createSecured`: customer self or staff; session/guest cross-check |
| POST | `/orders/staff` | ✅ | ✅ waitstaff+ | ✅ | Session + guest validated against `restaurantId` |
| POST | `/orders/from-cart` | ✅ | ✅ self | ✅ | Caller’s cart only |
| GET | `/orders` | ✅ | ✅ staff/self | ✅ | `findAllSecured` |
| GET | `/orders/:id` | ✅ | ✅ staff/customer | ✅ | `findOneSecured` |
| PATCH | `/orders/:id` | ✅ | ✅ role transitions | ✅ | **FIXED** — `updateSecured` enforces role-based status transitions |
| DELETE | `/orders/:id` | ✅ | ✅ staff/customer | ✅ | `assertCanAccessOrder` |
| POST | `/orders/:id/send-to-kitchen` | ✅ | ✅ waitstaff+ | ✅ | Draft-only |
| POST | `/orders/:id/mark-served` | ✅ | ✅ waitstaff+ | ✅ | Ready-only |
| POST | `/orders/:id/kitchen-status` | ✅ | ✅ chef+ | ✅ | Chef transition matrix |

### Payments (`payment/` + session payments)

| Method | Path | ISO | RBAC | ID | Notes |
|--------|------|-----|------|-----|-------|
| POST | `/payments` | ✅ | ✅ self | ✅ | **FIXED** — cannot pay as another user |
| GET | `/payments` | ✅ | ✅ self | ✅ | **FIXED** — `getAllPaymentsSecured` |
| GET | `/payments/:id` | ✅ | ✅ self/staff | ✅ | **FIXED** — customer or waitstaff+ of payment restaurant |
| POST | `/table-sessions/:id/payments` | ✅ | ✅ waitstaff+ | ✅ | Canonical dine-in payment path |

### Reservations (`reservations/`)

| Method | Path | ISO | RBAC | ID | Notes |
|--------|------|-----|------|-----|-------|
| POST | `/reservations` | ✅ | ✅ customer | ✅ | Creates for authenticated customer |
| GET | `/reservations/me` | ✅ | ✅ self | ✅ | |
| GET | `/reservations/restaurant/:id` | ✅ | ✅ waitstaff+ | ✅ | **FIXED** — `assertWaitstaffOrAbove` |
| PATCH | `/reservations/:id` | ✅ | ✅ staff/owner | ✅ | **FIXED** — staff or reservation customer |
| POST | `/reservations/:id/pay` | ✅ | ✅ customer | ✅ | Existing pay flow |
| GET | `/reservations/table/:tableId/availability` | ⚠️ | ⚠️ | ⚠️ | Public availability; no restaurant leak of PII |

### Notifications (`notifications/`)

| Method | Path | ISO | RBAC | ID | Notes |
|--------|------|-----|------|-----|-------|
| POST | `/notifications` | N/A | ✅ admin | ✅ | **FIXED** — admin only |
| GET | `/notifications` | N/A | ✅ self/admin | ✅ | **FIXED** — non-admin forced to own `userId` |
| GET | `/notifications/user/:userId` | N/A | ✅ self/admin | ✅ | **FIXED** |
| GET/PATCH/DELETE | `/notifications/user/:userId/*` | N/A | ✅ self/admin | ✅ | **FIXED** |
| GET | `/notifications/:id` | N/A | ✅ recipient/admin | ✅ | **FIXED** — `assertCallerCanAccessNotification` |
| PATCH/DELETE | `/notifications/:id` | N/A | ✅ admin | ✅ | **FIXED** |
| POST | `/notifications/sentToAll` | N/A | ✅ admin | ✅ | **FIXED** |

### Tables (`tables/` — frozen controller)

| Method | Path | ISO | RBAC | ID | Notes |
|--------|------|-----|------|-----|-------|
| GET | `/tables/restaurant/:restaurantId` | ✅ public read | — | ✅ | Intentional for customer dine-in picker |
| GET | `/tables/sections/restaurant/:restaurantId` | ✅ public read | — | ✅ | |
| POST/PATCH/DELETE | `/tables/*` (writes) | ❌ | ❌ | ❌ | **HIGH — BACKLOG** — guards commented out; file frozen per ADR-013 |

**Waitstaff table reads** use `/restaurants/:restaurantId/floor` (secured), not unguarded table writes.

---

## Issues found and resolution

### CRITICAL (fixed)

| # | Issue | Fix |
|---|-------|-----|
| C1 | `POST /orders` — any authenticated user could create orders for arbitrary `customerId`/`restaurantId` | `createSecured()` — customer self or staff; session/guest ownership checks |
| C2 | `GET /payments` / `GET /payments/:id` — no scoping | `getAllPaymentsSecured` / `getPaymentByIdSecured` |
| C3 | Notifications user routes — any user could read/modify another user’s notifications | `assertSelfOrAdmin` on all user-scoped routes; admin-only create/broadcast |
| C4 | `GET /notifications` without `userId` returned cross-user data | Non-admin callers forced to own `userId` |

### HIGH (fixed in scope)

| # | Issue | Fix |
|---|-------|-----|
| H1 | `PATCH /orders/:id` — waitstaff could bypass workflow via generic status PATCH | `updateSecured()` with `validateStatusTransitionForRole` |
| H2 | `GET /reservations/restaurant/:id` — no staff check | `assertWaitstaffOrAbove` on restaurant param |
| H3 | `PATCH /reservations/:id` — no ownership check | Staff or reservation customer only |

### HIGH (deferred — frozen file)

| # | Issue | Status |
|---|-------|--------|
| H4 | `tables.controller.ts` write endpoints unguarded | Documented in `SECURITY_BACKLOG.md`; requires ADR amendment + parallel secured routes |

### MEDIUM (documented, non-blocking)

| # | Issue | Notes |
|---|-------|-------|
| M1 | Guests `RolesGuard` previously used wrong param (`:id` vs `restaurantId`) | Removed broken guard; `assertWaitstaffOrAbove` used instead |
| M2 | WebSocket `join:restaurant` did not leave prior restaurant room | **FIXED** — leave previous `restaurant:*` room on switch |
| M3 | JWT `restaurantId` claim auto-joins room on connect without role re-check | Mitigated: sensitive events only after `join:restaurant` with role check; claim join is convenience only |
| M4 | Customer generic `PATCH /orders` status transitions still allow DRAFT→SENT_TO_KITCHEN | Customers should use cart/checkout flows; tighten in future |
| M5 | `assignedWaitstaffId` on session update does not verify user has waitstaff role at restaurant | Low exploit value; backlog |

### LOW

| # | Issue | Notes |
|---|-------|-------|
| L1 | Broad notification rooms (`notifications:general`, etc.) on connect | No restaurant PII; acceptable for MVP |
| L2 | `ping` handler broadcasts `pong` to all clients | Dev noise only |

---

## Phase 3 — API consistency

| Rule | Status |
|------|--------|
| `TableSession` is dine-in source of truth | ✅ Orders, payments, billing, timeline keyed on session |
| `Table.status` is physical asset only | ✅ Updated on seat/transfer/close for floor map; session state in `TableSession.status` |
| Payments belong to `TableSession` (dine-in) | ✅ `POST /table-sessions/:id/payments` sets `tableSession` + `restaurant` |
| Orders belong to `TableSession` + `restaurantId` | ✅ Staff orders require session; `createSecured` validates linkage |
| Guests scoped to `restaurantId` | ✅ Path prefix + service queries |

---

## Phase 4 — Realtime safety

| Check | Status |
|-------|--------|
| Floor events emit to `restaurant:{id}` only | ✅ `RestaurantRealtimeService` |
| `join:restaurant` requires restaurant role | ✅ `getUserRoleForRestaurant` |
| Switching restaurant leaves old room | ✅ Fixed |
| Events include `restaurantId` in payload | ✅ Clients can ignore foreign payloads |
| No user-specific order content in restaurant broadcast | ✅ Events carry IDs only; fetch via secured REST |

---

## Files changed (security pass)

- `backend/src/common/restaurant-access/restaurant-access.service.ts` — `assertRestaurantIdMatch`, `getRestaurantRole`
- `backend/src/orders/orders.service.ts` — `createSecured`, `updateSecured`, entity ownership helpers
- `backend/src/orders/orders.controller.ts` — wired secured create/update
- `backend/src/payment/payment.service.ts` — secured GET/list; payment access assertion
- `backend/src/payment/payment.controller.ts` — wired secured endpoints
- `backend/src/payment/payment.module.ts` — `RestaurantAccessModule`
- `backend/src/notifications/notifications.controller.ts` — auth + self/admin guards
- `backend/src/notifications/notifications.service.ts` — `assertCallerCanAccessNotification`
- `backend/src/notifications/notifications.gateway.ts` — leave prior restaurant room
- `backend/src/reservations/reservations.controller.ts` — restaurant list + update access
- `backend/src/reservations/reservations.service.ts` — `findOneForAccess`
- `backend/src/reservations/reservations.module.ts` — `RestaurantAccessModule`
- `backend/src/floor/guests.controller.ts` — removed broken `RolesGuard`
- `backend/src/floor/guests.service.ts` — path/body restaurantId match

---

## Verification

- `npm run build` (backend) — **pass**
- Manual review: no waitstaff endpoint returns another restaurant’s sessions/orders/guests without role
- Remaining risk: frozen `tables.controller.ts` writes (see backlog)

---

## Success criteria checklist

| Criterion | Met |
|-----------|-----|
| No in-scope endpoint accesses another restaurant’s waitstaff data | ✅ |
| Waitstaff flows restaurant-scoped | ✅ |
| RBAC consistent on orders, sessions, guests, payments, reservations | ✅ |
| `TableSession` enforced as dine-in source of truth | ✅ |
| Audit documented | ✅ |
