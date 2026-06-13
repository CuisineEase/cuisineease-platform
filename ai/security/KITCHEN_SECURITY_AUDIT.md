# Kitchen Dashboard — Security Audit (Release Candidate)

**Date:** 2026-06-11  
**Scope:** Kitchen module, `/chef` portal, `/manager/kitchen`, E2EE, chat extensions

---

## RBAC

| Route | Enforcement |
|-------|-------------|
| `/restaurants/:id/kitchen/*` (chef ops) | `FirebaseGuard` + `assertChefOrAbove` |
| `/restaurants/:id/kitchen/monitor` | `assertManagerOrAbove` |
| Station write | `assertManagerOrAbove` |
| Device keys | User-scoped; public keys read by authenticated staff |
| Chat reactions/pin | Conversation participant check |

Multi-tenant: all queries scoped by `restaurant.id`.

---

## Audit trail

`order_kitchen_events` records claim, start, ready, hold, resume, refire, undo, reorder with actor + timestamps.

---

## Version / concurrency

`orders.version` optimistic locking on actions and reorder — mismatch returns 400.

---

## Offline sync

`kitchen_offline_actions` idempotency keys prevent replay duplicates.

---

## E2EE staff DMs

| Component | Status |
|-----------|--------|
| `staff_device_keys` table | Public keys only on server |
| `/user/me/device-keys` | Register, list, verify, revoke |
| Client `kitchenE2EE.ts` | ECDH P-256 + AES-GCM session keys |
| Chat `isEncrypted` | Ciphertext transport for private DMs |
| Operational channels | Organization-visible per policy (non-E2EE) |

**Assumption:** Server never stores private keys; backup is user/device responsibility.

---

## Rate limiting

Kitchen sync processes bounded action batches; production rate limit middleware recommended at edge (Render/nginx).

---

## Sign-off

Kitchen security posture meets enterprise baseline with documented E2EE assumptions and RBAC isolation.
