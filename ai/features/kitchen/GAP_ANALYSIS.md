# Kitchen Dashboard — Gap Analysis

**Date:** 2026-06-11  
**Baseline:** [AUDIT.md](./AUDIT.md) vs [SPEC.md](./SPEC.md)  
**Status:** Pre-implementation — see [IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md) v2 (K0–K12) and [ENTERPRISE_REVIEW.md](./ENTERPRISE_REVIEW.md)

---

## Summary

| Layer | Complete | Partial | Missing | Broken |
|-------|----------|---------|---------|--------|
| Backend domain | 1 | 2 | 8 | 0 |
| Backend API/security | 2 | 2 | 6 | 0 |
| Frontend shell & nav | 0 | 0 | 10 | 0 |
| Frontend features | 0 | 1 | 9 | 0 |
| Realtime | 1 | 1 | 3 | 0 |
| Offline / E2EE | 0 | 0 | 4 | 0 |
| **Overall portal** | **~5%** | **~10%** | **~85%** | **0%** |

---

## Existing capabilities (reuse)

| Capability | Location |
|------------|----------|
| Chef role routing | `middleware.ts`, JWT claims |
| Kitchen status API | `POST /orders/:id/kitchen-status` |
| Order list/filter | `GET /orders?statuses=` |
| Waitstaff → kitchen handoff | `send-to-kitchen` |
| WS restaurant room | `join:restaurant`, `order:updated` |
| Menu read for chef | `menu:read` RBAC |
| Chat + notifications infra | `chat/`, `notifications/` |
| Inventory data | `inventory/` (manager writes) |
| Design system | shadcn + waitstaff shell patterns |

---

## Critical gaps (P0) — MVP kitchen workspace

| # | Gap | Spec § |
|---|-----|--------|
| P0-1 | Chef layout, nav, mobile drawer | §2 |
| P0-2 | Kitchen Queue (Kanban + list) with item detail | §2.2 |
| P0-3 | WS-driven live refresh on queue | §2.2, §5 |
| P0-4 | Ticket card: items, mods, allergies, table, timer | §1, §9 |
| P0-5 | `chef:` i18n namespace — zero hardcoded strings | §3 |
| P0-6 | Touch-optimized responsive queue | §4 |
| P0-7 | `useNotificationLive` + kitchen alert types | §7 |
| P0-8 | Waitstaff ready notifications wired end-to-end | §10 |
| P0-9 | Chef profile (restricted fields) | §2.9 |
| P0-10 | RBAC audit on all chef API routes | §12 |

---

## High priority (P1) — production parity

| # | Gap | Spec § |
|---|-----|--------|
| P1-1 | Dashboard overview metrics | §2.1 |
| P1-2 | Stations CRUD (manager) + chef station views | §2.3 |
| P1-3 | Tickets history + timeline API | §2.4 |
| P1-4 | Internal messages (operational channels) | §2.5 |
| P1-5 | Hold / resume / re-fire flows | §2.2 |
| P1-6 | Manager read-only kitchen monitor | §11 |
| P1-7 | Inventory read-only page | §2.7 |
| P1-8 | Settings (theme, language, sounds, timers) | §2.8 |
| P1-9 | Accessibility pass (WCAG AA) | §6 |
| P1-10 | Implement `KitchenDisplayService` (replace stub) | Technical |

---

## Medium (P2) — intelligence & collaboration

| # | Gap | Spec § |
|---|-----|--------|
| P2-1 | Analytics dashboard | §2.6 |
| P2-2 | Auto station assignment | §8 |
| P2-3 | Delay prediction + escalation | §7, §8 |
| P2-4 | Task handoff / claim tickets | §9 |
| P2-5 | Station group chats | §2.5 |
| P2-6 | Bulk queue actions | §2.2 |
| P2-7 | Chef presence (online) | §2.1 |

---

## Advanced (P3)

| # | Gap | Spec § |
|---|-----|--------|
| P3-1 | E2EE private staff DMs | §2.5, §12 |
| P3-2 | Offline queue + sync | §5 |
| P3-3 | Voice notes, reactions, pins | §2.5 |
| P3-4 | Forecasting / ML prep times | §8 |
| P3-5 | TV / kiosk display mode | §4 |

---

## Backend entities to add

| Entity | Purpose |
|--------|---------|
| `kitchen_stations` | Station config per restaurant |
| `kitchen_station_assignments` | Chef ↔ station shift |
| `order_kitchen_events` | Timeline / audit replay |
| `kitchen_ticket_routing` | Item → station map (optional join table) |
| `staff_device_keys` | E2EE key material (public keys only on server) |

---

## Definition of done

Portal matches [SPEC.md](./SPEC.md) §14 checklist and [IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md) final phase sign-off.
