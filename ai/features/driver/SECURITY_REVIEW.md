# Driver Dashboard — Security Review

**Date:** 2026-06-11  
**Status:** Pre-release — critical issues addressed

---

## Restaurant isolation ✅

- All driver routes require `restaurantId` path param
- `assertDeliveryOrAbove` validates restaurant role before data access
- Assignments filtered by `order.restaurant.id` + `deliveryPerson.id`

---

## RBAC ✅

| Action | Guard |
|--------|-------|
| List deliveries (generic) | Driver sees own only |
| Driver dashboard API | `assertDeliveryOrAbove` |
| Manager assign driver | `RestaurantRole.MANAGER` on `/delivery` POST/PATCH |
| Customer view | Own order only |

---

## Ownership ✅

- `getOwnedDelivery()` throws `ForbiddenException` if not assigned driver
- Generic PATCH blocked for drivers on unassigned deliveries
- Drivers cannot change `status` via generic PATCH (manager-only status override)

---

## JWT / WebSocket ✅

- Firebase JWT required (existing `FirebaseGuard`)
- Drivers join `user:{id}` and `delivery` rooms — no cross-user data in room payloads without assignment check on REST

---

## Proof integrity 🟡

- OTP stored in DB (not hashed) — acceptable for short-lived delivery codes; consider hashing for PCI-adjacent deployments
- Photo URLs stored as strings — must be validated storage URLs in future
- Timeline in `delivery_events` provides audit trail

---

## Privacy ✅

- Customer phone/email exposed only to assigned driver on detail endpoint
- Profile page read-only for PII; managers control identity

---

## Rate limiting ⬜

- No driver-specific rate limits yet — inherits global API limits
- Recommend throttle on `/driver/sync` and status transitions

---

## Remaining actions

1. Apply migration on production before enabling driver portal
2. Add integration test: driver A cannot PATCH driver B's delivery
3. Sanitize `proofPhotoUrl` to same-origin or Firebase storage domains
