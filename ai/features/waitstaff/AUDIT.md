# Waitstaff Portal — Phase 1 Audit

**Date:** 2026-06-11  
**Authority:** Code over documentation where they disagree  
**Scope:** Existing waitstaff-related functionality across `app/` and `backend/`

---

## Executive summary

The Waitstaff Portal is a **stub**. CuisineEase has the **role model**, **routing hooks**, **order/reservation/table primitives**, and **shared realtime/notification/chat infrastructure** needed for a full front-of-house system — but almost no waitstaff-specific product surface, APIs, or operational workflows.

| Area | Classification |
|------|----------------|
| Waitstaff functionality | **Partial** (read-only ready-order list only) |
| Backend endpoints (waitstaff-relevant) | **Partial** (shared APIs exist; no FOH-specific layer) |
| Frontend pages | **Missing** (one minimal page, no layout/nav) |
| Table management | **Partial** (CRUD + sections; no live occupancy) |
| Reservation management | **Partial** (CRUD + manager UI; no check-in/seating FOH flow) |
| Order workflows | **Partial** (full lifecycle exists; dine-in FOH path untailored) |
| Role permissions | **Partial** (model exists; guards largely disabled or absent) |
| Realtime infrastructure | **Complete** (Socket.io notifications; no order/table room events) |
| Notification infrastructure | **Complete** (staff receive order events via `getRestaurantStaffIds`) |
| Chat infrastructure | **Complete** (restaurant staff inbox; no waitstaff shell entry) |

---

## 1. Existing waitstaff functionality

### Classification: **Partial**

**What exists**

| Item | Location | Notes |
|------|----------|-------|
| `RestaurantRole.WAITSTAFF` | `backend/src/auth/dto/roles.ts` | Enum value `waitstaff` |
| Staff invite | `backend/src/user/user-restaurant-role.service.ts` | Invited as `SystemRole.USER` + `RestaurantRole.WAITSTAFF` |
| JWT claims sync | `syncFirebaseRestaurantClaims()` | Sets `{ restaurantId, role: primaryRestaurantRole }` — waitstaff get `role: waitstaff` in token |
| Post-login routing | `app/src/lib/authRouting.ts` | Maps `waitstaff` → `/waitstaff` |
| Middleware gate | `app/src/middleware.ts` | Protects `/waitstaff/*`; requires JWT `role === waitstaff` |
| Dashboard page | `app/src/app/waitstaff/page.tsx` | Lists orders with `status: ready` for `restaurantId` from localStorage |
| i18n keys | `app/src/locales/en.json` | `waitstaffDashboard.*`, staff role label |
| Order entity relation | `backend/src/orders/entities/order.entity.ts` | `waitstaff` / `waitstaffId` column in DB |
| Manager staff UI | `app/src/components/RestaurantDashboard/Staffs/*` | Managers can invite waitstaff |

**What is missing**

- Dedicated layout, sidebar, top nav, or `waitstaffNav` config (chef has the same gap)
- Sub-routes (tables, reservations, orders, guests, chat, shift)
- Actions: seat guests, create/modify orders, send to kitchen, mark served, request bill, take payment, transfer table
- `waitstaffId` assignment on orders (column exists; not exposed in DTOs or service logic)
- Table assignment to waitstaff
- Shift summary, metrics, or task list tied to waitstaff

**Risks / broken behavior**

- `persistAuthSession()` falls back to `role: "manager"` when `restaurantId` exists but role is absent from login payload (`app/src/services/auth.ts`). Waitstaff with stale tokens could mis-route.
- Middleware maps `restaurantId` without `waitstaff` claim → `manager` (`middleware.ts` lines 49–50). Users with wrong claims land on manager portal.
- No `waitstaff/layout.tsx` — no `useNotificationLive`, `useChatLive`, or `ManagerStatusGuard` equivalent.

---

## 2. Existing backend endpoints (waitstaff-relevant)

