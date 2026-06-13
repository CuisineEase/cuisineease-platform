# SYSTEM CONSISTENCY GAP REPORT

**Date:** 2026-06-13  
**Method:** Three-layer cross-reference: `ai/` specs vs `backend/src/` implementation vs `app/src/` frontend  
**Authority:** `ai/` is the source of truth for expected behavior

---

## 1. Cross-Portal Feature Matrix

| Feature | Customer | Manager | Waitstaff | Chef | Driver | Admin |
|---------|:--------:|:-------:|:---------:|:----:|:------:|:-----:|
| **Notifications inbox** | FULL | FULL | FULL | FULL | FULL | FULL |
| **Notification push (WS)** | PARTIAL (poll 15s) | PARTIAL (poll 15s) | PARTIAL (poll 15s) | PARTIAL (poll 15s) | PARTIAL (poll 15s) | PARTIAL (poll 15s) |
| **Chat** | FULL | FULL | FULL | FULL | FULL | **MISSING** |
| **Chat top-nav badge** | FULL | FULL | INCONSISTENT (text button) | INCONSISTENT (text button) | INCONSISTENT (text button) | **MISSING** |
| **Activity/audit logs** | **MISSING** | **MISSING** | **MISSING** | FULL | FULL | PARTIAL |
| **Global search** | **MISSING** | FULL (Ctrl+K) | **MISSING** | INCONSISTENT (cosmetic) | **MISSING** | **MISSING** |
| **Settings/profile** | FULL | PARTIAL (theme only) | PARTIAL (profile only) | INCONSISTENT (not persisted) | FULL (localStorage) | **MISSING** |
| **Help page** | FULL | **MISSING** | **MISSING** | FULL | FULL | **MISSING** |
| **i18n (EN/FR)** | FULL | FULL | PARTIAL (~28 hardcoded) | PARTIAL (~10 hardcoded) | FULL | PARTIAL (~37 hardcoded) |
| **Language switcher** | FULL | FULL | **MISSING** | **MISSING** | **MISSING** | **MISSING** |
| **Offline support** | **MISSING** | **MISSING** | **MISSING** | PARTIAL (IndexedDB, no UI) | FULL (IndexedDB + banner) | **MISSING** |
| **Realtime (WS)** | PARTIAL (chat only) | PARTIAL (chat + roles) | FULL (floor + chat) | FULL (kitchen + chat) | FULL (delivery + chat) | **MISSING** |
| **Payments** | PARTIAL (MoMo only) | **MISSING** | FULL (cash + session) | **MISSING** | **MISSING** | **MISSING** |
| **Orders lifecycle** | FULL (checkout) | FULL (oversight) | FULL (dine-in) | FULL (kitchen queue) | PARTIAL (delivery only) | **MISSING** |
| **Analytics** | PARTIAL (spend) | PARTIAL (revenue) | **MISSING** | FULL (prep times, CSV) | PARTIAL (stubs) | **MISSING** |

### Key Inconsistencies

1. **NotificationBell i18nNs bug** — Waitstaff, Chef, Driver top-navs pass `i18nNs="manager"` instead of their own namespace. Aria-labels render wrong-language text.
   - Files: `WaitstaffTopNav.tsx:154`, `ChefTopNav.tsx:147`, `DeliveryTopNav.tsx:153`

2. **Admin portal is feature-isolated** — No chat, no search, no settings, no help, no realtime beyond notification polling.

3. **Chef settings not persisted** — `KitchenSettingsForm` uses `useState` only. All settings reset on refresh. No API call, no localStorage.

4. **Chat route naming** — Customer uses `/chats`, all others use `/messages`. Driver has dead redirect from `/chats` to `/messages`.

---

## 2. System Expectations vs Actual State

### DOMAIN: Orders

