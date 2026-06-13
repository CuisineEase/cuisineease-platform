# Driver Dashboard — Architecture Review

**Date:** 2026-06-11  
**Status:** Implemented

---

## Backend

### Module layout

```
backend/src/driver/
  driver.module.ts
  driver.controller.ts
  driver.service.ts
  dto/index.ts
  entities/
    driver-shift.entity.ts
    delivery-event.entity.ts
    driver-offline-action.entity.ts
```

Routes follow kitchen pattern: `restaurants/:restaurantId/driver/*` with `FirebaseGuard` + `RestaurantAccessService.assertDeliveryOrAbove`.

### Delivery module changes

- `DeliveryService.findAll` — filters by driver ID for `SystemRole.DELIVERY`
- `DeliveryService.update` — ownership check; auto-assign status; emits realtime
- Extended `Delivery` entity with proof + lifecycle timestamps

### Realtime

`RestaurantRealtimeService.emitDeliveryUpdated` + `emitDriverShiftUpdated` — fan-out to restaurant room, `delivery` room, and `user:{driverId}`.

### Database

Migration `1772930000000-DriverDashboard`:
- Columns on `deliveries`
- Tables: `driver_shifts`, `delivery_events`, `driver_offline_actions`

---

## Frontend

### Structure

```
app/src/app/delivery/          — pages + layout shell
app/src/config/deliveryNav.ts  — navigation config
app/src/services/driver.ts     — API client
app/src/hooks/useDriverLive.ts — WebSocket invalidation
app/src/components/Driver/     — shared UI
app/src/type/driver.ts         — TypeScript types
```

### Data flow

React Query for REST; `useDriverLive` invalidates on socket events. Mutations in `ActiveDeliveryWorkspace` and `DriverShiftPanel` invalidate scoped query keys.

### Reuse

- `ChatInbox`, `NotificationInbox`, `NotificationBell`
- `SidebarProvider` pattern from chef/waitstaff
- `useRestaurantId`, `useCurrentUser`, `useNotificationLive`, `useChatLive`

---

## Scaling considerations

| Concern | Approach |
|---------|----------|
| Large assignment lists | Pagination on API (`page`, `limit`); frontend limit 100 |
| Concurrent status updates | Last-write-wins; timeline audit for disputes |
| Multi-restaurant drivers | One `restaurantId` claim per JWT (existing model) |
| Offline | Idempotent sync via `idempotencyKey` |

---

## Recommendations (future)

1. Add optimistic updates on status buttons with rollback
2. Virtualize delivery list when >50 rows (`@tanstack/react-virtual`)
3. Extract shared `StaffPortalLayout` from chef/waitstaff/delivery shells
4. Event-sourced delivery state machine with version column for conflict detection