### Classification: **Partial**

Global `FirebaseGuard` is applied app-wide (`backend/src/app.module.ts`). `RolesGuard` is **selective** — many waitstaff-relevant controllers have roles commented out.

### Orders (`/orders`)

| Method | Path | Guards | Waitstaff utility |
|--------|------|--------|-------------------|
| POST | `/orders` | Firebase only | Create order (needs customerId) |
| GET | `/orders` | Firebase only | List/filter by restaurant, status |
| GET | `/orders/:id` | Firebase only | Order detail |
| PATCH | `/orders/:id` | Firebase only | Status transitions, item edits |
| DELETE | `/orders/:id` | Firebase only | Pending/cancelled only |
| POST | `/orders/from-cart` | Firebase + user | Customer checkout path |

**Status machine** (`orders.service.ts`): `pending → preparing → ready → delivered` (+ `on_hold`, `cancelled`). No `served` state for dine-in.

**Gaps:** No `waitstaffId` on create/update DTOs; no restaurant-scoped authorization; any authenticated user can patch any order if they know the UUID.

### Tables (`/tables`)

| Method | Path | Guards | Notes |
|--------|------|--------|-------|
| GET | `/tables/restaurant/:restaurantId` | **Public** | Customer dine-in picker |
| GET | `/tables/sections/restaurant/:restaurantId` | **Public** | Sections |
| POST/PATCH/DELETE | various | **None** (guards commented; file marked DO NOT TOUCH) | Manager CRUD |

**Gaps:** No `status` (available/occupied/dirty/reserved); no `assignedWaitstaffId`; no `notes` field.

### Reservations (`/reservations`)

| Method | Path | Guards | Notes |
|--------|------|--------|-------|
| POST | `/` | Firebase + RolesGuard (roles commented) | Create |
| GET | `/me` | Firebase + RolesGuard | Customer list |
| GET | `/restaurant/:id` | Firebase + RolesGuard (WAITSTAFF commented) | Restaurant list |
| PATCH | `/:id` | Firebase + RolesGuard (WAITSTAFF commented) | Update |
| POST | `/:id/pay` | Firebase + RolesGuard | MeSomb payment |
| GET | `/table/:tableId/availability` | MANAGER, USER | Availability check |

**Reservation statuses:** `pending`, `confirmed`, `cancelled` only — no `checked_in`, `seated`, `no_show`, `completed`.

### Users / staff

| Method | Path | Relevance |
|--------|------|-----------|
| POST | `/user/restaurant/:restaurantId/staff` | Manager invites waitstaff |
| GET | `/user/restaurant/:restaurantId` | List staff |
| GET/PATCH | `/user/notification-preferences` | Per-user notification settings |

### Notifications (`/notifications`)

Full CRUD + WebSocket delivery. Order events fan out to **all** restaurant staff IDs (`notification-recipients.service.ts`).

### Chat (`/chat`)

Conversations, messages, image upload. Staff access via `UserRestaurantRole` (fixed in recent chat work). No waitstaff-specific endpoints.

### Payments (`/payment`)

`processPayment` requires order belongs to **customer** making payment — no staff-initiated cash/POS path for dine-in bill settlement.

### Reviews (`/reviews`)

Customer → restaurant reviews exist; no guest lookup API for waitstaff.

### Tasks (`/tasks`)

Generic tutorial entity (title, description, status) — **no** `restaurantId`, assignee, or shift linkage. Not suitable for waitstaff shift tasks without extension.

### Kitchen display

`kitchen-display.service.ts` — **stub** (TODO logs only).

---

## 3. Existing frontend pages

### Classification: **Missing** (portal shell); **Partial** (shared manager components exist)

