# Phase 0 Implementation Report — Enterprise Audit

**Date:** 2026-06-11  
**Scope:** Critical (🔴) findings ENT-C01 through ENT-C08

## Summary

All eight Phase 0 critical integrity items were implemented across backend and frontend. Backend and frontend builds pass. Migration `1772940000000-EnterprisePhase0` adds a unique partial index on `deliveries.orderId`.

---

## C-01 — Delivery Completion → Order Status Sync

**Status:** ✅ Resolved

- Added `OrdersService.syncOrderFromDeliveryCompletion()` to set `Order.status = delivered`, emit `OrderUpdatedEvent`, and broadcast realtime kitchen/order events.
- Wired from `DriverService.updateStatus` on `deliver` action and `DeliveryService.update` when status becomes `DELIVERED`.

**Files:** `backend/src/orders/orders.service.ts`, `backend/src/driver/driver.service.ts`, `backend/src/delivery/delivery.service.ts`

---

## C-02 — Dine-in Must Use TableSession

**Status:** ✅ Resolved

- Dine-in order creation now requires an active `TableSession`, resolved via `tableSessionId` or `tableId` (active session lookup).
- Customer cart checkout passes `tableId` for dine-in orders.

**Files:** `backend/src/orders/orders.service.ts`, `backend/src/orders/dto/create-order.dto.ts`, `app/src/services/order.ts`, `app/src/app/customer/cart/page.tsx`

---

## C-03 — Manager Revenue Accuracy

**Status:** ✅ Resolved

- Shared revenue eligibility constants: `delivered`, `completed`, `served`.
- Manager overview stats use `isRevenueEligibleOrderStatus()` instead of `delivered` only.

**Files:** `backend/src/common/order-revenue.util.ts`, `app/src/lib/orderRevenue.ts`, `app/src/services/dashboard.ts`

---

## C-04 — Delivery Orders Enter Kitchen Queue

**Status:** ✅ Resolved

- Added `OrdersService.autoRoutePaidOrderToKitchen()` for paid delivery/takeout orders (`pending`/`draft` → `sent_to_kitchen`).
- Called from `PaymentService.processPayment` after successful payment.
- Allowed transition `pending` → `sent_to_kitchen`.

**Files:** `backend/src/orders/orders.service.ts`, `backend/src/payment/payment.service.ts`

---

## C-05 — Complete Session Payment Flow

**Status:** ✅ Resolved (mobile money path)

- Session `recordPayment` now processes MTN/Orange mobile money via `PaymentGatewayResolver`.
- Successful mobile payments update order payment status, auto-close session when billing allows, emit `payment:recorded`.
- Cash path unchanged; session closes when fully settled.

**Files:** `backend/src/floor/table-sessions.service.ts`, `backend/src/floor/floor.module.ts`

---

## C-06 — Standardize WebSocket Events

**Status:** ✅ Resolved

- Backend emits canonical names: `kitchen:station:updated`, `kitchen:summary:updated`.
- Legacy aliases `station:update`, `metrics:update` retained for backward compatibility.
- Frontend kitchen hook listens to both canonical and legacy names.
- Added `delivery:pickup_ready` event for driver dispatch.
- Waitstaff floor hook now consumes `payment:recorded`.

**Files:** `backend/src/floor/restaurant-realtime.service.ts`, `app/src/hooks/useKitchenLive.ts`, `app/src/hooks/useFloor.ts`, `app/src/hooks/useDriverLive.ts`

---

## C-07 — Prevent Duplicate Deliveries

**Status:** ✅ Resolved

- Backend validation rejects duplicate delivery per order.
- Migration `UQ_deliveries_order_active` unique index on `orderId` where not soft-deleted.

**Files:** `backend/src/delivery/delivery.service.ts`, `backend/src/lib/database/migrations/1772940000000-EnterprisePhase0.ts`

---

## C-08 — Live Restaurant RBAC

**Status:** ✅ Resolved

- Added `GET /user/me/restaurant-roles` for live DB role assignments.
- Frontend `useLiveRestaurantAccess` hook validates assignment on staff portal layouts (manager, chef, waitstaff, delivery).

**Files:** `backend/src/user/user.controller.ts`, `app/src/hooks/useLiveRestaurantAccess.ts`, staff portal layouts

---

## Validation

| Check | Result |
|-------|--------|
| Backend build | ✅ Pass |
| Frontend build | ✅ Pass |
| Unit test `order-revenue.util.spec.ts` | ✅ Pass |
| Migration | ⚠️ Run `1772940000000-EnterprisePhase0` on deploy |

## Deployment Note

Run migration after backend deploy:

```bash
npm run migration:run
```