**Expected (ai/ specs):**
- Unified order lifecycle: `draft → sent_to_kitchen → preparing → ready → served → completed`
- Delivery legacy path: `pending → preparing → ready → delivered`
- Role-based transitions enforced (waitstaff: draft→kitchen, chef: kitchen→ready, waitstaff: ready→served)
- Dine-in orders require active `TableSession`
- Paid delivery/takeout auto-routes to kitchen

**Actual:**
- Backend: All transitions implemented. `createSecured`, `updateSecured`, role-based `validateStatusTransitionForRole` enforced.
- `syncOrderFromDeliveryCompletion` forces order to DELIVERED with `logger.warn` when transition map doesn't allow it.
- Frontend: `normalizeOrder()` maps 15+ snake_case fields. `bill` field formatted as display string `"XAF 19,500"` — dashboard must reverse-parse for revenue.
- Response envelope: Orders controller returns **raw** data (no `apiSuccess()` wrapper). Frontend unwraps defensively with 4-level fallback.

**Gaps:**
- 🟡 `useCreateMenuItem` and `useUpdateMenuItem` do NOT invalidate query keys — menu list won't refresh after create/update
- 🟡 Order normalization creates a formatted `bill` string that dashboard must reverse-parse — fragile coupling
- 🟡 No integration tests for order lifecycle transitions

### DOMAIN: Reservations

**Expected:**
- Full lifecycle: `pending → confirmed → checked_in → seated → completed | no_show | cancelled`
- Customer can cancel within 2h window
- Staff can confirm, check-in, no-show, seat
- Table released on cancel/no-show

**Actual:**
- Backend: `reservation-transitions.util.ts` with validated matrix + unit tests (4/4 pass).
- Frontend: `ReservationActions` shared component used by manager + waitstaff.
- `FloorMaintenanceService` cron every 15min sweeps expired reservations to NO_SHOW.

**Gaps:**
- 🟡 `GET /reservations/table/:tableId/availability` lacks `restaurantId` param (SEC-005) — table UUID enumeration possible
- 🟡 Commented-out `@Roles(SystemRole.USER)` on create and findMy endpoints — relies on global guard only
- 🟢 Reservation payment flow exists but untested end-to-end

### DOMAIN: Delivery

**Expected:**
- One delivery per order (unique constraint)
- Manager assigns driver → driver accepts → pickup → deliver → order syncs to DELIVERED
- Driver chat opens on assignment, closes on terminal status

**Actual:**
- Backend: `1772940000000-EnterprisePhase0` adds unique partial index on `deliveries(orderId)`.
- `syncOrderFromDeliveryCompletion` called from both `DriverService` and `DeliveryService`.
- `ChatService.onDeliveryUpdated` closes chat on terminal states.
- Frontend: `normalizeDelivery` maps snake_case fields. Driver portal has full shift management.

**Gaps:**
- 🟡 Driver performance API returns `distanceKm: 0`, `ratings: null` — stubs not connected
- 🟡 No GPS live tracking — only external Google Maps links
- 🟢 Photo/signature proof — URL fields exist, camera upload not implemented

### DOMAIN: Kitchen Flow

**Expected:**
- Kanban/List/Compact queue modes
- Station-based routing, claim, workload balancing
- Offline sync with IndexedDB
- E2EE for private staff DMs
- Manager read-only monitor

**Actual:**
- Backend: 17 kitchen endpoints, all with `assertChefOrAbove`. `order_kitchen_events` audit trail. Optimistic locking via `orders.version`.
- Frontend: DnD kanban, virtualization, TV display mode, analytics export. E2EE via `kitchenE2EE.ts`.
- WebSocket: Listens to 11+ event names (including legacy aliases).

**Gaps:**
- 🟡 `station:update` and `kitchen:station:updated` are duplicate event names for same action — legacy alias retained by design
- 🟡 `metrics:update` and `kitchen:summary:updated` same duplication
- 🟡 Chef offline IndexedDB exists but no visible sync banner (unlike driver)
- 🟡 `join:kitchen` client emit has no server handler
- 🟡 `kitchen_saved_views` table exists but no API/UI queries it
- 🟢 Station CRUD not in UI (API exists, read-only page)

