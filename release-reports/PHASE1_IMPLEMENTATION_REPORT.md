# Phase 1 Implementation Report — Cross-Dashboard Consistency

**Date:** 2026-06-11  
**Scope:** High Priority (🟠) findings ENT-H01 through ENT-H14

## Summary

Phase 1 high-priority items were addressed where they had clear code paths. Several cross-cutting themes (universal search, full i18n pass, admin expansion) remain partially complete and are tracked in the backlog below.

---

## Completed High-Priority Items

| ID | Finding | Resolution |
|----|---------|------------|
| **H-01** | Manager driver UUID paste | Driver picker dropdown from `getRestaurantStaff` filtered to `delivery` role |
| **H-04** | Staff role edit after invite | Role select + PATCH on manager staff detail page |
| **H-06** | `payment:recorded` not consumed | Added listener in `useFloor` |
| **H-07** | Driver offline sync unwired | `driverOffline.ts` IndexedDB queue + `DriverOfflineSyncBanner` + existing `syncDriverOfflineActions` API |
| **H-10** | `expireSession` no route | `POST /table-sessions/:id/expire` + frontend `expireTableSession()` |
| **H-11** | AnalyticsService stub | In-memory per-restaurant metrics + structured JSON logging |
| **H-13** | Notify driver when order kitchen-ready | `emitDeliveryPickupReady` on order `ready` for delivery type; driver WS listener |

---

## Partial / Foundation Only

| Theme | Status | Notes |
|-------|--------|-------|
| **H-02–H03** Reservations lifecycle | 🟡 Backlog | Cancel/confirm APIs may exist; manager/customer UI actions not fully wired in this pass |
| **H-05** Inventory forms | 🟡 Backlog | Page exists; real CRUD vs rename decision pending |
| **H-08** Waitstaff i18n | 🟡 Backlog | Delivery/chef namespaces complete; waitstaff pages need key audit |
| **H-09** Chef notifications parity | 🟡 Partial | Shared notification components exist; chef inbox uses same route as other portals |
| **H-12** Manager global search | 🟡 Backlog | Stub input remains; needs orders/reservations/staff unified search |
| **H-14** Admin expansion | 🟡 Backlog | Admin dashboard minimal |

---

## Cross-Dashboard Themes (Phase 1 scope)

| Theme | Progress |
|-------|----------|
| Universal search | Not started (manager stub remains) |
| Unified notifications | Shared `/notifications` route across portals; live WS via `useNotificationLive` |
| Unified activity history | Driver/kitchen history exist; no shared timeline component yet |
| Shift management | Driver + waitstaff shift panels exist; metrics not fully standardized |
| Translation completeness | Delivery fixed; waitstaff/admin pass incomplete |
| Offline consistency | Kitchen + driver offline queues aligned |
| Profile & settings | Per-portal settings pages exist; field restrictions not fully enforced in UI |
| Analytics standardization | Manager revenue fixed (Phase 0); backend AnalyticsService improved |

---

## Files Changed (Phase 1 highlights)

- `app/src/components/RestaurantDashboard/OrderPage/ManagerDeliveriesContent.tsx`
- `app/src/app/manager/staff/[id]/page.tsx`
- `app/src/lib/driverOffline.ts`
- `app/src/components/Driver/DriverOfflineSyncBanner.tsx`
- `app/src/services/floor.ts`
- `app/src/services/staff.ts`
- `backend/src/floor/table-sessions.controller.ts`
- `backend/src/orders/services/analytics.service.ts`

---

## Remaining Backlog (by priority)

### Critical
_None remaining from Phase 0 audit._

### High
- H-02–H03 Reservation manager confirm/cancel + customer cancel UI
- H-05 Inventory real implementation or honest placeholder rename
- H-08 Waitstaff EN/FR completeness
- H-09 Chef notification center parity (mark read, filters)
- H-12 Manager operational search
- H-14 Admin SaaS operator dashboard

### Normal / Low
- Shared activity timeline component
- Saved views, chat reactions/pins exposure
- Payment websocket listeners on customer tracking
- Device keys UI for non-kitchen roles

---

## Validation

| Check | Result |
|-------|--------|
| Backend build | ✅ Pass |
| Frontend build | ✅ Pass |
