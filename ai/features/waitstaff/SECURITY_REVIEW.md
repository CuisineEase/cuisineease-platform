# Waitstaff Portal — Security Review

**Date:** 2026-06-11  
**Scope:** Phase 8 backend RBAC, floor/session APIs, orders access control

---

## Summary

| Area | Before Phase 8 | After Phase 8 | Risk |
|------|----------------|---------------|------|
| Order list (`GET /orders`) | Any authenticated user could query any restaurant | Restaurant queries require staff role; default scopes to own orders | Mitigated |
| Order detail (`GET /orders/:id`) | Open to any authenticated UUID holder | Staff of restaurant OR order customer | Mitigated |
| Order update (`PATCH /orders/:id`) | Unrestricted | Draft: waitstaff+ only; other statuses: staff or customer access check | Improved |
| Order delete | Unrestricted | Staff or customer with access | Improved |
| Staff order actions | Guarded per endpoint | Unchanged — still guarded | OK |
| Table sessions | `RestaurantAccessService` on all routes | Unchanged | OK |
| Floor / billing / timeline | N/A | `assertWaitstaffOrAbove` on every route | OK |
| WebSocket `join:restaurant` | Added in Phase 5 | Validates `UserRestaurantRole` | OK |
| Tables CRUD (`/tables`) | **Unguarded** (frozen) | **Not modified** | High (pre-existing) |
| Generic `POST /orders` | Unscoped create | **Not modified** — legacy/customer path | Medium |

---

## Authentication

- All waitstaff routes require Firebase JWT (`APP_GUARD` + route-level guards).
- Waitstaff routing depends on JWT `role: waitstaff` from `syncFirebaseRestaurantClaims`.
- `persistAuthSession()` no longer defaults unknown roles to `manager`.

**Recommendation:** After staff invite, force token refresh before first portal use.

---

## Authorization model

### Restaurant-scoped (floor module)

Enforced via `RestaurantAccessService`:

| Role rank | Can access |
|-----------|------------|
| WAITSTAFF+ | Floor, sessions, guests, billing, timeline, payments |
| CHEF+ | Floor read, kitchen status |
| MANAGER+ | Full override (future explicit force-close) |

### Orders

```
findAll:
  restaurantId → assertWaitstaffOrAbove
  userId → must match caller
  neither → defaults to caller's userId

findOne / delete:
  staff of order.restaurant OR order.customer

update:
  DRAFT → updateDraftByStaff (waitstaff+)
  else → assertCanAccessOrder then update (manager path via rank)
```

### Gaps remaining

1. **`POST /orders`** — Still allows any authenticated user to create orders if they know `customerId` and `restaurantId`. Mitigation: customer flows only; monitor abuse; future: restrict to customer self or staff endpoints only.

2. **`tables.controller.ts`** — Write endpoints without guards. Waitstaff does not use these; managers do. **Do not expose table write APIs publicly** until guards restored.

3. **Manager override on close** — `close(force)` exists in service but not exposed; managers cannot force-close unpaid sessions via API yet.

4. **Payment PENDING** — Non-cash payments do not auto-complete; staff could record duplicate pending payments. UI should prefer cash for in-person settlement until gateway integration.

---

## Data isolation

- Session queries always filter by `restaurantId` from path or session entity.
- `assertNoActiveSession` prevents double-booking tables.
- Order session payments validate `orderId` belongs to session when provided.

---

## Realtime security

- `join:restaurant` requires membership in `UserRestaurantRole` for that restaurant.
- Auto-join on connect uses JWT `restaurantId` claim only (no cross-restaurant leak).
- Floor events emitted only to `restaurant:{id}` room.

---

## Notifications

- Bill request notifies all restaurant staff IDs (including waitstaff). Acceptable for MVP; may reduce noise later with role targeting.

---

## Audit recommendations

| Priority | Action |
|----------|--------|
| P1 | Restore guards on `tables.controller.ts` in dedicated security sprint |
| P1 | Restrict `POST /orders` to customer-self or `/orders/staff` only |
| P2 | Add integration tests for cross-restaurant denial |
| P2 | Rate-limit payment recording per session |
| P3 | Manager `POST /table-sessions/:id/close?force=true` with audit log |

---

## Verdict

Waitstaff-specific surfaces are **adequately protected** for production MVP. Platform-wide order create and table write endpoints remain the primary pre-existing risks and are documented, not introduced by this phase.
