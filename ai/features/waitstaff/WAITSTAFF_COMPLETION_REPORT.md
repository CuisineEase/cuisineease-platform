# Waitstaff Portal — Phase 8 Completion Report

**Date:** 2026-06-11  
**Status:** Production-complete for dine-in workflow (MVP)

---

## Objective

Deliver an end-to-end dine-in workflow entirely within the waitstaff portal:

**Seat Guest → Create Order → Send To Kitchen → Serve → Request Bill → Record Payment → Close Session**

---

## Workflow verification

| Step | UI location | API | Status |
|------|-------------|-----|--------|
| Seat guest | `/waitstaff/tables/[tableId]` | `POST /table-sessions` + `POST /guests` | Complete |
| Create order | `/waitstaff/sessions/[sessionId]/orders/new` | `POST /orders/staff` | Complete |
| Send to kitchen | Table detail — order card | `POST /orders/:id/send-to-kitchen` | Complete |
| Serve | Table detail — order card | `POST /orders/:id/mark-served` | Complete |
| Request bill | Table detail + billing page | `POST /table-sessions/:id/request-bill` | Complete |
| Record payment | Billing page — payment dialog | `POST /table-sessions/:id/payments` | Complete |
| Close session | Billing page — close dialog | `POST /table-sessions/:id/close` | Complete |

Chef steps (`sent_to_kitchen` → `preparing` → `ready`) on `/chef` — see [../kitchen/SPEC.md](../kitchen/SPEC.md) for full kitchen roadmap.

Flow diagrams: [FLOWS.md](./FLOWS.md)

---

## Deliverables added in Phase 8

### Backend

| Item | Path / endpoint |
|------|-----------------|
| `billRequestedAt` on `TableSession` | Migration `1772890000000-TableSessionBillRequested` |
| Billing summary | `GET /table-sessions/:id/billing` |
| Guest timeline | `GET /table-sessions/:id/timeline` |
| Request bill | `POST /table-sessions/:id/request-bill` |
| Table detail | `GET /restaurants/:restaurantId/tables/:tableId` |
| Session payment (session-level cash) | Enhanced `POST /table-sessions/:id/payments` |
| Close session guards | Balance due = 0, orders settled |
| Orders RBAC | `findAllSecured`, `findOneSecured`, `updateDraftByStaff` |

### Frontend

| Item | Route / component |
|------|-------------------|
| Table detail page | `/waitstaff/tables/[tableId]` |
| Menu picker / order builder | `/waitstaff/sessions/[sessionId]/orders/new` |
| Billing page | `/waitstaff/sessions/[sessionId]/billing` |
| Payment dialog | `RecordPaymentDialog` |
| Close session dialog | `CloseSessionDialog` |
| Guest timeline | `GuestTimeline` on table + billing |
| Floor → detail navigation | `/waitstaff/tables` links to detail |

---

## Files changed (summary)

### Backend
- `src/floor/table-sessions.service.ts` — billing, timeline, requestBill, payment/close logic
- `src/floor/table-sessions.controller.ts` — new routes
- `src/floor/floor.service.ts` — `getTableDetail`
- `src/floor/floor.controller.ts` — table detail route; Firebase-only guards
- `src/floor/entities/table-session.entity.ts` — `billRequestedAt`
- `src/orders/orders.service.ts` — RBAC helpers
- `src/orders/orders.controller.ts` — secured list/get/update/delete
- `src/lib/database/migrations/1772890000000-TableSessionBillRequested.ts`

### Frontend
- `src/app/waitstaff/tables/[tableId]/page.tsx`
- `src/app/waitstaff/sessions/[sessionId]/orders/new/page.tsx`
- `src/app/waitstaff/sessions/[sessionId]/billing/page.tsx`
- `src/components/Waitstaff/*` — GuestTimeline, RecordPaymentDialog, CloseSessionDialog
- `src/services/floor.ts`, `src/hooks/useTableDetail.ts`
- `src/app/waitstaff/tables/page.tsx` — navigation to detail

---

## Migrations required

1. `1772880000000-WaitstaffFloor` (if not yet applied)
2. `1772890000000-TableSessionBillRequested`

---

## Acceptance criteria

- [x] Full dine-in workflow executable without leaving `/waitstaff/*`
- [x] Table detail shows guest, orders, timeline, workflow actions
- [x] Menu picker creates draft orders on active session
- [x] Billing shows totals, payments, balance, close when paid
- [x] Manager notified on bill request (WebSocket + push)
- [x] Orders list/get scoped by RBAC
- [x] Backend and frontend builds pass

---

## Known limitations (post-MVP)

- Mobile Money staff-initiated payment records `PENDING` (customer app flow unchanged)
- No split bill per guest
- No section-based floor filtering
- Generic `POST /orders` (non-staff) still unscoped — customers use cart/checkout paths
- French `waitstaff` i18n partial (English fallback)

---

## Sign-off

The waitstaff portal meets Phase 8 completion criteria for production dine-in service at restaurants using CuisineEase table sessions.