### DOMAIN: Table Sessions

**Expected:**
- `TableSession` is dine-in source of truth (not `Table.status`)
- Full lifecycle: seat → order → kitchen → serve → bill → pay → close
- Transfer, expire, guest-left actions

**Actual:**
- Backend: 12 table-session endpoints, all with `assertWaitstaffOrAbove`. `TableSession.status` enum with 9 states.
- `FloorMaintenanceService` cron releases expired reservations.
- Frontend: `useFloor` hook with WebSocket + 30s polling. Floor view, table detail, billing, timeline.

**Gaps:**
- 🔴 `tables.controller.ts` write endpoints completely unguarded — class-level `@UseGuards` commented out (SEC-001)
- 🟡 Table session transfer API exists but no UI
- 🟡 Guest search API exists but not wired to seat dialog
- 🟡 `assignedWaitstaffId` not validated against restaurant role (SEC-003)

### DOMAIN: Payments

**Expected:**
- MeSomb Mobile Money (MTN/Orange) + Cash on Delivery
- Customer checkout: digital payment
- Session payments: waitstaff records cash or initiates mobile money
- Auto-close session when billing settled

**Actual:**
- Backend: `MeSombGateway` + `CashOnDeliveryGateway`. `PaymentGatewayResolver` selects by method.
- `payment.service.ts:33` has `// TODO: complete the services`. Lines 92-99 have commented-out `paymentProcessor.processPayment()`.
- `transactionId` column and `transaction` relation commented out in entity.
- Frontend: Payment modal shows ONLY MTN/Orange MoMo. Credit card and COD mapped but unreachable in UI.

**Gaps:**
- 🔴 MeSomb gateway exists but service-level processing is incomplete (TODO + commented code)
- 🔴 Payment entity has commented-out `transactionId` and `transaction` relation — transaction tracking broken
- 🟡 No payment error recovery UI for failed MeSomb transactions
- 🟡 Reservation payment flow untested
- 🟡 Response envelope inconsistent — payment controller returns raw, not `apiSuccess()`

### DOMAIN: Inventory

**Expected:**
- Ingredient-backed stock with quantities, par levels
- Low-stock alerts
- Menu item → ingredient linkage

**Actual:**
- Backend: Migration `1772950000000-IngredientStock` adds `quantityOnHand`, `parLevel`, `unit`.
- Frontend: Manager inventory page with beta banner. CRUD wired to ingredient API.

**Gaps:**
- 🟡 No usage history timeline (depletion on sale)
- 🟡 No low-stock push notifications
- 🟡 Chef inventory page is read-only menu list — not connected to ingredient stock
- 🟢 Category taxonomy deferred

### DOMAIN: Chat System

**Expected:**
- `customer_restaurant` and `customer_driver` conversation types
- Staff synced as participants
- Driver chat only during active delivery
- Image attachments via backend storage

**Actual:**
- Backend: 9 chat endpoints. Global `FirebaseGuard` applies (no `@Public()`). `ensureConversationAccess` auto-syncs staff.
- 6 conversation types defined in enum but only 2 used (`CUSTOMER_RESTAURANT`, `CUSTOMER_DRIVER`). `KITCHEN_STATION`, `KITCHEN_BROADCAST`, `STAFF_OPERATIONAL`, `STAFF_DM` defined but unused.
- Frontend: All portals share `ChatInbox` component. `useChatSocket` uses Socket.io + 5s polling fallback.

**Gaps:**
- 🔴 Admin portal has NO chat access — no route, no nav link, no `useChatLive` in layout
- 🟡 Chat reactions/pin API exists but no frontend UI
- 🟡 4 conversation types defined but unused (kitchen station, broadcast, staff operational, staff DM)
- 🟡 Notification service emits to TWO event names for same action (`notification:read` + `notification`)
- 🟢 Voice notes, reply threads, read receipts — schema fields exist, no UI

