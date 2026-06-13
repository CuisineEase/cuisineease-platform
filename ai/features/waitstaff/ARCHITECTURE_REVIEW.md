# Waitstaff Portal — Phase 4 Architecture Review

**Date:** 2026-06-11  
**Inputs:** `ARCHITECTURE.md`, `AUDIT.md`, `SPEC.md`, `GAP_ANALYSIS.md`  
**Verdict:** **Approved to implement** after spec review, with mitigations below

---

## Alignment with CuisineEase architecture

### Matches

| Principle | How waitstaff spec adheres |
|-----------|---------------------------|
| Two-repo layout | Frontend `app/`, backend `backend/` — no monorepo coupling |
| NestJS modules per domain | New `table-sessions` module; extend `orders`, `reservations` |
| TypeORM entities + migrations | `TableSession` via standard migration path |
| DTO + class-validator | All new endpoints follow existing DTO pattern |
| `apiSuccess` envelope | Floor/guest endpoints return consistent shape |
| Firebase global guard | `APP_GUARD` FirebaseGuard unchanged |
| `RolesGuard` + `RestaurantRole` | Waitstaff permissions use existing guard |
| Event emitter for side effects | Order/reservation listeners extend for notifications + WS |
| Frontend App Router by role | `/waitstaff/*` mirrors `/manager/*` |
| TanStack Query + thin services | New hooks wrap new API clients |
| shadcn/ui + Tailwind | Reuse `components/ui` and manager patterns |
| Socket.io on API host | Restaurant rooms extend existing gateway pattern |
| Chat/notifications reuse | No fork — mount existing components in waitstaff layout |
| UUID for Postgres APIs | `getUuidBackedUserId` in layout |

### Extends (does not replace)

- **Orders** remain system of record for kitchen and billing.
- **Tables** remain configuration (capacity, section, pricing).
- **TableSession** holds ephemeral FOH state — separation matches reservation vs table config split.
- **Chef portal** consumes same order events; shared `StaffOrderBoard` component avoids duplicate UI logic.

---

## Reuse map

```
┌─────────────────────────────────────────────────────────────┐
│                     Waitstaff Portal (new shell)             │
├─────────────┬───────────────┬──────────────┬────────────────┤
│ TableSession│ Orders (ext)  │ Reservations│ Chat/Notif      │
│   (new)     │ waitstaffId   │ (ext status)│ (existing)      │
├─────────────┴───────┬───────┴──────┬───────┴────────────────┤
│       Table (read)  │  Menu (read) │  User (guest search)  │
│       existing      │  existing    │  existing             │
└─────────────────────┴──────────────┴───────────────────────┘
```

### Entities — no parallel models

| Spec concept | Entity | Action |
|--------------|--------|--------|
| Table config | `Table` | Read only |
| Table occupancy | `TableSession` | **New** |
| Order | `Order` | Extend |
| Reservation | `Reservation` | Extend enum |
| Guest | `User` | Read; optional profile fields later |
| Staff assignment | `UserRestaurantRole` | Existing |
| Payment | `Payment` + `Order.paymentStatus` | Extend for cash |
| Task | `Task` | **Do not reuse** (unrelated scaffold) |

### DTO patterns

Follow `CreateOrderDto`, `FindAllOrdersDto`, `CreateReservationDto` conventions:

- UUID fields with `@IsUUID()`
- Optional fields with `@IsOptional()`
- Swagger `@ApiProperty` on all new DTOs
- Update DTOs via `PartialType` where appropriate

### UI patterns

| Pattern | Source to copy |
|---------|----------------|
| Sidebar layout | `manager/layout.tsx` |
| Nav config | `config/managerNav.ts` → `waitstaffNav.ts` |
| Data tables | `manager/reservations/page.tsx` |
| Order tabs | `ManagerOrdersContent.tsx` |
| Chat master-detail | `ChatInbox` mobile behavior |
| Status badges | Order status styles in manager orders |
| Error/toast | `sonner` + `getApiErrorMessage` |

---

## Risks and mitigations

### R1 — Frozen `tables.controller.ts` (no guards)

**Risk:** Accidental modification breaks production table management or customer dine-in picker.

**Mitigation:**

- Do **not** edit `tables.controller.ts` in waitstaff work.
- All FOH state in `TableSession` module.
- Document in `DECISIONS.md` as Accepted constraint.

**Severity:** High if violated · **Likelihood:** Medium

---

### R2 — Orders RBAC enforcement breaks anonymous/internal callers

**Risk:** Orders endpoints today allow any authenticated user; tightening may break scripts or tests.

**Mitigation:**

- Add service-layer `assertCanAccessOrder(user, order)` rather than only controller guard.
- Audit `orders.controller.ts` `from-cart` path separately (customer).
- Integration tests for waitstaff, manager, customer, cross-restaurant denial.

**Severity:** High · **Likelihood:** Medium

---

### R3 — JWT role / middleware misrouting

**Risk:** `persistAuthSession` defaults to `manager` when role missing; stale tokens route waitstaff incorrectly.

**Mitigation:**

