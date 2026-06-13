# Driver Dashboard — Updated Specification

**Date:** 2026-06-11  
**Supersedes:** Initial SPEC gaps identified in AUDIT  
**Reflects:** Actual shipped implementation

---

## Shipped scope

### Pages (all functional, no placeholders)

| Route | Implementation |
|-------|----------------|
| `/delivery` | `DriverDashboard` + top 5 active assignments |
| `/delivery/deliveries` | Search + `DriverDeliveriesList` |
| `/delivery/active/[id]` | `ActiveDeliveryWorkspace` — workflow, proof, timeline |
| `/delivery/history` | Completed/failed list |
| `/delivery/performance` | `DriverPerformanceView` |
| `/delivery/shift` | `DriverShiftPanel` |
| `/delivery/messages` | `ChatInbox` |
| `/delivery/notifications` | `NotificationInbox` |
| `/delivery/map` | Per-assignment Google Maps links |
| `/delivery/activity` | Recent history feed |
| `/delivery/profile` | Read-only identity |
| `/delivery/settings` | LocalStorage preferences |
| `/delivery/help` | FAQ accordion |
| `/delivery/chats` | Redirects to `/delivery/messages` |

### API surface

Base: `/restaurants/:restaurantId/driver/`

See [ARCHITECTURE_REVIEW.md](./ARCHITECTURE_REVIEW.md) for endpoint list.

### Entity extensions

`deliveries`: `assignedAt`, `acceptedAt`, `pickedUpAt`, `arrivedAt`, proof fields, `driverNotes`, `failureReason`

New tables: `driver_shifts`, `delivery_events`, `driver_offline_actions`

---

## Deviations from original enterprise spec

| Original ask | Shipped as | Reason |
|--------------|------------|--------|
| Embedded map | External navigation links | Faster ship; no map SDK dependency |
| Photo capture UI | OTP + notes + URL API | Storage integration deferred |
| Signature pad | API field only | UI deferred to D4 |
| Full offline IndexedDB | Connection banner + backend sync | Partial; kitchen pattern reusable |
| Keyboard shortcuts page | Not included | Lower priority for mobile-first drivers |

---

## Acceptance criteria met

- [x] SDD documentation suite under `ai/features/driver/`
- [x] Driver sees only own assignments
- [x] Complete workflow accept → deliver
- [x] Shift management
- [x] Chat + notifications integrated
- [x] EN/FR i18n
- [x] Mobile-responsive shell
- [x] Realtime cache invalidation
- [x] Build passes
