# Kitchen Dashboard — Current State Audit

**Date:** 2026-06-11  
**Scope:** `/chef` portal + backend kitchen support  
**Baseline for:** [GAP_ANALYSIS.md](./GAP_ANALYSIS.md)

---

## Executive summary

| Layer | State |
|-------|-------|
| Frontend portal | **~5%** — single page, no layout shell, no nav |
| Backend KDS | **~2%** — stub service with TODOs |
| Realtime | **Partial** — `order:updated` exists; chef UI does not subscribe |
| RBAC | **Partial** — `kitchen-status` endpoint guarded; list uses generic orders API |
| i18n | **Partial** — borrows `manager:chefDashboard` keys; hardcoded English strings |
| Integration | **Minimal** — waitstaff send-to-kitchen works; chef UI polls via React Query only |

---

## Frontend inventory

### Routes

| Route | File | Purpose |
|-------|------|---------|
| `/chef` | `app/src/app/chef/page.tsx` | Only route — kitchen display |

**Missing:** layout.tsx, chefNav, sub-routes (queue, stations, history, messages, analytics, inventory, settings, profile).

### Current page behavior

- Loads orders with `status: sent_to_kitchen` and `status: preparing` via `useOrdersList`
- Displays two static sections: "Incoming" and "In preparation"
- Actions: `advanceKitchenOrderStatus(id, 'preparing' | 'ready')`
- Hardcoded strings: "Incoming", "In preparation", "Start preparing", "Mark ready", "No new tickets."
- No WebSocket subscription
- No restaurant floor context (table session, allergies, items detail)
- No responsive shell / sidebar / mobile nav
- Uses `getWaitstaffOrderStatusLabel` for display (acceptable interim)

### Auth & routing

- `middleware.ts` — `chef` role locked to `/chef/*`
- `authRouting.ts` — `chef: "/chef"`
- No `ChefLayout` with notification/chat hooks (unlike waitstaff/manager)

---

## Backend inventory

### KitchenDisplayService

Path: `backend/src/orders/services/kitchen-display.service.ts`

- `updateOrderDisplay(order)` — logs only, TODO KDS integration
- `removeFromDisplay(orderId)` — logs only

### Order kitchen endpoints (existing)

| Endpoint | Purpose |
|----------|---------|
| `POST /orders/:id/kitchen-status` | Chef advances preparing / ready |
| `POST /orders/:id/send-to-kitchen` | Waitstaff (not chef) |
| `GET /orders` | List with `statuses`, `restaurantId` |

Chef role included in `menu:read` for menu reference.

### Missing backend domains (for full spec)

- Kitchen stations entity + assignment
- Ticket timeline / audit log API
- Kitchen-specific analytics aggregation
- E2EE message key management
- Offline action replay endpoint
- Station workload metrics
- Chef presence / online status

---

## Reusable assets

| Asset | Location | Kitchen use |
|-------|----------|-------------|
| Order hooks | `useOrdersList`, `useOrder` | Queue data |
| Status advance | `advanceKitchenOrderStatus` in `order.ts` | Core actions |
| Order status views | `orderStatusViews.ts` | Labels (add chef-specific layer) |
| WS gateway | `notifications.gateway.ts` | `join:restaurant`, `order:updated` |
| Chat module | `chat/` | Extend for station groups + E2EE |
| Notifications | `notifications/` | Smart kitchen alerts |
| Inventory | `inventory/` | Read-only chef view |
| Waitstaff layout patterns | `app/waitstaff/*` | Shell template |
| Manager analytics UI patterns | OverviewPage | Analytics charts |

---

## Risks if shipping current UI as "complete"

1. Chefs cannot see line items, modifiers, or allergies on tickets
2. No realtime — stale queue during rush
3. No station routing — single flat list does not scale
4. No communication with waitstaff from kitchen UI
5. Placeholder feel vs manager/customer portals
6. Hardcoded English breaks i18n policy

---

## Sign-off

Audit confirms kitchen portal requires **full greenfield build** per [SPEC.md](./SPEC.md), reusing order + WS + chat foundations.
