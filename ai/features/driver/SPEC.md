# Driver Dashboard — Specification

**Version:** 1.0  
**Date:** 2026-06-11  
**Status:** Implemented — see [DRIVER_COMPLETION_REPORT.md](./DRIVER_COMPLETION_REPORT.md)

Path prefix: **`/delivery`** (middleware role `delivery`). Shell mirrors chef/waitstaff: sidebar + top nav + mobile drawer.

---

## 1. Vision

The Driver Portal is a **complete shift operating environment** — not merely a delivery list. Drivers clock in, accept assignments, navigate, verify proof, communicate, and review performance without leaving CuisineEase.

---

## 2. Personas & permissions

| Persona | Access |
|---------|--------|
| Delivery driver | Own assignments only; shift self-service; profile read-only for identity fields |
| Manager | Assign drivers via `/manager/orders` Deliveries tab; full delivery CRUD |
| Customer | Own delivery tracking + driver chat |
| Admin | Full access |

Drivers **cannot** edit: name, email, restaurant assignment, employee ID, role.

---

## 3. Navigation

Config: `app/src/config/deliveryNav.ts`

Dashboard · Deliveries · History · Performance · Shift · Messages · Notifications · Map · Activity · Profile · Settings · Help

Active delivery: `/delivery/active/[id]` (deep link from list).

---

## 4. Workflow

```
Assignment → Accept → Pickup → In transit → Arrive → Proof → Deliver
                                    ↓
                              Fail / Cancel
```

Status enum: `pending` → `assigned` → `in_transit` → `delivered` | `failed` | `cancelled`

Manager assignment auto-sets `assigned` + `assignedAt`.

---

## 5. Realtime

Events (restaurant room + driver user room):

- `delivery:assigned`, `delivery:updated`, `delivery:started`, `delivery:completed`, `delivery:failed`
- `driver:shift:updated`

Frontend: `useDriverLive` invalidates React Query caches.

---

## 6. Internationalization

Namespace `delivery:` — EN + FR in `app/src/locales/en.json` and `fr.json`. No hardcoded UI strings in portal pages.

---

## 7. Security

- `RestaurantAccessService.assertDeliveryOrAbove` on all driver routes
- Delivery list filtered by `deliveryPerson.id` for DELIVERY role
- PATCH ownership enforced on generic `/delivery/:id`
- Proof and timeline stored in `delivery_events` audit table

---

## 8. Integrations

| System | Integration |
|--------|-------------|
| Chat | `ChatInbox`, `customer_driver` threads |
| Notifications | `NotificationInbox`, `useNotificationLive` |
| Manager | Delivery assignment on orders |
| Kitchen | Indirect — order ready triggers pickup |
| Design system | Sidebar, cards, badges, shared inbox components |

---

## 9. Non-goals (phase 1)

- Third-party fleet APIs (Uber Direct, etc.)
- Embedded turn-by-turn navigation
- E2EE driver-customer chat (architecture-ready via chat module)
