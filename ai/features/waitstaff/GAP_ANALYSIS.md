# Waitstaff Portal — Phase 3 Gap Analysis

**Date:** 2026-06-11  
**Baseline:** `AUDIT.md` (current code) vs `SPEC.md` (target)  
**Status:** Pre-implementation

---

## Summary

| Layer | Complete | Partial | Missing | Broken |
|-------|----------|---------|---------|--------|
| Backend domain | 2 | 6 | 4 | 2 |
| Backend API/security | 1 | 4 | 3 | 2 |
| Frontend | 0 | 3 | 12 | 1 |
| Realtime | 2 | 1 | 2 | 0 |
| **Overall portal** | **~15%** | **~35%** | **~45%** | **~5%** |

---

## Existing capabilities (reuse as-is)

| Capability | Location | Waitstaff use |
|------------|----------|---------------|
| Order CRUD + status machine | `orders/` | Core lifecycle |
| Order list/filter hooks | `useOrdersList` | Boards |
| Table config read | `GET /tables/restaurant/:id` | Floor labels, capacity |
| Reservation list/create/update | `reservations/` | Base data |
| Manager reservations UI patterns | `manager/reservations/page.tsx` | Copy patterns |
| Manager tables UI patterns | `manager/tables/page.tsx` | Read-only reference |
| Staff invite + roles | `user-restaurant-role.service.ts` | Access control |
| JWT `restaurantId` + `role` claims | `syncFirebaseRestaurantClaims` | Routing |
| Notifications + WebSocket | `notifications/` | Alerts |
| Chat module | `chat/` | Messaging |
| Menu items API | `menu/` | Order creation |
| Payment (customer MM) | `payment/` | Reference for cash extension |
| Design system + layout patterns | `manager/layout.tsx` | Shell template |
| i18n infrastructure | `locales/` | Translation |

---

## Missing capabilities

### Critical (P0) — MVP blockers

| # | Gap | Spec reference |
|---|-----|----------------|
| P0-1 | Waitstaff layout, nav, protected sub-routes | Navigation |
| P0-2 | `TableSession` entity + floor API | API A |
| P0-3 | Restaurant-scoped RBAC on orders | Permission matrix |
| P0-4 | Order actions: send-to-kitchen, mark-served, assign waitstaff | API C |
| P0-5 | Reservation FOH statuses + check-in/no-show | API B |
| P0-6 | Overview dashboard composing tables/orders/reservations | Screen 1 |
| P0-7 | Floor view + table detail with seat/transfer | Screens 2–4 |
| P0-8 | Order board + detail with staff actions | Screens 7–8 |
| P0-9 | Wire `useNotificationLive` + `useChatLive` on waitstaff layout | Realtime |
| P0-10 | Fix auth fallback: don't default unknown restaurant users to `manager` | Audit §1 |

### High (P1) — Production completeness

| # | Gap | Spec reference |
|---|-----|----------------|
| P1-1 | Create order flow + menu picker | Screen 9 |
| P1-2 | Guest search + profile + order history APIs | API D |
| P1-3 | Request bill + manager notification | API C |
| P1-4 | Record cash payment endpoint | API C |
| P1-5 | Restaurant socket room + `order:updated` / `floor:updated` | Realtime |
| P1-6 | Reservations list UI for waitstaff | Screen 5–6 |
| P1-7 | `waitstaff` i18n namespace | NFR |
| P1-8 | Tablet-optimized touch UI | NFR |
| P1-9 | Upgrade chef portal with shared `StaffOrderBoard` + actions | Chef parity |

### Medium (P2) — Enhancements

| # | Gap |
|---|-----|
| P2-1 | Shift summary API + page |
| P2-2 | Table notes (jsonb on session) |
| P2-3 | Section-based waiter assignment |
| P2-4 | New notification types + copy |
| P2-5 | Guest preferences/allergies on `User` entity |
| P2-6 | `ShiftTask` entity for manager-assigned tasks |
| P2-7 | Walk-in guest without account (ephemeral user) |

### Low (P3) — Future

| # | Gap |
|---|-----|
| P3-1 | Floor plan designer / coordinates |
| P3-2 | Item-level kitchen status |
| P3-3 | KDS implementation in `kitchen-display.service.ts` |
| P3-4 | Split bill |

---

## Required backend work

### Database migrations (new)

| Migration | Tables/columns |
|-----------|----------------|
| `TableSessions` | `table_sessions` table (see SPEC) |
| `ReservationStatus` | Extend enum: `checked_in`, `seated`, `no_show`, `completed` |
| `Order` | Optional: `billRequestedAt`, `servedAt`; ensure `waitstaffId` used |
| `User` (optional P2) | `dietaryNotes`, `preferences` jsonb |

### New modules / services

| Module | Responsibility |
|--------|----------------|
| `table-sessions/` | Session CRUD, floor aggregation, transfer, close |
| `waitstaff/` or `floor/` controller | Shift summary, guest search (thin orchestration) |

### Changes to existing modules