---

## 3. Critical Flow Breakdown

### Flow A: Customer → Order → Kitchen → Waitstaff → Delivery → Payment

```
Customer checkout → POST /orders/from-cart
  → Order created (pending/draft)
  → POST /payments (MeSomb or COD)
  → autoRoutePaidOrderToKitchen → status = sent_to_kitchen
  → Chef queue picks up → preparing → ready
  → Manager assigns driver → POST /delivery
  → Driver accepts → pickup → deliver
  → syncOrderFromDeliveryCompletion → order = delivered
```

**Breaks at:**
1. 🔴 **MeSomb payment incomplete** — `payment.service.ts` has TODO + commented processor. If customer pays via MoMo, the gateway call may fail silently.
2. 🟡 **Order status forced to DELIVERED** — `syncOrderFromDeliveryCompletion` logs a warning but proceeds even when transition map doesn't allow it. This can create inconsistent audit trails.
3. 🟡 **No driver notification when kitchen-ready** — `delivery:pickup_ready` event exists but driver may not be subscribed if not on the portal.
4. 🟡 **Customer tracking map uses hardcoded coordinates** when address data is missing.

### Flow B: Reservation → Table Assignment → Check-in → Billing

```
Customer books → POST /reservations (pending)
  → Staff confirms → POST /reservations/:id/confirm
  → Guest arrives → POST /reservations/:id/check-in
  → Waitstaff seats → POST /table-sessions (creates session)
  → Orders placed → POST /orders/staff
  → Kitchen prepares → Chef actions
  → Bill requested → POST /table-sessions/:id/request-bill
  → Payment recorded → POST /table-sessions/:id/payments
  → Session closed → POST /table-sessions/:id/close
```

**Breaks at:**
1. 🟡 **Customer cannot cancel** — FAQ mentions cancel but reservations page has no action button for customer. Backend supports it via `POST /reservations/:id/cancel` with 2h window check.
2. 🟡 **No post-dine review prompt** — Session close doesn't trigger a review request. Reviews exist but are not tied to the close event.
3. 🟡 **Table availability endpoint lacks restaurantId** (SEC-005) — table UUID enumeration reveals availability.
4. 🟢 **Guest search not wired** to seat dialog — walk-in seating is manual only.

### Flow C: Driver Assignment → Delivery Lifecycle

```
Manager assigns → POST /delivery (or PATCH to assign)
  → Status = assigned, assignedAt set
  → Chat conversation opened (customer_driver)
  → Driver accepts → POST .../status {action: "accept"}
  → Pickup → POST .../status {action: "pickup"}
  → In transit → navigate to address
  → Deliver → POST .../status {action: "deliver"} + proof
  → syncOrderFromDeliveryCompletion → order = delivered
  → Chat conversation closed
```

**Breaks at:**
1. 🟡 **No live GPS tracking** — only external Google Maps links. Customer sees hardcoded coordinate fallback.
2. 🟡 **Photo proof not implemented** — URL fields exist in API but no camera upload UI.
3. 🟡 **Offline sync has banner but IndexedDB queue is minimal** — `driverOffline.ts` exists but sync is basic.
4. 🟡 **Driver performance metrics are stubs** — `distanceKm: 0`, `ratings: null`, empty trends.

---

## 4. Prioritized Gap List

### 🔴 CRITICAL (5 items)

| # | Gap | Domain | Impact |
|---|-----|--------|--------|
| C-1 | `tables.controller.ts` write endpoints completely unguarded (SEC-001) | Tables | Cross-tenant floor tampering |
| C-2 | MeSomb payment processing incomplete (TODO + commented code) | Payments | Digital payments may fail silently |
| C-3 | Payment entity `transactionId` + `transaction` relation commented out | Payments | No transaction tracking |
| C-4 | Admin portal has NO chat access | Chat | Platform admins cannot communicate |
| C-5 | Debug endpoints `/test` and `/yes` exist without role checks | Security | Information leakage (requires JWT but no role check) |