- Fix fallback: use JWT-decoded role only; never assume manager.
- Middleware: prefer explicit `role` claim over `restaurantId` → manager mapping when role is `waitstaff` or `chef`.
- Force token refresh after staff invite (already partial in login flow).

**Severity:** High · **Likelihood:** Medium (seen in chat bugs)

---

### R4 — Duplicate business logic between manager and waitstaff

**Risk:** Copy-paste order/reservation UIs diverge over time.

**Mitigation:**

- Extract shared components to `components/RestaurantOperations/` (orders, reservations).
- Manager retains superset permissions; waitstaff uses same components with `mode="waitstaff"` prop or permission hook `useRestaurantPermissions()`.

**Severity:** Medium · **Likelihood:** High without refactor plan

---

### R5 — Realtime room authorization

**Risk:** Client joins `restaurant:{id}` for restaurant they don't belong to.

**Mitigation:**

- Server validates `UserRestaurantRole` on `join:restaurant`.
- Reject join if no membership; log attempts.

**Severity:** Medium · **Likelihood:** Low

---

### R6 — Reservation status migration

**Risk:** PostgreSQL enum alterations fail on production if not idempotent.

**Mitigation:**

- Use TypeORM migration with `ADD VALUE IF NOT EXISTS` pattern (match existing migrations).
- Backfill: existing `confirmed` stays valid; new transitions explicit.

**Severity:** Medium · **Likelihood:** Low

---

### R7 — Cash payment without audit trail

**Risk:** Staff marks paid without financial record.

**Mitigation:**

- Create `Payment` row with `method: CASH`, `recordedBy: waitstaffId`.
- Manager reports can reconcile later.

**Severity:** Medium · **Likelihood:** Medium (operational)

---

### R8 — Walk-in guest without User record

**Risk:** `CreateOrderDto` requires `customerId`.

**Mitigation (MVP):**

- Require phone lookup first; manager can register walk-in.
- P2: `POST /guests/walk-in` creates minimal `User` with consent flag.

**Severity:** Medium · **Likelihood:** High for real restaurants

---

### R9 — Chef + waitstaff status transition conflicts

**Risk:** Both can advance order status incorrectly.

**Mitigation:**

- Enforce existing `validateStatusTransition` matrix.
- Chef: `preparing → ready`. Waitstaff: `ready → delivered` (dine-in served). Waitstaff: `pending → preparing` only if spec allows (or chef/manager only for preparing — **clarify in spec review**).

**Severity:** Medium · **Likelihood:** Medium

**Recommendation:** Waitstaff may `pending → preparing` (send to kitchen); chef owns `preparing → ready`.

---

### R10 — Performance on floor view

**Risk:** N+1 queries for tables × sessions × orders.

**Mitigation:**

- Single `GET /restaurants/:id/floor` aggregated query with joins.
- Cache 5s staleTime on client; invalidate on socket event.

**Severity:** Low · **Likelihood:** Medium at scale

---

## Proposed module structure (backend)

```
backend/src/
  table-sessions/
    entities/table-session.entity.ts
    dto/
    table-sessions.service.ts
    table-sessions.controller.ts
    table-sessions.module.ts
  orders/                    # extend only
  reservations/              # extend only
  floor/
    floor.gateway.ts         # optional: restaurant WS room
    floor.module.ts
```

Alternatively, colocate gateway in `notifications/gateway` if that's where socket lives today — **inspect at implementation** and follow existing gateway registration in `app.module.ts`.

---

## Proposed frontend structure

```
app/src/
  app/waitstaff/
    layout.tsx
    page.tsx
    tables/...
    orders/...
    reservations/...
    guests/...
    chat/...
    notifications/...
    shift/...
  components/
    Waitstaff/              # waitstaff-specific composition
    RestaurantOperations/   # shared manager/waitstaff (extract)
  config/waitstaffNav.ts
  hooks/useFloor.ts
  hooks/useRestaurantSocket.ts
  services/tableSession.ts
```

---

## Architecture decisions to record in `DECISIONS.md` (on approval)

1. **TableSession entity** for FOH state — do not add status to `Table`.
2. **Do not modify** `tables.controller.ts` without dedicated security project.
3. **Waitstaff served** = order status `delivered` for dine-in (reuse enum, no `served` status).
4. **Restaurant socket room** pattern for floor/order live updates.
5. **Shared RestaurantOperations components** between manager and waitstaff.

---

## Pre-implementation checklist

- [ ] Spec reviewed and open questions answered
- [ ] `DECISIONS.md` updated with ADRs above
- [ ] `TASKS.md` updated with Sprint 1 items
- [ ] Migration strategy agreed for production
- [ ] Chef status ownership confirmed (R9)

---

## Conclusion

The proposed waitstaff portal **fits CuisineEase architecture**: it extends existing domains, reuses auth/notifications/chat, and introduces one focused new entity (`TableSession`) rather than parallel systems. Primary risks are **permissions tightening on orders**, **auth routing edge cases**, and **UI duplication** — all mitigable with the strategies above.

**Do not start Phase 5** until spec open questions are resolved and this review is acknowledged.
