# Kitchen Dashboard — Updated Specification (As Built)

**Version:** 3.0 (release candidate)  
**Date:** 2026-06-11  
**Baseline spec:** [SPEC.md](./SPEC.md)  
**Completion:** [KITCHEN_COMPLETION_REPORT.md](./KITCHEN_COMPLETION_REPORT.md)

---

## Portal routes (implemented)

| Route | Function |
|-------|----------|
| `/chef` | Dashboard — summary metrics |
| `/chef/queue` | Primary queue — DnD Kanban + virtualized list/compact |
| `/chef/active` | In-progress filter |
| `/chef/stations` | Station workload |
| `/chef/history` | Virtualized audit search |
| `/chef/messages` | Staff DMs (ChatInbox + E2EE) |
| `/chef/team` | Operational channels |
| `/chef/alerts` | Notifications inbox |
| `/chef/shift` | Shift summary |
| `/chef/analytics` | Metrics, heatmaps, export, AI insights |
| `/chef/performance` | KPI cards |
| `/chef/inventory` | Read-only stock |
| `/chef/activity` | Today's audit |
| `/chef/profile` | Restricted identity |
| `/chef/settings` | Theme, language, sounds |
| `/chef/help` | Shortcuts + FAQ |
| `/chef/display` | TV/kiosk — auto-rotate, themes, fullscreen |
| `/manager/kitchen` | Manager read-only kitchen monitor |

---

## API (implemented)

| Method | Path |
|--------|------|
| GET | `/restaurants/:id/kitchen/queue` |
| GET | `/restaurants/:id/kitchen/summary` |
| GET | `/restaurants/:id/kitchen/stations` |
| POST/PATCH | `/restaurants/:id/kitchen/stations` |
| POST | `/restaurants/:id/kitchen/queue/reorder` |
| GET | `/restaurants/:id/kitchen/monitor` |
| GET | `/restaurants/:id/kitchen/analytics/export` |
| GET | `/restaurants/:id/kitchen/orders/:orderId` |
| POST | `/restaurants/:id/kitchen/orders/:orderId/:action` |
| POST | `/restaurants/:id/kitchen/orders/:orderId/claim` |
| POST | `/restaurants/:id/kitchen/orders/:orderId/undo` |
| PATCH | `/restaurants/:id/kitchen/orders/:orderId/items/:itemId` |
| GET | `/restaurants/:id/kitchen/history` |
| GET | `/restaurants/:id/kitchen/analytics` |
| POST | `/restaurants/:id/kitchen/sync` |
| POST | `/user/me/device-keys` |
| GET | `/user/me/device-keys` |
| POST | `/user/:userId/device-keys/public` |
| POST | `/chat/conversations/:id/messages/:messageId/reactions` |
| POST | `/chat/conversations/:id/messages/:messageId/pin` |

---

## Realtime events

`order:updated`, `order:new`, `order:started`, `order:ready`, `order:paused`, `order:refire`, `kitchen:queue:updated`, `station:update`, `alert:new`, `metrics:update`

---

## E2EE architecture

- Device identity: ECDH P-256 key pairs registered in `staff_device_keys`
- Session keys: derived per conversation, rotated on demand
- Server transports ciphertext only for `staff_dm` conversations
- Operational channels remain organization-visible

---

## Profile policy (enforced)

Chefs cannot edit name, email, restaurant, employee ID, or job title — managers retain identity management.

---

## Migrations

1. `1772910000000-KitchenDashboard.ts`
2. `1772920000000-KitchenEnterprisePolish.ts`
