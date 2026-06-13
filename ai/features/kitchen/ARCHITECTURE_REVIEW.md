# Kitchen Dashboard — Architecture Review

**Date:** 2026-06-11  
**Inputs:** [SPEC.md](./SPEC.md), [GAP_ANALYSIS.md](./GAP_ANALYSIS.md), [ARCHITECTURE.md](../../ARCHITECTURE.md)  
**Verdict:** **Approved to implement** in phases — reuse orders + WS + chat; add kitchen module

---

## Alignment with CuisineEase architecture

| Principle | Kitchen adherence |
|-----------|-------------------|
| Two-repo layout | `app/` frontend, `backend/` API |
| NestJS domain modules | New `kitchen/` module; extend `orders/` |
| TypeORM + migrations | New entities for stations, events |
| FirebaseGuard global | All new routes secured |
| RestaurantAccessService | `assertChefOrAbove`, `assertManagerOrAbove` for config |
| Event emitter | Order status → notifications + WS + KDS service |
| Frontend App Router | `/chef/*` mirrors `/waitstaff/*` |
| TanStack Query + services | `chef.ts` service layer |
| Socket.io on API host | Reuse `join:restaurant` |
| i18n | New `chef` namespace |

---

## Module design

```
┌──────────────────────────────────────────────────────────────┐
│                  Kitchen Portal (new shell)                   │
├────────────┬─────────────┬──────────────┬────────────────────┤
│ Kitchen    │ Orders      │ Chat (ext)   │ Notifications      │
│ Module     │ kitchen-    │ station      │ kitchen alert      │
│ (new)      │ status      │ groups E2EE  │ types              │
├────────────┴──────┬──────┴──────┬───────┴────────────────────┤
│ Stations config   │ Menu read   │ Inventory read             │
│ (manager write)   │ existing    │ existing                   │
└───────────────────┴─────────────┴────────────────────────────┘
```

### Backend: proposed `kitchen/` module

```
backend/src/kitchen/
├── kitchen.module.ts
├── entities/
│   ├── kitchen-station.entity.ts
│   ├── order-kitchen-event.entity.ts
│   └── menu-item-station.entity.ts
├── dto/
├── kitchen-stations.service.ts
├── kitchen-queue.service.ts      # aggregated queue DTOs
├── kitchen-analytics.service.ts
├── kitchen.controller.ts           # stations, history, analytics
└── kitchen.gateway.ts            # optional: kitchen-specific events
```

**Do not duplicate order state.** `Order.status` remains source of truth; kitchen events are audit/timeline only.

### Frontend structure

```
app/src/app/chef/
├── layout.tsx                 # Sidebar, top nav, WS, notifications
├── page.tsx                   # Dashboard (replace current)
├── queue/page.tsx
├── stations/page.tsx
├── history/page.tsx
├── messages/page.tsx
├── analytics/page.tsx
├── inventory/page.tsx
├── settings/page.tsx
└── profile/page.tsx

app/src/config/chefNav.ts
app/src/services/kitchen.ts
app/src/hooks/useKitchenQueue.ts
app/src/components/Kitchen/
```

---

## Realtime strategy

| Event | Payload | Subscribers |
|-------|---------|-------------|
| `order:updated` | `{ orderId, restaurantId }` | Chef queue, waitstaff floor |
| `kitchen:station:updated` | `{ stationId }` | Chef stations view |
| `kitchen:alert` | `{ type, orderId, priority }` | Chef + optional manager |
| `chat:message:new` | existing | Messages |

Chef layout: emit `join:restaurant` on mount (same as waitstaff).

---

## E2EE messaging architecture (P3)

Separate from operational chat:

1. Client generates device key pair (Web Crypto API)
2. Public keys registered via `POST /user/device-keys`
3. DM send: encrypt body client-side → store `{ ciphertext, nonce, senderDeviceId }`
4. Server never stores plaintext for E2EE threads
5. Manager audit channels use standard `chat/` (no E2EE) with type `staff_operational`

Document in [SECURITY_REVIEW.md](./SECURITY_REVIEW.md); ADR required in `DECISIONS.md` before P3 start.

---

## Offline architecture (P3)

- **IndexedDB** store: `kitchen-offline-queue` (actions), `kitchen-queue-cache`
- Service worker optional (PWA) — not required for MVP
- Replay endpoint: `POST /kitchen/sync` with batched actions + client sequence
- Server validates order still in valid state transition

---

## Manager ↔ kitchen boundary

| Action | Manager | Chef |
|--------|---------|------|
| Configure stations | write | read |
| View queue | read | read/write status |
| Intervene priority | optional permission | — |
| Analytics | read | read (own kitchen) |
| Inventory | write | read |

---

## Migration from current `/chef/page.tsx`

1. Move queue logic to `/chef/queue`
2. Dashboard at `/chef` shows aggregates
3. Deprecate inline page — no breaking URL if `/chef` redirects to dashboard with queue CTA

---

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Scope explosion | Strict phased roadmap; MVP = queue + WS + i18n |
| E2EE complexity | Defer to P3; use standard chat for MVP |
| Performance at high volume | Virtualized list; compact mode; pagination |
| Duplicate order logic | Single `OrdersService.updateKitchenStatus` |

---

## Approval

Architecture aligns with existing waitstaff implementation patterns. **v2 roadmap:** [IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md) (K0–K12). Critical review: [ENTERPRISE_REVIEW.md](./ENTERPRISE_REVIEW.md).