| Path | Role | Status |
|------|------|--------|
| `/waitstaff` | Waitstaff | **Partial** — single page, ready orders list |
| `/waitstaff/*` | Waitstaff | **Missing** — no sub-routes |
| `/chef` | Chef | **Partial** — mirror of waitstaff (preparing orders) |
| `/manager/tables` | Manager | **Complete** — full table CRUD UI |
| `/manager/reservations` | Manager | **Complete** — list, create, filters |
| `/manager/orders` | Manager | **Complete** — tabs, status filters, detail pages |
| `/manager/chat` | Manager | **Complete** — `ChatInbox` |
| `/manager/notifications` | Manager | **Complete** |
| `/customer/cart` | Customer | Dine-in table picker uses public tables API |

**No** `app/src/config/waitstaffNav.ts`. **No** `waitstaff/layout.tsx`.

Reusable hooks/services waitstaff can leverage today:

- `useOrdersList`, `useRestaurantId`, `useRestaurantReservations`, `useTablesByRestaurant`
- `services/order.ts`, `services/reservation.ts`, `services/tables.ts`
- `useNotificationLive`, `useChatLive` (not wired on waitstaff routes)

---

## 4. Table management

### Classification: **Partial**

**Complete**

- Entity: section, number, name, seatCapacity, pricePerHourXaf, image, soft delete
- Sections CRUD + photo upload
- Manager UI: list, create, edit, delete, sections panel
- Public read for customer dine-in checkout
- Unique index per restaurant/section/number

**Missing**

- Runtime status: vacant, seated, ordering, eating, dirty, blocked
- Current party / reservation link on table
- Active order link on table
- Waitstaff assignment
- Floor plan spatial layout (coordinates)
- Table notes (allergy alert, VIP, etc.)
- Realtime table status broadcasts

**Broken / risk**

- Write endpoints on `/tables` have **no auth guards** (intentionally frozen per comment). Security risk for production; waitstaff spec must not duplicate CRUD — read + status update only.

---

## 5. Reservation management

### Classification: **Partial**

**Complete**

- Create (customer + manager), list by restaurant, update status, pay via MeSomb
- Table availability check
- Manager reservations page with create dialog
- Payment status on reservation entity

**Missing**

- FOH check-in workflow (mark arrived, assign alternate table)
- Seating workflow (transition to seated → open table session)
- No-show handling
- Walk-in queue
- Arrival notifications tuned for waitstaff (staff get generic order notifications only today)
- Reservation → order linking

**Broken**

- `@Roles(RestaurantRole.WAITSTAFF)` commented out on restaurant list/update — waitstaff **can** call APIs today if they obtain tokens, but **no UI** and no explicit permission documentation.

---

## 6. Order workflows

### Classification: **Partial**

**Complete**

- Full order CRUD, cart checkout, status transitions, payment (customer Mobile Money)
- Order types: dine-in, takeout, delivery
- `tableNumber` on dine-in orders
- Event emitter: `order.created`, `order.updated`, `order.deleted`
- Notifications to customer + all restaurant staff on status changes
- Manager order UI with status advancement (`ready` → `delivered`)
- Analytics hooks on order events

**Partial for dine-in FOH**

- Chef page: view `preparing` orders (no actions)
- Waitstaff page: view `ready` orders (no mark served / delivered)
- No "send to kitchen" distinct action (status `pending → preparing` is the transition)
- No split bill, add items mid-meal UX for staff
- `waitstaffId` never set

**Missing**

- Staff-created walk-in orders (guest without app account — would need guest customer record or nullable customer)
- Cash payment recording by staff
- Bill request signal to manager/POS
- Course firing / item-level kitchen status

---

## 7. Role permissions

### Classification: **Partial**

**Complete**

- `SystemRole` vs `RestaurantRole` separation
- `UserRestaurantRole` join table
- `RolesGuard` with restaurant-scoped checks via `hasAnyRole`
- Firebase custom claims for routing
- Staff invite flow for waitstaff

**Partial / disabled**

