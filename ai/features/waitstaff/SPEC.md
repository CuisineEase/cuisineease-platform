# Waitstaff Portal — Specification

**Version:** 1.0 (implemented)  
**Date:** 2026-06-11  
**Status:** **Implemented** — see [WAITSTAFF_COMPLETION_REPORT.md](./WAITSTAFF_COMPLETION_REPORT.md) and [FLOWS.md](./FLOWS.md)

---

## Business goals

Restaurant front-of-house (FOH) staff need a single operational surface to run the dining room without switching to manager tools or paper tickets.

| Problem | Outcome |
|---------|---------|
| Staff cannot see what needs attention now | Real-time shift dashboard: tables, orders, reservations, alerts |
| Seating and reservations are disconnected | Unified seating workflow from reservation → table → order |
| Kitchen and floor are out of sync | Clear handoffs: send to kitchen, ready pings, mark served |
| Guests lack context at the table | Guest profile: history, preferences, allergies (where data exists) |
| Communication is fragmented | In-app chat with manager, kitchen context, and customers when appropriate |
| Managers cannot delegate floor work | Waitstaff actions respect RBAC; managers retain oversight |

The portal must feel native to CuisineEase: same design system, auth, APIs, notifications, and chat patterns as `/manager`.

---

## Personas

| Persona | Primary goals |
|---------|---------------|
| **Waitstaff** | Run tables, take orders, serve food, handle payments, communicate |
| **Floor lead** (waitstaff + elevated permissions) | Transfer tables, void items, escalate to manager |
| **Manager** | Configure floor, assign sections, view waitstaff activity (existing portal) |
| **Chef** | Receive orders, mark preparing/ready (existing `/chef` — integrated, not duplicated) |

---

## User stories

### Overview & shift

- As waitstaff, I see a **shift summary** (active tables, open orders, ready items, upcoming reservations) when I open the app.
- As waitstaff, I see **only my restaurant** (from JWT `restaurantId`).
- As waitstaff, I receive **alerts** (order ready, reservation in 15 min, manager message) via bell and optional push.

### Tables

- As waitstaff, I view **assigned tables** in list and floor layout.
- As waitstaff, I see each table's **status** (available, reserved, seated, ordering, served, needs cleaning).
- As waitstaff, I **seat a walk-in** by selecting a table and party size.
- As waitstaff, I **seat a reservation** by checking in and confirming table assignment.
- As waitstaff, I **transfer a table** to another table or another server (manager approval optional — see permissions).
- As waitstaff, I add **table notes** (birthday, allergy flag, VIP) visible to staff.

### Reservations

- As waitstaff, I see **upcoming reservations** for the next N hours.
- As waitstaff, I **check in** a guest when they arrive.
- As waitstaff, I **assign seating** if the reserved table is unavailable.
- As waitstaff, I mark **no-show** after grace period.
- As waitstaff, I open an **order** from a seated reservation.

### Orders

- As waitstaff, I **create a dine-in order** for a seated table (existing or walk-in customer).
- As waitstaff, I **add/modify items** on an active order before it is preparing.
- As waitstaff, I **send order to kitchen** (`pending` → `preparing`).
- As waitstaff, I **track preparation** status on my tables.
- As waitstaff, I receive **ready notifications** and mark items **served**.
- As waitstaff, I **request bill** (notify manager / flag order).
- As waitstaff, I **record payment** (cash or initiate customer Mobile Money) and close the table.

### Guest management

- As waitstaff, I look up a guest by name, phone, or reservation.
- As waitstaff, I view **order history** at this restaurant.
- As waitstaff, I see **preferences and allergies** when stored on the customer profile (future-safe; display empty state when absent).

### Messaging

- As waitstaff, I **message the manager** via restaurant chat thread.
- As waitstaff, I **message kitchen** via shared staff channel or order-scoped note (MVP: restaurant staff chat + order context).
- As waitstaff, I **message the customer** when they have an active order or reservation thread.

### Shift management