### 🟡 NORMAL (28 items)

| # | Gap | Domain | Portal |
|---|-----|--------|--------|
| N-1 | NotificationBell i18nNs bug (waitstaff/chef/driver use "manager") | i18n | Waitstaff, Chef, Driver |
| N-2 | Notification transport is polling-only (15s) — `useNotificationSocket` unused | Realtime | All |
| N-3 | Chat top-nav inconsistency (icon vs text button) | Chat | Waitstaff, Chef, Driver |
| N-4 | Chat route naming (`/chats` vs `/messages`) | Chat | Customer vs others |
| N-5 | No language switcher in waitstaff/chef/driver/admin | i18n | 4 portals |
| N-6 | Heavy `defaultValue` reliance (Dashboard ~37, Waitstaff ~28) | i18n | Dashboard, Waitstaff |
| N-7 | Chef settings not persisted (useState only) | Settings | Chef |
| N-8 | No help page in manager/waitstaff/admin | Help | 3 portals |
| N-9 | No settings/profile in admin | Settings | Admin |
| N-10 | No global search in customer/waitstaff/driver/admin | Search | 4 portals |
| N-11 | Chef queue search is cosmetic (no handler) | Search | Chef |
| N-12 | Activity/audit logs missing in customer/manager/waitstaff | Activity | 3 portals |
| N-13 | Analytics missing in waitstaff/admin | Analytics | 2 portals |
| N-14 | `useCreateMenuItem` / `useUpdateMenuItem` don't invalidate query keys | Menu | Manager |
| N-15 | Order `bill` field is formatted string — dashboard reverse-parses | Orders | Manager |
| N-16 | Response envelope inconsistent (apiSuccess vs raw) | API | Backend-wide |
| N-17 | snake_case normalization inconsistent across services | API | Frontend-wide |
| N-18 | Duplicate WS event names (`station:update` / `kitchen:station:updated`) | Realtime | Kitchen |
| N-19 | Duplicate notification emit (`notification:read` + `notification`) | Realtime | Notifications |
| N-20 | Table session transfer API exists but no UI | Floor | Waitstaff |
| N-21 | Guest search API exists but not wired to seat dialog | Floor | Waitstaff |
| N-22 | `assignedWaitstaffId` not validated against restaurant role (SEC-003) | Security | Floor |
| N-23 | Table availability lacks `restaurantId` param (SEC-005) | Security | Reservations |
| N-24 | `join:kitchen` client emit has no server handler | Realtime | Kitchen |
| N-25 | 4 chat conversation types defined but unused | Chat | Backend |
| N-26 | Chat reactions/pin API exists but no frontend UI | Chat | All |
| N-27 | Driver performance metrics are stubs | Delivery | Driver |
| N-28 | `Restaurant.status` type includes `"approved"` but code uses `"active"` | Types | Frontend |

### 🟢 LOW (15 items)

| # | Gap | Domain | Portal |
|---|-----|--------|--------|
| L-1 | Customer header height differs (7.25rem vs 4.5rem) | Layout | Customer |
| L-2 | Chef offline IndexedDB has no visible sync banner | Offline | Chef |
| L-3 | `kitchen_saved_views` table orphaned (no API/UI) | Kitchen | Chef |
| L-4 | Payment methods hardcoded to MoMo (card/COD unreachable) | Payments | Customer |
| L-5 | No post-dine review prompt on session close | Reviews | Customer |
| L-6 | No GPS live tracking for deliveries | Delivery | Driver, Customer |
| L-7 | Photo/signature proof not implemented | Delivery | Driver |
| L-8 | No inventory usage history | Inventory | Manager |
| L-9 | No low-stock push notifications | Inventory | Manager |
| L-10 | Stale `type/auth.ts/user.ts` directory with unused types | Types | Frontend |
| L-11 | Dead code in `services/restaurant.ts` (lines 1-36) | Services | Frontend |
| L-12 | Empty `lib/auth/options.ts` file | Lib | Frontend |
| L-13 | `warmBackend()` calls `GET /test` (debug endpoint) | Lib | Frontend |
| L-14 | Notification cleanup cron not implemented | Notifications | Backend |
| L-15 | `husky` in backend `dependencies` instead of `devDependencies` | Packages | Backend |

