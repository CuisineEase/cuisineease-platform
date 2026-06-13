# Manager Portal — Specification

**Version:** 1.0  
**Date:** 2026-06-11  
**Status:** Production

---

## Business goals

Restaurant owners and managers run day-to-day operations: catalog, orders, floor layout, staff, inventory, reservations, reviews, and customer messaging — without platform-admin access.

| Problem | Outcome |
|---------|---------|
| Scattered ops tools | Single `/manager` dashboard |
| Pending restaurants blocked | Dedicated pending/suspended states |
| Order noise | Tabbed order views with status buckets |
| Staff coordination | Role assignment + chat/notifications |

---

## Personas

| Persona | Primary goals |
|---------|---------------|
| **Owner / Manager** | Full restaurant ops |
| **Applicant** | Submit restaurant, wait for admin approval |
| **Suspended manager** | Read-only suspended page until reinstated |

---

## Access & routing

| Item | Value |
|------|-------|
| **Restaurant roles** | `owner`, `manager` |
| **System role** | Often `user` + JWT `restaurantId` |
| **Path prefix** | `/manager` |
| **Middleware** | `app/src/middleware.ts` — `restaurantId` claim → manager routes |
| **Status guard** | `ManagerStatusGuard` — pending/suspended/rejected routing |

### Restaurant status routing

| Backend status | Route | Behavior |
|----------------|-------|----------|
| `pending` | `/manager/pending` | Limited UI until admin approves |
| `active` | `/manager/*` | Full dashboard |
| `suspended` | `/manager/suspended` | Blocked from ops |
| `rejected` | Logout → login | Stale session cleanup |

**Guard:** `app/src/components/manager/ManagerStatusGuard.tsx`  
**Status API:** `getManagerApplicationStatus` from `platformAdmin.ts`

---

## Navigation

Config: `app/src/config/managerNav.ts`

| Route | Sidebar key | Purpose |
|-------|-------------|---------|
| `/manager` | `sidebar.overview` | Dashboard home |
| `/manager/menu` | `sidebar.menu` | Menu CRUD |
| `/manager/orders` | `sidebar.orders` | Order list + tabs |
| `/manager/reservations` | `sidebar.reservations` | Reservation management |
| `/manager/tables` | `sidebar.tables` | Tables & sections |
| `/manager/inventory` | `sidebar.inventory` | Stock / ingredients |
| `/manager/staff` | `sidebar.staff` | Staff & roles |
| `/manager/reviews` | `sidebar.reviews` | Customer reviews |
| `/manager/messages` | `sidebar.messages` | Customer chat inbox |
| `/manager/notifications` | `sidebar.notifications` | Alerts |
| `/manager/settings` | `sidebar.settings` | Restaurant profile |

**Nested routes:**

| Route | Purpose |
|-------|---------|
| `/manager/orders/[id]` | Order detail |
| `/manager/staff/[id]` | Staff member detail |
| `/manager/pending` | Awaiting approval (no sidebar) |
| `/manager/suspended` | Suspended notice (no sidebar) |

**Layout:** `app/src/app/manager/layout.tsx` — `AdminSideBar`, `AdminTopNav`, live hooks.

---

## Key components

| Area | Location |
|------|----------|
| Overview | `components/RestaurantDashboard/OverviewPage/*` |
| Menu | `components/RestaurantDashboard/MenuPage/*` |
| Orders | `components/RestaurantDashboard/OrderPage/ManagerOrdersContent.tsx` |
| Deliveries | `components/RestaurantDashboard/OrderPage/ManagerDeliveriesContent.tsx` |
| Tables | `components/RestaurantDashboard/Tables/*` |
| Staff | `components/RestaurantDashboard/Staffs/*` |
| Inventory | `components/RestaurantDashboard/Inventary/*` |
| Profile | `components/RestaurantDashboard/RestaurantProfileEditor.tsx` |
| Shell | `app/manager/AdminSideBar.tsx`, `AdminTopNav.tsx` |

---

## Frontend services

