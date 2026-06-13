# CuisineEase — Security Backlog

**Last updated:** 2026-06-11  
**Related:** `ai/security/WAITSTAFF_SECURITY_AUDIT.md`, ADR-013, ADR-014

Non-blocking and deferred security items after the Waitstaff production hardening pass.

---

## Open — HIGH

### SEC-001 — Unguarded table write endpoints

**File:** `backend/src/tables/tables.controller.ts` (frozen per ADR-013)  
**Risk:** Any authenticated (or in some cases unauthenticated-adjacent) caller may create/update/delete tables and sections across restaurants if they discover endpoint paths and UUIDs.  
**Impact:** Cross-tenant floor configuration tampering; not direct PII leak but breaks multi-tenant integrity.  
**Mitigation today:** Waitstaff reads use secured `/restaurants/:id/floor`; dine-in writes go through `table-sessions`.  
**Recommended fix:** Add parallel secured routes (e.g. `manager/tables`) with `RestaurantAccessService.assertManagerOrAbove`, deprecate frozen writes, or amend ADR-013 to allow guard restoration behind feature flag.  
**Blocked by:** ADR-013 “do not modify frozen tables.controller.ts”

---

## Open — MEDIUM

### SEC-002 — Customer order status via generic PATCH

**Endpoint:** `PATCH /orders/:id`  
**Risk:** Customer who owns an order could transition `draft` → `sent_to_kitchen` via generic transition matrix instead of staff-only workflow.  
**Mitigation:** Dine-in orders are created by staff (`POST /orders/staff`); customer-owned drafts are mostly delivery/cart flows.  
**Fix:** Restrict customer PATCH to cancel-only or delivery-specific transitions.

### SEC-003 — Session `assignedWaitstaffId` not validated against restaurant role

**Endpoint:** `PATCH /table-sessions/:id`  
**Risk:** Manager could assign arbitrary user UUID as waitstaff.  
**Fix:** Verify assignee has `WAITSTAFF` (or above) role for session restaurant.

### SEC-004 — WebSocket JWT `restaurantId` claim auto-join

**File:** `notifications.gateway.ts` `handleConnection`  
**Risk:** Stale or incorrect custom claim could join wrong room before `join:restaurant` re-validates.  
**Mitigation:** Floor events are ID-only; REST fetches are secured.  
**Fix:** Remove auto-join from JWT claim; require explicit `join:restaurant` only.

### SEC-005 — Reservation table availability lacks restaurant param

**Endpoint:** `GET /reservations/table/:tableId/availability`  
**Risk:** Table UUID enumeration reveals availability (not guest PII).  
**Fix:** Require `restaurantId` query param and validate table belongs to restaurant.

---

## Open — LOW

### SEC-006 — Global notification socket rooms

All connected clients join `notifications:general`, `notifications:orders`, etc. Review whether any payload in those channels could leak cross-tenant metadata.

### SEC-007 — `ping` WebSocket handler uses `server.emit`

Broadcasts to all sockets; replace with `client.emit` or remove in production.

---

## Resolved (2026-06-11 waitstaff hardening)

| ID | Summary |
|----|---------|
| — | `POST /orders` open create → `createSecured` |
| — | `PATCH /orders/:id` RBAC bypass → `updateSecured` |
| — | Payments GET/list unscoped → secured accessors |
| — | Notifications cross-user access → self/admin guards |
| — | Reservations restaurant list/update unguarded → `RestaurantAccessService` |
| — | Guests broken `RolesGuard` param → removed; path RBAC |
| — | WebSocket restaurant room leak on switch → leave previous room |
| — | Guest body/path `restaurantId` mismatch → validated |