- As waitstaff, I see **shift activity** (orders served, tables turned, avg serve time — MVP: counts from today's data).
- As waitstaff, I see **assigned tasks** from manager (MVP: optional; Phase 2 if `ShiftTask` entity deferred).

---

## Navigation

Waitstaff shell mirrors manager: sidebar (desktop) + bottom/top nav (mobile/tablet).

```
/waitstaff                          → Overview (shift dashboard)
/waitstaff/tables                   → Tables (default: floor view)
/waitstaff/tables/list              → Tables list view
/waitstaff/tables/[tableId]         → Table detail + actions
/waitstaff/reservations             → Reservations timeline
/waitstaff/reservations/[id]        → Reservation detail + check-in
/waitstaff/orders                   → Active orders board
/waitstaff/orders/[id]              → Order detail + actions
/waitstaff/orders/new               → Create order (table context)
/waitstaff/guests/[customerId]      → Guest profile
/waitstaff/chat                     → Chat inbox (reuse ChatInbox)
/waitstaff/chat/[conversationId]    → Conversation (mobile detail)
/waitstaff/notifications            → Notifications inbox
/waitstaff/shift                    → Shift summary & metrics
```

### Nav items (primary)

| Label | Route | Icon family |
|-------|-------|-------------|
| Overview | `/waitstaff` | LayoutDashboard |
| Tables | `/waitstaff/tables` | LayoutGrid |
| Orders | `/waitstaff/orders` | ClipboardList |
| Reservations | `/waitstaff/reservations` | Calendar |
| Chat | `/waitstaff/chat` | MessageSquare |
| More | sheet/menu | Notifications, Shift, Profile, Sign out |

---

## Screens

### 1. Overview (`/waitstaff`)

**Purpose:** At-a-glance shift control center.

**Sections:**

- Greeting + restaurant name
- KPI chips: Active tables | Orders in kitchen | Ready to serve | Arrivals (30 min)
- **Assigned tables** carousel (or all floor tables if no assignment model in MVP)
- **Ready orders** list (quick "Mark served")
- **Upcoming reservations** (next 3)
- **Alerts** feed (notifications condensed)

**States:** loading skeleton, empty shift, error retry.

### 2. Tables — Floor view (`/waitstaff/tables`)

**Purpose:** Spatial/table status map.

**Layout:** Sections as tabs or accordion; tables as cards colored by status.

**Table card shows:** label, seats, status, party size, elapsed time, active order badge.

**Actions (context menu):** Seat walk-in, View detail, Transfer, Add note, Start order.

### 3. Tables — List view (`/waitstaff/tables/list`)

Sortable/filterable table: status, section, server, party size, timer.

### 4. Table detail (`/waitstaff/tables/[tableId]`)

- Status timeline
- Active reservation (if any)
- Active order(s) with status
- Notes list (add/edit)
- Primary actions: Seat, Start order, Send to kitchen, Request bill, Mark cleaning done, Transfer

### 5. Reservations (`/waitstaff/reservations`)

- Tabs: Upcoming | Checked in | No-show | All today
- Row: time, guest name, party, table, status, actions (Check in, Seat, No-show)

### 6. Reservation detail (`/waitstaff/reservations/[id]`)

- Guest info, special requests (future field), table, payment status
- Actions: Check in, Change table, Seat & open order, Cancel (manager only)

### 7. Orders board (`/waitstaff/orders`)

- Columns or tabs: Draft | Sent (preparing) | Ready | Served
- Filter: my tables, all floor
- Card: table, items summary, elapsed, status

### 8. Order detail (`/waitstaff/orders/[id]`)

- Line items, modifiers, instructions
- Status stepper
- Actions by state:
  - Draft/pending: Edit items, Send to kitchen, Cancel
  - Preparing: View only (kitchen owns)
  - Ready: Mark served
  - Served/unpaid: Request bill, Record payment
  - Paid: Close table session

### 9. Create order (`/waitstaff/orders/new?tableId=`)

- Customer search (phone/email/name) or walk-in guest flow
- Menu browser (reuse manager/customer menu components, read-only prices)
- Cart summary → create `dine-in` order with `tableNumber` / `tableId`

### 10. Guest profile (`/waitstaff/guests/[customerId]`)

- Name, contact, visit count, last visit
- Order history at restaurant (paginated)
- Preferences/allergies section (empty state until profile fields exist)

### 11. Chat (`/waitstaff/chat`)

Reuse `ChatInbox` with `restaurantId` context — same as manager.

### 12. Notifications (`/waitstaff/notifications`)

Reuse `NotificationInbox` pattern from manager.

### 13. Shift (`/waitstaff/shift`)

- Today's stats: tables served, orders completed, avg time ready→served
- Activity log (order/reservation events involving current user)

---

## Components

### New (waitstaff-specific)

| Component | Responsibility |
|-----------|----------------|
| `WaitstaffLayout` | Sidebar, top nav, notification/chat live hooks |
| `WaitstaffNav` / `waitstaffNav.ts` | Nav config |
| `ShiftOverviewCards` | KPI metrics |
| `TableFloorGrid` | Sectioned table cards |
| `TableStatusBadge` | Consistent status colors |
| `TableDetailPanel` | Table context + actions |
| `SeatWalkInDialog` | Party size, optional guest link |
| `ReservationCheckInDialog` | Arrival confirmation |
| `TransferTableDialog` | Target table + optional waiter |
| `TableNotesList` | CRUD notes |
| `WaitstaffOrderBoard` | Kanban/tabs by status |
| `WaitstaffOrderDetail` | Actions + item list |
| `StaffMenuPicker` | Add items to order (wraps menu APIs) |
| `GuestLookupCombobox` | Search customers |
| `RecordPaymentDialog` | Cash / send pay link to customer |
| `RequestBillButton` | Flags order + notifies manager |

### Reuse (existing)

| Component | Source |
|-----------|--------|
| `ChatInbox`, `ChatNavLink` | `components/Chat/` |
| `NotificationInbox`, `NotificationBell` | notifications components |
| UI primitives | `components/ui/*` |
| `useOrdersList`, `useRestaurantId` | hooks |
| `useRestaurantReservations`, `useTablesByRestaurant` | hooks |
| Order normalize/types | `services/order.ts`, `type/Order.ts` |
| Reservation formatters | `lib/reservationDisplay.ts` |

---

## API requirements

### Principles

- Extend existing modules; no duplicate order/table services.
- Enforce `FirebaseGuard` + `RolesGuard` with `RestaurantRole.WAITSTAFF` (and MANAGER where appropriate).
- Validate `restaurantId` matches JWT claim or `UserRestaurantRole`.
- Swagger annotations on all new/changed endpoints.

### A. Table sessions (new)

Operational state for FOH — avoids overloading `Table` config entity.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/restaurants/:restaurantId/floor` | Tables + session status + active order summary |
| POST | `/table-sessions` | Open session (seat walk-in or reservation) |
| PATCH | `/table-sessions/:id` | Update status, notes, assign waitstaff |
| POST | `/table-sessions/:id/transfer` | Move to another table |
| POST | `/table-sessions/:id/close` | End session (paid, cleaning) |

**TableSession fields (proposed):**

`id`, `tableId`, `restaurantId`, `status` (enum), `partySize`, `waitstaffId`, `reservationId?`, `customerId?`, `notes` (jsonb), `seatedAt`, `closedAt`

### B. Reservations (extend)

| Change | Detail |
|--------|--------|
| Status enum | Add `checked_in`, `seated`, `no_show`, `completed` |
| PATCH | Allow waitstaff transitions with validation |
| POST | `/reservations/:id/check-in` | Convenience endpoint |
| POST | `/reservations/:id/no-show` | Convenience endpoint |

### C. Orders (extend)

| Change | Detail |
|--------|--------|
| DTO | Add `waitstaffId`, `tableId` (optional UUID alongside `tableNumber`) |
| POST | `/orders` — allow waitstaff to create for restaurant customers |
| PATCH | Status transitions scoped to restaurant role |
| POST | `/orders/:id/send-to-kitchen` | `pending` → `preparing` (alias) |
| POST | `/orders/:id/mark-served` | `ready` → `delivered` for dine-in |
| POST | `/orders/:id/request-bill` | Sets flag + notifies manager |
| POST | `/orders/:id/record-cash-payment` | Staff records cash (sets `paymentStatus: paid`) |

**Authorization:** Verify caller has `WAITSTAFF` or `MANAGER` for `order.restaurantId`.

### D. Guests (read)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/restaurants/:restaurantId/guests/search?q=` | Lookup customers |
| GET | `/restaurants/:restaurantId/guests/:customerId` | Profile + stats |
| GET | `/restaurants/:restaurantId/guests/:customerId/orders` | History |

Uses existing `User` + `Order` — no new guest entity.

### E. Shift / metrics (read)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/restaurants/:restaurantId/waitstaff/shift-summary` | Aggregates for current user today |

### F. Notifications (extend types)

Add notification types: `RESERVATION_ARRIVING`, `ORDER_READY_TABLE`, `BILL_REQUESTED`, `TABLE_TRANSFER`.

### G. Realtime (extend)

Emit on existing Socket.io gateway:

- `floor:updated` → room `restaurant:{id}`
- `order:updated` → room `restaurant:{id}` (payload: order id, status, tableId)
- `reservation:updated` → room `restaurant:{id}`

Clients join restaurant room on waitstaff/chef/manager layout mount.

### H. Tables CRUD

**No change** to frozen `tables.controller.ts` for waitstaff MVP. Floor status via `TableSession` only.

---

## Permission matrix

| Action | Waitstaff | Chef | Manager | Customer |
|--------|-----------|------|---------|----------|
| View floor/tables | ✅ | ✅ (read) | ✅ | ❌ |
| Seat / open table session | ✅ | ❌ | ✅ | ❌ |
| Transfer table | ✅ | ❌ | ✅ | ❌ |
| View restaurant reservations | ✅ | ❌ | ✅ | Own only |
| Check-in / no-show | ✅ | ❌ | ✅ | ❌ |
| Create dine-in order | ✅ | ❌ | ✅ | Own (app) |
| Edit pending order items | ✅ | ❌ | ✅ | Own pending |
| Send to kitchen | ✅ | ✅ | ✅ | ❌ |
| Mark preparing → ready | ❌ | ✅ | ✅ | ❌ |
| Mark ready → served | ✅ | ❌ | ✅ | ❌ |
| Request bill | ✅ | ❌ | ✅ | ❌ |
| Record cash payment | ✅ | ❌ | ✅ | ❌ |
| Mobile Money pay | ❌ | ❌ | ✅ | Own order |
| Cancel order | ❌ | ❌ | ✅ | Own (rules) |
| Table config CRUD | ❌ | ❌ | ✅ | ❌ |
| Staff management | ❌ | ❌ | ✅ | ❌ |
| Chat: restaurant inbox | ✅ | ✅ | ✅ | ✅ |
| Chat: customer thread | ✅ | ❌ | ✅ | ✅ |

**Enforcement:** `RolesGuard` + service-level `assertRestaurantAccess(userId, restaurantId)`.

---

## Realtime requirements

| Event | Trigger | Subscribers | Client behavior |
|-------|---------|-------------|-----------------|
| `notification:new` | Existing | Individual user | Bell badge + toast |
| `order:updated` | Order PATCH | Restaurant room | Invalidate `useOrdersList`, update board |
| `floor:updated` | Session change | Restaurant room | Invalidate floor query |
| `reservation:updated` | Reservation PATCH | Restaurant room | Refresh arrivals list |
| `chat:message:new` | Existing | Chat participants | Unread badges |

**Fallback:** 15s polling on overview when socket disconnected (match chat pattern).

**Room join:** Authenticated staff emit `join:restaurant` with JWT; server validates `UserRestaurantRole`.

---

## Non-functional requirements

- **Tablet-first:** Touch targets ≥ 44px; readable in restaurant lighting.
- **Performance:** Floor view loads < 2s on 4G; optimistic UI on status changes.
- **i18n:** en/fr keys under `waitstaff` namespace.
- **Accessibility:** Status not color-only; ARIA on dialogs.
- **Offline:** Show banner; queue actions not required for MVP.

---

## Acceptance criteria

### Authentication & access

- [ ] Invited waitstaff with refreshed token lands on `/waitstaff` after login.
- [ ] Waitstaff cannot access `/manager` routes (middleware).
- [ ] Waitstaff API calls without restaurant role return 403.

### Overview

- [ ] Dashboard shows active tables, ready orders, and upcoming reservations for the signed-in restaurant.
- [ ] Data refreshes within 15s of order/status change (socket or poll).

### Tables

- [ ] Floor view shows all tables with correct status colors.
- [ ] Waitstaff can seat a walk-in and open a table session.
- [ ] Waitstaff can transfer an active session to another table.
- [ ] Table notes persist and appear for other staff.

### Reservations

- [ ] Waitstaff sees today's reservations.
- [ ] Check-in updates status and appears in arrivals.
- [ ] No-show frees the table for seating.

### Orders

- [ ] Waitstaff creates dine-in order linked to table.
- [ ] Send to kitchen moves status to `preparing` and chef board updates.
- [ ] Ready order triggers notification; mark served moves to `delivered` and frees table session.
- [ ] Request bill notifies manager.
- [ ] Cash payment marks order paid.

### Messaging & notifications

- [ ] Chat inbox works at `/waitstaff/chat` with unread badges in nav.
- [ ] Notifications page lists order-ready and reservation alerts.

### Regression

- [ ] Customer checkout, manager portal, chef page, and delivery flows unchanged.
- [ ] No unauthorized table config mutations introduced.

---

## Out of scope (v0.2+)

- Full POS hardware integration
- Tip pooling and payroll
- Multi-restaurant waitstaff (single `restaurantId` claim only)
- Drag-and-drop floor designer
- Item-level KDS bump bars
- Guest allergy database (until `User` profile fields added)

---

## Open questions for review

1. **TableSession vs Table.status** — Prefer session entity (recommended) to avoid touching frozen tables controller?
2. **Walk-in without account** — Create ephemeral guest `User` or require phone lookup?
3. **Cash payment** — Simple status flag or integrate with `Payment` entity + receipt?
4. **Floor lead** — Separate role or manager approval for voids/transfers?
5. **Assignment model** — Section-based assignment in MVP or all staff see full floor?

---

**Next:** Review this spec → approve → proceed to implementation per `GAP_ANALYSIS.md`.