---

## 5. System Maturity Score

| Layer | Score | Rationale |
|-------|-------|-----------|
| **Backend maturity** | **82%** | All core modules implemented, RBAC mostly solid (except tables), migrations complete, WebSocket taxonomy aligned. Deducted for: unguarded tables controller, incomplete payment service, inconsistent response envelopes, debug endpoints. |
| **Frontend maturity** | **76%** | All 6 portals have shells, i18n architecture proven, shared components (ChatInbox, NotificationInbox) ensure baseline consistency. Deducted for: i18n gaps (dashboard ~37, waitstaff ~28 hardcoded), inconsistent settings persistence, missing features in admin portal, no centralized envelope unwrapper, query key invalidation gaps. |
| **Cross-system integration** | **71%** | WebSocket events aligned (with legacy aliases), order lifecycle synced across portals, delivery→order sync works. Deducted for: notification polling instead of push, admin portal isolated from chat/realtime, inconsistent API envelopes requiring per-service normalization, no integration test suite. |
| **Overall enterprise readiness** | **76%** | Phase 0 (8/8 critical) and Phase 1 (14/14 high) complete. Core flows (dine-in, delivery, kitchen) work end-to-end. Production-viable for single-restaurant operations. Deducted for: SEC-001 security gap, payment incompleteness, admin portal feature gaps, test coverage gaps. |

---

## 6. Final Conclusion

### Is CuisineEase logically consistent as a single enterprise system?

**Mostly yes, with notable exceptions.**

The system achieves strong consistency in:
- **Order lifecycle** — unified status machine with role-based transitions across all staff portals
- **Chat** — shared `ChatInbox` component, consistent WebSocket events, cross-role conversations
- **Notifications** — shared `NotificationInbox` component, consistent polling pattern
- **Realtime** — canonical WebSocket event taxonomy with intentional legacy aliases
- **i18n architecture** — namespace pattern proven across all 6 portals with mirrored en/fr files

The system is **inconsistent** in:
- **Admin portal** — feature-isolated (no chat, search, settings, help, realtime)
- **Settings persistence** — ranges from server-side (customer) to localStorage (driver) to volatile state (chef)
- **API envelope** — half the backend uses `apiSuccess()`, half returns raw; frontend compensates with 29 different unwrap patterns
- **Notification transport** — all portals poll at 15s despite WebSocket infrastructure existing
- **Search** — only manager has functional global search; chef has a cosmetic input

### Minimum work to make it production-grade unified

**Immediate (blocks production confidence):**
1. Fix SEC-001 (tables controller RBAC) — 1-2 hours
2. Complete MeSomb payment processing — 4-8 hours
3. Add chat to admin portal — 2-3 hours
4. Remove/protect debug endpoints — 30 minutes

**Short-term (unifies cross-portal experience):**
5. Fix NotificationBell i18nNs bug — 30 minutes
6. Wire `useNotificationSocket` for WS push (replace 15s polling) — 4-6 hours
7. Add language switcher to waitstaff/chef/driver/admin — 2 hours
8. Add help page to manager/waitstaff/admin — 2 hours
9. Fix `useCreateMenuItem`/`useUpdateMenuItem` query invalidation — 30 minutes

**Medium-term (enterprise polish):**
10. Standardize API response envelope across all backend modules — 8-12 hours
11. Add integration tests for auth, orders, chat — 16-24 hours
12. Implement chat reactions/pin UI — 4-6 hours
13. Add activity/audit logs to customer/manager/waitstaff — 8-12 hours

**Estimated total: ~60-80 hours of focused work to reach 90+/100 release score.**