| Module | Change | Risk |
|--------|--------|------|
| `orders/` | RBAC, DTO fields, action endpoints, cash payment | Medium — don't break customer cart |
| `reservations/` | Status enum, check-in endpoints, enable WAITSTAFF roles | Low |
| `notifications/` | New types, bill request copy | Low |
| `notifications/gateway` or new gateway | Restaurant rooms | Medium |
| `orders/events` | Emit floor-related events on session link | Low |
| `payment/` | Staff cash recording (or orders service) | Medium |
| `tables/` | **Avoid** — read only | High if touched |

### API changes (breaking risk)

| Change | Breaking? | Mitigation |
|--------|-----------|------------|
| Add reservation statuses | No (additive enum) | Default existing rows unchanged |
| Add order endpoints | No | New routes |
| Enforce RolesGuard on orders PATCH | **Yes** for unauthorized clients | Deploy with frontend; audit callers |
| TableSession APIs | No | New routes |

---

## Required frontend work

### New routes & layout

```
app/src/app/waitstaff/layout.tsx
app/src/app/waitstaff/page.tsx          (replace stub)
app/src/app/waitstaff/tables/...
app/src/app/waitstaff/reservations/...
app/src/app/waitstaff/orders/...
app/src/app/waitstaff/guests/...
app/src/app/waitstaff/chat/...
app/src/app/waitstaff/notifications/...
app/src/app/waitstaff/shift/...
app/src/config/waitstaffNav.ts
app/src/components/Waitstaff/** (see SPEC components)
```

### New hooks & services

| Hook/service | Purpose |
|--------------|---------|
| `useFloor()` | Table sessions + tables |
| `useTableSession()` | Mutations: seat, transfer, close |
| `useWaitstaffOrders()` | Extended order actions |
| `services/tableSession.ts` | API client |
| `services/waitstaff.ts` | Shift summary, guest search |
| `useRestaurantSocket()` | Join room, invalidate queries |

### Changes to existing

| File | Change |
|------|--------|
| `middleware.ts` | Ensure waitstaff claim routing; review manager fallback |
| `services/auth.ts` | Remove dangerous `manager` default for unknown roles |
| `locales/en.json`, `fr.json` | `waitstaff` namespace |
| `chef/page.tsx` | Share order board component (P1) |

---

## Permission changes

| Area | Current | Target |
|------|---------|--------|
| Orders controller | Firebase only | Firebase + RolesGuard + restaurant membership check in service |
| Reservations | Roles commented | Enable WAITSTAFF on read/update/check-in |
| Table sessions | N/A | WAITSTAFF + MANAGER |
| Tables write | Unguarded | **No change** (manager only via existing routes) |
| Guest search | N/A | WAITSTAFF + MANAGER |

**New helper (recommended):** `RestaurantAccessService.assertStaff(userUid, restaurantId, roles[])`

---

## Realtime changes

| Item | Effort |
|------|--------|
| `join:restaurant` / `leave:restaurant` handlers | S |
| Emit `order:updated` on order events (already have listener — add gateway emit) | S |
| Emit `floor:updated` on session changes | S |
| Emit `reservation:updated` on reservation changes | S |
| `useRestaurantSocket` hook + query invalidation | M |
| Chef + waitstaff + manager layouts join room | S |

---

## Priority-ranked implementation order

### Sprint 1 — Foundation (P0)

1. Auth/routing fixes (`persistAuthSession`, middleware review)
2. `TableSession` entity + migration + service + controller
3. Orders RBAC + `waitstaffId` + `mark-served` / `send-to-kitchen`
4. Reservation status extension + check-in/no-show
5. Waitstaff layout + nav + overview (read-only aggregation)
6. Floor view (read) + table detail

### Sprint 2 — Core FOH actions (P0/P1)

7. Seat walk-in + reservation check-in flows
8. Order board + detail + send to kitchen + mark served
9. Create order + menu picker
10. Notifications + chat routes on waitstaff shell
11. Restaurant socket room + live invalidation

### Sprint 3 — Polish (P1/P2)

12. Request bill + cash payment
13. Guest search/profile
14. Transfer table + notes
15. Shift summary
16. Chef shared board upgrade
17. i18n pass + tablet QA

---

## Effort estimate (rough)

| Sprint | Backend | Frontend | Total |
|--------|---------|----------|-------|
| 1 | 3–4 d | 3–4 d | ~1.5 w |
| 2 | 2–3 d | 4–5 d | ~1.5 w |
| 3 | 2 d | 3 d | ~1 w |
| **Total** | **7–9 d** | **10–12 d** | **~4 w** |

---

## Dependencies

- Chat migration on production (already tracked in `TASKS.md`)
- `NEXT_PUBLIC_WS_URL` points to API host
- No change to frozen `tables.controller.ts` without explicit approval

---

## Verification mapping (Phase 7 preview)

Each acceptance criterion in `SPEC.md` maps to Sprint 1–3 deliverables. Regression suite must include customer checkout, manager orders, delivery driver chat, and chef page before marking complete.
