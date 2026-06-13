# Driver Dashboard — Current State Audit

**Date:** 2026-06-11  
**Scope:** `/delivery` portal + backend delivery/driver modules  
**Baseline for:** [GAP_ANALYSIS.md](./GAP_ANALYSIS.md)

---

## Executive summary

| Layer | Before SDD | After implementation |
|-------|------------|----------------------|
| Frontend portal | **~5%** — single list page + chats | **~90%** — full layout, 12 routes, i18n EN/FR |
| Backend driver API | **0%** — generic `/delivery` only | **~85%** — `DriverModule`, scoped RBAC, shift, proof |
| Realtime | **Partial** — `delivery` WS room only | **Implemented** — `delivery:*`, `driver:shift:updated` |
| RBAC | **Broken** — drivers saw all deliveries | **Fixed** — driver-scoped list + ownership on PATCH |
| i18n | **Partial** — borrowed `manager:` keys | **Implemented** — `delivery:` namespace EN/FR |
| Offline | **None** | **Partial** — sync endpoint + connection banner |

---

## Frontend inventory (pre-SDD)

| Route | File | Purpose |
|-------|------|---------|
| `/delivery` | `app/src/app/delivery/page.tsx` | Minimal delivery list |
| `/delivery/chats` | `app/src/app/delivery/chats/page.tsx` | Chat inbox (no layout) |

**Missing (now implemented):** layout, `deliveryNav`, dashboard metrics, active workspace, shift, performance, history, notifications, profile, settings, help, map workspace.

---

## Backend inventory (pre-SDD)

### Delivery module

| Endpoint | Issue |
|----------|-------|
| `GET /delivery` | No filter by assigned driver for `SystemRole.DELIVERY` |
| `PATCH /delivery/:id` | No ownership check; status not auto-set on assign |
| Entity | No proof fields, shift, or audit timeline |

### Chat integration

- `customer_driver` conversations auto-open on assignment (`ChatService.onDeliveryUpdated`)
- Driver role included in chat participant resolution

### WebSocket

- Drivers join `delivery` room on connect
- No delivery-specific event taxonomy

---

## Post-implementation inventory

### Routes (`/delivery/*`)

| Route | Purpose |
|-------|---------|
| `/delivery` | Dashboard — metrics, active assignments |
| `/delivery/deliveries` | Searchable assignment list |
| `/delivery/active/[id]` | Full delivery workspace |
| `/delivery/history` | Completed/failed archive |
| `/delivery/performance` | Driver KPIs |
| `/delivery/shift` | Clock in/out, break, availability |
| `/delivery/messages` | ChatInbox (replaces `/delivery/chats`) |
| `/delivery/notifications` | NotificationInbox |
| `/delivery/map` | Navigation links per assignment |
| `/delivery/activity` | Recent delivery feed |
| `/delivery/profile` | Read-only identity (manager-controlled) |
| `/delivery/settings` | Local preferences |
| `/delivery/help` | FAQ accordion |

### Backend (`DriverModule`)

| Endpoint | Purpose |
|----------|---------|
| `GET .../driver/dashboard` | Summary metrics + shift |
| `GET .../driver/assignments` | Scoped active list |
| `GET .../driver/assignments/:id` | Detail + timeline |
| `POST .../driver/assignments/:id/status` | Workflow actions |
| `POST .../driver/assignments/:id/proof` | OTP / notes / photo URLs |
| `GET .../driver/history` | Archive |
| `GET .../driver/performance` | KPI aggregation |
| `GET/POST .../driver/shift/*` | Shift lifecycle |
| `PATCH .../driver/availability` | Available / busy / offline |
| `POST .../driver/sync` | Offline replay |

### Migration

`1772930000000-DriverDashboard` — delivery proof columns, `driver_shifts`, `delivery_events`, `driver_offline_actions`.

---

## Remaining gaps

- Live GPS tracking / in-app map (external Google Maps links only)
- Photo upload UI for proof (URL field ready; no camera capture)
- Signature canvas component
- Stacked delivery optimization
- Full IndexedDB offline queue on frontend (backend sync exists)
- Production migration not yet applied on Render
