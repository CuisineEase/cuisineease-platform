# Kitchen Dashboard — Security Review

**Date:** 2026-06-11  
**Scope:** Planned kitchen portal per [SPEC.md](./SPEC.md)  
**Status:** Pre-implementation checklist

---

## RBAC matrix (target)

| Resource | chef | waitstaff | manager | owner | admin |
|----------|------|-----------|---------|-------|-------|
| Kitchen queue read | ✅ own restaurant | ❌ | ✅ read | ✅ | ✅ |
| `kitchen-status` advance | ✅ | ❌ | ⚠️ intervene only | ✅ | ✅ |
| Stations config | read | read | write | write | ✅ |
| Kitchen analytics | ✅ | ❌ | ✅ | ✅ | ✅ |
| Ticket history | ✅ | ❌ | ✅ | ✅ | ✅ |
| Inventory (kitchen view) | read | read | write | write | ✅ |
| E2EE private DM | ✅ participants | ✅ participants | ❌ decrypt | ❌ decrypt | ❌ decrypt |
| Operational staff chat | ✅ | ✅ | ✅ | ✅ | ✅ |

Enforcement: `RestaurantAccessService.assertChefOrAbove` + `assertRestaurantIdMatch` on every path/body `restaurantId`.

---

## Multi-tenant isolation

- All queries filter by `restaurantId` from JWT role assignment
- WS `join:restaurant` must verify `user_restaurant_roles` (existing waitstaff pattern)
- Order ids validated: `order.restaurantId === caller restaurant`
- Cross-restaurant ticket access → 403 + security log

---

## Audit requirements

Log to `order_kitchen_events` (or audit table):

| Action | Fields |
|--------|--------|
| Status change | orderId, from, to, actorId, timestamp |
| Station assign | orderId, stationId, actorId |
| Handoff | orderId, fromChefId, toChefId |
| Hold / resume | orderId, reason |
| Manager intervene | orderId, action, reason |

E2EE messages: log metadata only (conversationId, senderId, timestamp) — **not** plaintext.

---

## E2EE private messaging (P3)

| Requirement | Implementation |
|-------------|----------------|
| Confidentiality | AES-GCM or libsodium sealed boxes client-side |
| Server role | Ciphertext transport + public key directory |
| Key management | Per-device keys; rotation on logout device |
| Device verification | Safety number / QR for high-risk users |
| Admin access | **No** decryption of E2EE threads |
| Operational audit | Separate non-E2EE `staff_operational` channel type |
| Recovery | Lost device = lost history (document UX) |

ADR required before implementation — see `ai/DECISIONS.md` (proposed ADR-015).

---

## Offline sync security

- Signed action tokens (HMAC with session secret) or JWT-scoped nonce
- Replay window: reject actions older than 5 minutes unless idempotent
- Idempotency key per client action id
- Validate state machine server-side — never trust client status alone

---

## Threat model (summary)

| Threat | Control |
|--------|---------|
| Privilege escalation | Server RBAC; ignore client role |
| Cross-tenant data leak | restaurantId on every query |
| WS room snooping | Verified join:restaurant |
| Message interception (E2EE) | End-to-end encryption for DMs |
| Replay attacks | Nonce + timestamp on offline sync |
| Alert fatigue social engineering | Priority categories + manager escalation |

---

## Pre-ship checklist

- [ ] All `/chef/*` API routes have integration tests for RBAC
- [ ] Chef cannot PATCH user identity fields (profile API scoped)
- [ ] Chef cannot access manager-only inventory writes
- [ ] WS events contain IDs only — no PII in payload
- [ ] Security audit doc updated post-implementation (mirror waitstaff audit)
- [ ] E2EE penetration review before P3 launch

---

## Related

- Waitstaff audit: [../../security/WAITSTAFF_SECURITY_AUDIT.md](../../security/WAITSTAFF_SECURITY_AUDIT.md)
- Backlog: [../../SECURITY_BACKLOG.md](../../SECURITY_BACKLOG.md)