| Resource | Expected | Actual |
|----------|----------|--------|
| Orders | Restaurant-scoped RBAC | Firebase only |
| Tables | Manager write, staff read | Guards commented out |
| Reservations | WAITSTAFF on read/update | Roles commented out |
| Payments | Staff cash tender | Customer-only |

**Missing**

- Permission matrix documented and enforced for waitstaff actions
- Restaurant membership check on order mutations (any UUID holder can update)
- Fine-grained actions (e.g. waitstaff can PATCH status but not DELETE menu)

---

## 8. Realtime infrastructure

### Classification: **Complete** (platform); **Missing** (FOH-specific)

**Complete**

- Socket.io on main API HTTP server (`IoAdapter`)
- Notifications gateway pushes `notification:new` to user rooms
- Chat gateway: `chat:message:new`, conversation updates
- Frontend: `useNotificationLive`, `useChatLive`, `useChatSocket`
- Order events trigger WebSocket notifications to staff

**Missing**

- Dedicated order board room per restaurant (clients poll REST today for chef/waitstaff pages)
- Table status change events
- Reservation arrival events
- Optimistic order cache invalidation on waitstaff portal (no React Query subscriptions)

---

## 9. Notification infrastructure

### Classification: **Complete**

- `NotificationsService` with WEB_SOCKET + MOBILE_PUSH channels
- Order lifecycle messages for customer and manager copy (`order-notification.messages.ts`, `manager-order-notification.messages.ts`)
- All restaurant staff (manager, chef, waitstaff) receive manager-style order alerts via `getRestaurantStaffIds`
- User notification preferences entity
- Bell + inbox pages exist for manager/customer (not mounted on waitstaff layout)

**Missing**

- Notification types specific to FOH: `RESERVATION_ARRIVING`, `TABLE_WAITING`, `GUEST_READY_FOR_BILL`
- Role-targeted notifications (e.g. only assigned waitstaff)
- Waitstaff notifications page route

---

## 10. Chat infrastructure

### Classification: **Complete** (platform); **Partial** (waitstaff access path)

**Complete**

- Conversations: customer↔restaurant, delivery, order-linked threads
- Staff participant sync for restaurant roles
- Manager `/manager/chat` with `ChatInbox`, unread badges, mobile master-detail
- Image attachments, WebSocket + polling

**Missing for waitstaff**

- `/waitstaff/chat` route and nav link
- Kitchen-specific channel (today: same restaurant inbox or order threads)
- Quick actions from table/order context ("message kitchen about table 5")

---

## Cross-cutting findings

### Auth & routing

- Invited waitstaff: `syncFirebaseRestaurantClaims` sets JWT `role: waitstaff` — **correct** when claims are fresh.
- Login response includes `role: claimRole` from Firebase (`auth.service.ts`).
- Users must refresh token after invite for middleware to see new role.

### Data model reuse (no parallel models needed)

| Concept | Entity | Waitstaff gap |
|---------|--------|---------------|
| Tables | `Table` | Add operational fields or companion `TableSession` |
| Orders | `Order` | Wire `waitstaffId`; optional `servedAt` |
| Reservations | `Reservation` | Extend status enum |
| Guests | `User` (customer role) | No allergies/preferences fields |
| Staff | `UserRestaurantRole` | No shift/assignment entity |
| Tasks | `Task` | Unrelated scaffold — extend or new `ShiftTask` |

### Chef portal parity

Chef and waitstaff pages are identical patterns (filter orders by status). Any KDS/waitstaff investment should share components (`StaffOrderBoard`, order detail drawer).

---

## Audit conclusion

CuisineEase is **not production-ready** for front-of-house operations. The platform provides **~40% of the plumbing** (roles, orders, tables, reservations, notifications, chat) and **~5% of waitstaff product** (one passive list page).

**Recommended next step:** Review `SPEC.md`, then `GAP_ANALYSIS.md` and `ARCHITECTURE_REVIEW.md` before Phase 5 implementation.