| Service | Purpose |
|---------|---------|
| `dashboard.ts` | Overview stats aggregation |
| `restaurant.ts` | Profile, schedules, status |
| `menu.ts`, `category.ts`, `ingredient.ts`, `addOn.ts` | Catalog |
| `order.ts` | List/filter orders (`statuses` query) |
| `reservation.ts` | Restaurant reservation list |
| `tables.ts`, `tableSections.ts` | Floor layout CRUD |
| `staff.ts` | Invites, role assignment |
| `inventory.ts` | Inventory rows |
| `review.ts` | Review list |
| `chat.ts` | Customer messages |
| `notification.ts` | Manager inbox |
| `delivery.ts` | Delivery assignment views |
| `platformAdmin.ts` | Application status (manager scope) |
| `payment.ts` | Payment detail where shown |

**Order view layer:** `app/src/lib/orderStatusViews.ts` — `MANAGER_ORDER_VIEW_TABS`, `getManagerOrderStatusesQuery`.

---

## Backend modules (manager-facing)

| Module | Manager operations |
|--------|-------------------|
| `restaurant` | Update profile, `POST /restaurant/with-manager` (onboarding) |
| `menu-item` | Full CRUD, photo upload, `menu:write` |
| `orders` | List by restaurant, status updates (role-dependent) |
| `reservations` | `GET /reservations/restaurant/:id` |
| `tables` | Table/section management ⚠️ some routes unguarded (SEC-001) |
| `user` | Staff invites, `user/restaurant-roles` |
| `inventory` | Stock management |
| `review` | Read reviews for restaurant items |
| `chat` | Reply to customer threads |
| `notifications` | Manager user inbox |
| `delivery` | Assign/track deliveries |

**RBAC:** `RestaurantAccessService` + restaurant role checks. Manager and owner have equivalent portal access unless noted in ADRs.

### Menu permissions

| Permission | Roles |
|------------|-------|
| `menu:read` | manager, owner, admin (+ waitstaff/chef for ops) |
| `menu:write` | manager, owner, admin |

---

## Orders UI — status buckets

Manager never displays raw backend enums in tabs. Filtering uses `statuses` query param:

| Tab | Statuses sent to API |
|-----|----------------------|
| All | *(none)* |
| Pending | draft, sent_to_kitchen, preparing, pending, ready, on_hold |
| Completed | served, completed, delivered |
| Cancelled | cancelled |

**Component:** `ManagerOrdersContent.tsx`  
**Detail:** `/manager/orders/[id]`

Dine-in order progression is primarily driven by waitstaff/chef; manager has oversight and can intervene on supported transitions.

---

## Dashboard overview

`/manager` page composes:

- `WelcomeSection`, `StatsCards` — from `getManagerOverviewStats`
- `OrderSummary`, `ReservationSummary`
- `BestSellers`, `CustomerMap`
- `RecentOrders`, `RecentActivity`

Data loaded via `getManagerDashboardData({ restaurantId })` — parallel fetch orders, menu, reservations.

---

## Realtime

| Hook | Scope |
|------|-------|
| `useNotificationLive` | User notifications |
| `useChatLive` | Customer message threads |

Manager UI does not subscribe to `join:restaurant` floor room by default — floor ops are waitstaff portal. Order list may refresh on navigation or polling.

---

## Onboarding flow

1. Register via `POST /auth/register` (manager intent)
2. Apply: `POST /restaurant/with-manager`
3. Status `pending` → `/manager/pending`
4. Platform admin: `PATCH /restaurant/:id/status` → `active`
5. Manager gains full `/manager` access

---

## Multi-tenancy

All manager API calls scope to JWT `restaurantId` or explicit `restaurantId` param validated by `RestaurantAccessService`.

**Rule:** Never show another restaurant's orders, staff, or floor data.

---

## Known gaps & security notes

| Item | Notes |
|------|-------|
| SEC-001 | `tables.controller.ts` write routes lack guards — manager should use secured patterns where available |
| Inventory | UI present; depth of backend rules varies |
| Revenue stats | Overview uses `delivered` orders for revenue XAF sum |
| Waitstaff floor | Manager configures tables; waitstaff runs sessions |

---

## Related docs

- Flows: [FLOWS.md](./FLOWS.md)
- Waitstaff ops (floor): [../waitstaff/WAITSTAFF_COMPLETION_REPORT.md](../waitstaff/WAITSTAFF_COMPLETION_REPORT.md)
- System flows: [../../workflow/SYSTEM_FLOWS.md](../../workflow/SYSTEM_FLOWS.md)
- Security backlog: [../../SECURITY_BACKLOG.md](../../SECURITY_BACKLOG.md)
