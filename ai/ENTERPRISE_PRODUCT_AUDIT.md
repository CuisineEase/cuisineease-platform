# CuisineEase — Enterprise Product Audit

**Date:** 2026-06-11  
**Method:** Code-verified inspection of `app/` and `backend/` (not documentation alone)  
**Scope:** Customer · Manager · Waitstaff · Kitchen · Driver · Platform Admin  
**Purpose:** Strategic roadmap distinguishing strengths, gaps, and priority order

---

## Implementation status (2026-06-11)

**Phase 0 (Critical):** ✅ All 8 items resolved — see [`PHASE0_IMPLEMENTATION_REPORT.md`](../PHASE0_IMPLEMENTATION_REPORT.md)

**Phase 1 (High):** ✅ All 14 items resolved — see [`PHASE1_FINAL_IMPLEMENTATION_REPORT.md`](../PHASE1_FINAL_IMPLEMENTATION_REPORT.md)

**Deploy:** Run migrations `1772940000000-EnterprisePhase0`, `1772950000000-IngredientStock`.

---

## Executive summary

CuisineEase has **strong foundations** in multi-role portals, restaurant isolation, chat, notifications, waitstaff floor sessions, kitchen KDS, and the new driver portal. The platform is **production-viable for dine-in + kitchen + basic delivery** at a single-restaurant scale.

**Critical gaps** cluster around ~~broken end-to-end delivery revenue tracking~~ **(resolved Phase 0)**, ~~parallel dine-in order paths~~ **(resolved)**, ~~manager revenue undercounting~~ **(resolved)**, and ~~WebSocket event mismatches~~ **(resolved; legacy aliases retained)**.

Remaining **high-priority** debt from the original audit is **resolved** (reservations, waitstaff i18n, manager search, inventory beta, chef notifications, admin expansion).

**Cross-portal inconsistency** remains as **Normal/Low** debt: unified activity logs, notification WS push, shared shift model, and advanced inventory features.

| Priority | Count (this report) | Theme |
|----------|---------------------|-------|
| 🔴 Critical | 8 | Revenue/workflow breaks, RBAC/data integrity — **✅ Phase 0 complete** |
| 🟠 High | 14 | Operational friction, missing workflow steps — **✅ Phase 1 complete** |
| 🟡 Normal | 16 | Consistency, usability, reporting |
| 🟢 Low | 10 | Polish, delight, optional enterprise features |

---

## Cross-dashboard consistency matrix

| Capability | Customer | Manager | Waitstaff | Kitchen | Driver | Admin | Should exist? | Standardize? |
|------------|:--------:|:-------:|:---------:|:-------:|:------:|:-----:|---------------|--------------|
| **Portal shell** (layout, nav, mobile) | ✅ | ✅ | ✅ | ✅ | ✅ | 🟡 minimal | All staff | ✅ Done |
| **i18n (EN/FR)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | All | ✅ Done |
| **Advanced / global search** | 🟡 local | ✅ Ctrl+K | ❌ | ✅ queue | ✅ list | ❌ | Manager + waitstaff + driver | Waitstaff search Normal |
| **Notifications inbox** | ✅ full | ✅ full | ✅ full | ✅ full | ✅ full | ✅ full | All | ✅ Done |
| **Notification push (WS)** | ❌ poll | ❌ poll | ❌ poll | ❌ poll | ❌ poll | ❌ poll | All | Consider P1 |
| **Chat / messages** | ✅ | ✅ | ✅ | ✅ + team dup | ✅ | ❌ | All staff + customer | Remove dup routes |
| **Activity / audit history** | 🟡 chart | 🟡 counters | ❌ | ✅ + dup | 🟡 feed | ❌ | All staff | **Unified event log** |
| **Analytics / KPIs** | ✅ self | ✅ dashboard | ❌ | ✅ + export | 🟡 basic | ❌ | Manager primary; staff lite | **Tiered standard** |
| **Shift management** | ❌ | ❌ | 🟡 counters | 🟡 load only | ✅ clock/break | ❌ | Driver + waitstaff + chef | **Shared shift model** |
| **Offline support** | ❌ | ❌ | ❌ | ✅ sync | 🟡 banner only | ❌ | Mobile staff | **Kitchen pattern for all** |
| **Profile** | ✅ | 🟡 restaurant only | ✅ | ✅ read-only | ✅ read-only | ❌ | All staff | OK |
| **Settings** | ✅ rich | 🟡 theme only | 🟡 in profile | ✅ | ✅ | ❌ | All staff | Extend manager |
| **Help center** | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ | All portals | **Add to all** |
| **Realtime (WebSocket)** | 🟡 chat | 🟡 chat | ✅ floor | ✅ kitchen | ✅ delivery | ❌ | All ops portals | Fix event names |
| **RBAC in UI** | ✅ | 🟡 all-or-nothing | 🟡 | 🟡 | ✅ scoped | ✅ | Fine-grained | **Role-based nav** |
| **Map / navigation** | 🟡 tracking | 🟡 live map | ❌ | ❌ | 🟡 external links | ❌ | Driver + customer | Phase 2 |
| **Performance metrics** | ✅ spend | ✅ revenue* | ❌ | ✅ prep times | 🟡 stubs | ❌ | Manager + roles | Fix revenue calc |
| **Reviews** | ✅ submit | 🟡 read-only | ❌ | ❌ | ❌ | ❌ | Manager reply | Normal |
| **Inventory** | ❌ | 🟡 beta CRUD | ❌ | 🟡 read-only | ❌ | ❌ | Manager real stock | Usage history Normal |
| **Staff management** | ❌ | 🟡 invite only | ❌ | ❌ | ❌ | ❌ | Manager | High |
| **Reservations lifecycle** | ✅ cancel/reschedule | ✅ full | ✅ check-in | ❌ | ❌ | ❌ | All relevant | ✅ Done |
| **Proof of delivery** | ❌ | ❌ | ❌ | ❌ | 🟡 OTP/notes | ❌ | Driver | Normal |
| **E2EE chat** | ❌ | ❌ | ❌ | 🟡 device keys API | ❌ | ❌ | Optional | Low |

\* Manager revenue stats undercount — see Critical finding #3.

---

## Workflow completion audit

### Flow 1 — Dine-in (reservation → close)

```
Reservation → Arrival → Check-in → Seat → Order (draft)
  → Send to kitchen → Preparing → Ready → Served
  → Bill requested → Payment → Close session → (Feedback)
```

| Transition | Status | Gap |
|------------|--------|-----|
| Reservation → seated | ✅ Waitstaff check-in + `TableSession` | Customer cannot self check-in (OK) |
| Seat → staff order | ✅ `/waitstaff/sessions/.../orders/new` | — |
| Draft → kitchen | ✅ `send-to-kitchen` | — |
| Kitchen → ready | ✅ Chef queue | — |
| Ready → served | ✅ Waitstaff `mark-served` | — |
| Served → bill | ✅ `request-bill` | — |
| Payment → close | ✅ Cash works; card/PayPal UI disabled | 🔴 Non-cash session payment incomplete |
| Close → feedback | ❌ No post-dine review prompt tied to session | 🟡 Normal |
| **Parallel path:** customer cart dine-in | 🔴 Creates order with `tableNumber` only — **no `TableSession`** | Bypasses entire waitstaff workflow |
| Session expire | 🔴 `expireSession` exists in service, **no API route** | Stale sessions linger |
| Payment WS refresh | 🟠 `payment:recorded` emitted, **not consumed** | Billing relies on poll/manual refresh |

### Flow 2 — Delivery (order → deliver)

```
Order → Delivery record → Assign driver → Accept → Pickup
  → In transit → Proof → Deliver → (Order complete) → (Feedback)
```

| Transition | Status | Gap |
|------------|--------|-----|
| Customer checkout → delivery | ✅ Order + `createDelivery` | Order stays `pending`, not kitchen |
| Kitchen preparation | 🔴 Delivery orders **never auto-enter kitchen queue** | Food may not be prepared |
| Manager assign driver | 🟡 Works via PATCH but **UUID paste UI** | No driver picker |
| Driver workflow | ✅ Accept → pickup → deliver | — |
| Deliver → order status | 🔴 `DeliveryStatus.DELIVERED` but **`Order.status` not updated** | Revenue/stats broken |
| Manager schedule UI | 🟠 Targets `ready` orders; customer orders often `pending` | Confusing dispatch |
| Duplicate delivery records | 🟠 Manager can create second delivery for same order | Data inconsistency |
| Customer tracking | 🟡 Map with hardcoded coords fallback | Misleading ETA |
| Driver → kitchen coordination | ❌ No "pickup ready" notification to driver | High friction |

### Flow 3 — Takeout / customer order (non-session)

```
Browse → Cart → Pay → (Kitchen?) → Ready → Customer pickup / delivery
```

| Transition | Status | Gap |
|------------|--------|-----|
| Online order → kitchen | 🟠 Depends on order type; delivery path weak | Same as Flow 2 |
| Notification to customer | ✅ Order notifications exist | — |
| Pickup ready | 🟡 Status updates if kitchen used | Not guaranteed for delivery |

---

## Analytics review

### Manager dashboard (`/manager`) — *"Would this help me decide tomorrow?"*

**Partially yes** — good ops snapshot (today's orders, pending deliveries, best sellers, customer map, recent activity counters, CSV export).

**Why it falls short:**
- Revenue counts only `delivered` orders — **misses dine-in `completed`/`served`**
- "Recent activity" is **aggregated counters**, not an auditable timeline
- No week-over-week comparison, no labor cost, no table turn time, no reservation no-show rate
- No drill-down from widget → filtered order list
- Global search bar is **cosmetic** (state never queries)
- Kitchen analytics buried in read-only monitor, not on main dashboard

**Missing KPIs to add:** net revenue by channel (dine-in/delivery/takeout), average table turn, kitchen SLA breach rate, driver on-time %, reservation conversion, staff hours vs covers.

### Customer statistics (`/customer/statistics`) — **Good**

Real client-side analytics from orders, reservations, favorites. Useful for end users. Not a restaurant decision tool (correct scope).

### Kitchen analytics (`/chef/analytics`) — **Good**

Backend-computed from `order_kitchen_events`: prep times, refire rate, station comparison, CSV export. **Best analytics implementation in the product.**

### Driver performance (`/delivery/performance`) — **Limited**

Completion rate and avg minutes are real; `distanceKm: 0`, `ratings: null`, empty trends are stubs. **Not sufficient for fleet management.**

### Platform admin (`/dashboard`) — **None**

Only restaurant approval. No platform-wide metrics.

### Backend `AnalyticsService` — **Stub**

`backend/src/orders/services/analytics.service.ts` — log-only TODOs. Order events listener calls it but **persists nothing**.

---

## Hidden feature discovery

| Capability | Location | UI status |
|------------|----------|-----------|
| Chat message reactions & pin | `backend/src/chat/chat.controller.ts` | ❌ No frontend |
| Broadcast notifications | `POST /notifications/sentToAll` | ❌ Admin only internal |
| Reservation PATCH / no-show | `reservations.controller.ts` | Waitstaff only (not manager/customer) |
| Staff role change after invite | `PATCH /user/restaurant-roles/...` | ❌ No manager UI |
| Kitchen station CRUD | `kitchen.controller.ts` | ❌ Chef stations read-only |
| `fire-later` kitchen action | `kitchen.controller.ts` | ❌ Unused (hold exists) |
| Kitchen saved views | `kitchen_saved_views` table | ❌ Entity injected, never queried |
| Device keys (E2EE prep) | `staff-device-key` API + `kitchen.ts` | ❌ No UI |
| Driver offline sync | `POST .../driver/sync` | ❌ Service fn never called |
| `updateDelivery` PATCH | `app/src/services/delivery.ts` | ❌ Zero callers |
| `expireSession` | `table-sessions.service.ts` | ❌ No HTTP route |
| Guest search | `floor.ts` `searchGuests` | ❌ Not used in seat dialog |
| Table session transfer | `floor.ts` | ❌ No UI |
| `payment:recorded` WS | `restaurant-realtime.service.ts` | ❌ No listener |
| `metrics:update` / `alert:new` WS | backend emit | ❌ Never called or listened |
| `useNotificationSocket` | `app/src/hooks/useNotificationSocket.ts` | ❌ Dead code |
| `join:kitchen` client emit | `useKitchenLive.ts` | ❌ No server handler |
| WS name mismatch | `station:update` vs `kitchen:station:updated` | 🟠 Broken station refresh |
| Tasks CRUD API | `backend/src/tasks/` | ❌ Demo only |
| Customer `/overview` route | `customer/overview/page.tsx` | Stub, not in nav |
| Manager menu import | `manager/menu/page.tsx` | Button with no handler |
| Manager inventory add | `Inventary/ActionButtons.tsx` | Form stub |
| Staff search header | `StaffTableHeader.tsx` | Not wired to table |
| Ingredient CRUD | `ingredient.controller.ts` | Via menu form only |
| `kitchen:summary:updated` listener | `useKitchenLive.ts` | ❌ Backend never emits this name |

---

## Master findings catalog

Findings are ordered by priority. Each includes the required fields.

---

### 🔴 CRITICAL

#### C-01 — Delivery completion does not update order status

| Field | Detail |
|-------|--------|
| **Expected enterprise behavior** | When a driver marks delivered, the linked order becomes `delivered`, triggering revenue recognition, customer notification, and manager dashboard accuracy. |
| **Current CuisineEase behavior** | `DriverService.updateStatus` sets `Delivery.status = delivered` only. `Order.status` unchanged (`backend/src/driver/driver.service.ts`). Manager stats filter `status === "delivered"` in `app/src/services/dashboard.ts`. |
| **Gap** | Revenue, analytics, and customer order history show inconsistent states after successful delivery. |
| **Business impact** | Underreported revenue, wrong customer tracking, reconciliation failures. |
| **Priority** | 🔴 Critical |
| **Recommended solution** | On `deliver` action: set `order.status = delivered`, emit order notification, invalidate manager stats. Add integration test. |
| **Affected portals** | Driver, Manager, Customer |
| **Affected backend** | `driver.service.ts`, `orders.service.ts`, `order-events.listener.ts` |
| **Complexity** | Small |

#### C-02 — Customer dine-in cart bypasses table session model

| Field | Detail |
|-------|--------|
| **Expected** | Dine-in orders attach to an active `TableSession` for billing, kitchen coordination, and close. |
| **Current** | Customer cart checkout sets `tableNumber` only (`customer/cart/page.tsx`); no session join. Order stays outside waitstaff draft → send-to-kitchen flow. |
| **Gap** | Two incompatible dine-in paths; orphan orders on floor. |
| **Business impact** | Unpaid meals, kitchen misses orders, floor state wrong. |
| **Priority** | 🔴 Critical |
| **Recommended solution** | Either (a) disable customer dine-in ordering until session QR/table join exists, or (b) auto-create/join `TableSession` on checkout. |
| **Affected portals** | Customer, Waitstaff, Manager |
| **Affected backend** | `orders.service.ts`, `table-sessions.service.ts`, `cart` flow |
| **Complexity** | Large |

#### C-03 — Manager revenue metrics undercount real sales

| Field | Detail |
|-------|--------|
| **Expected** | Dashboard revenue includes all completed sales (dine-in completed/served + delivered delivery). |
| **Current** | `getManagerOverviewStats` counts only orders with `status === "delivered"`. Dine-in completions excluded. |
| **Gap** | Owners see false low revenue. |
| **Business impact** | Bad business decisions, loss of trust in dashboard. |
| **Priority** | 🔴 Critical |
| **Recommended solution** | Define revenue-eligible statuses enum; update dashboard aggregation; add channel breakdown. |
| **Affected portals** | Manager |
| **Affected backend** | Optional server-side stats endpoint |
| **Complexity** | Small |

#### C-04 — Delivery orders never enter kitchen queue

| Field | Detail |
|-------|--------|
| **Expected** | Delivery/takeout orders flow to kitchen when paid or confirmed. |
| **Current** | Customer delivery orders remain `pending`. Kitchen queue filters `sent_to_kitchen`+ only. No auto-advance for delivery type. |
| **Gap** | Food may never be prepared before driver assignment. |
| **Business impact** | Failed deliveries, customer complaints, driver wait time. |
| **Priority** | 🔴 Critical |
| **Recommended solution** | On paid delivery order: auto `send-to-kitchen` or manager confirmation step; notify kitchen + driver on `ready`. |
| **Affected portals** | Customer, Manager, Kitchen, Driver |
| **Affected backend** | `orders.service.ts`, `kitchen.service.ts`, notifications |
| **Complexity** | Medium |

#### C-05 — Non-cash session payments incomplete

| Field | Detail |
|-------|--------|
| **Expected** | Table session supports card/mobile money with gateway completion. |
| **Current** | `recordPayment` marks non-cash as `PENDING`; gateway block commented in `payment.service.ts`. Card/PayPal UI disabled in cart. |
| **Gap** | Dine-in digital payments broken. |
| **Business impact** | Cash-only dine-in in practice; limits market. |
| **Priority** | 🔴 Critical (for markets requiring digital pay) |
| **Recommended solution** | Complete MeSomb path for session payments; wire `PaymentDetailCard` or remove options. |
| **Affected portals** | Waitstaff, Customer |
| **Affected backend** | `payment.service.ts`, `table-sessions.service.ts` |
| **Complexity** | Large |

#### C-06 — WebSocket station event name mismatch

| Field | Detail |
|-------|--------|
| **Expected** | Station updates invalidate chef station views in real time. |
| **Current** | Backend emits `station:update`; frontend listens `kitchen:station:updated` (`restaurant-realtime.service.ts` vs `useKitchenLive.ts`). |
| **Gap** | Station board stale until manual refresh. |
| **Business impact** | Wrong station workload during rush. |
| **Priority** | 🔴 Critical (operational during service) |
| **Recommended solution** | Align event names; add contract test for WS taxonomy. |
| **Affected portals** | Kitchen, Manager monitor |
| **Affected backend** | `restaurant-realtime.service.ts` |
| **Complexity** | Small |

#### C-07 — Duplicate delivery records possible

| Field | Detail |
|-------|--------|
| **Expected** | One delivery entity per order; idempotent assignment. |
| **Current** | Checkout creates delivery; manager can create another via Deliveries tab for same order. |
| **Gap** | Split driver assignments, confused chat threads. |
| **Business impact** | Data integrity, customer/driver confusion. |
| **Priority** | 🔴 Critical |
| **Recommended solution** | Unique constraint on `orderId` in deliveries; manager UI shows existing delivery; PATCH assign only. |
| **Affected portals** | Manager, Driver, Customer |
| **Affected backend** | `delivery.service.ts`, migration |
| **Complexity** | Medium |

#### C-08 — JWT middleware RBAC vs live restaurant roles

| Field | Detail |
|-------|--------|
| **Expected** | Portal access reflects current `UserRestaurantRole`; revoked staff lose access immediately. |
| **Current** | `middleware.ts` uses JWT claims only; `restaurantId` from token; no live role lookup. Manager sees full sidebar regardless of restaurant role granularity. |
| **Gap** | Stale access after role removal; no fine-grained manager permissions. |
| **Business impact** | Security and compliance risk at scale. |
| **Priority** | 🔴 Critical (multi-location / high staff churn) |
| **Recommended solution** | Short-lived tokens + role refresh; hide nav items by `UserRestaurantRole`; server-side guards already exist — mirror in UI. |
| **Affected portals** | All staff |
| **Affected backend** | Auth claims refresh, `user-restaurant-role.service.ts` |
| **Complexity** | Large |

---

### 🟠 HIGH

#### H-01 — Manager driver assignment UX (UUID paste)

| Expected | Driver picker with availability and active load. |
| Current | `ManagerDeliveriesContent` requires raw driver UUID. |
| Gap | Unusable for non-technical managers. |
| Priority | 🟠 High |
| Solution | Dropdown from `getRestaurantStaff` filtered by DELIVERY role; show shift status. |
| Portals | Manager |
| Backend | `user-restaurant-role`, optional `driver/shift` |
| Complexity | Medium |

#### H-02 — No manager reservation lifecycle (cancel, no-show, edit)

| Expected | Managers handle no-shows, modifications, cancellations. |
| Current | Create + list only; PATCH/no-show APIs used by waitstaff only. |
| Gap | Managers depend on waitstaff for reservation ops. |
| Priority | 🟠 High |
| Solution | Add actions to `/manager/reservations`; reuse waitstaff API calls. |
| Portals | Manager |
| Backend | `reservations.controller.ts` |
| Complexity | Small |

#### H-03 — Customer cannot cancel reservations (FAQ mismatch)

| Expected | Self-service cancel within policy window. |
| Current | Help FAQ mentions cancel; reservations page has no action. |
| Gap | Support burden, trust issue. |
| Priority | 🟠 High |
| Solution | Add cancel button + `PATCH` status; align FAQ. |
| Portals | Customer |
| Backend | `reservations.service.ts` |
| Complexity | Small |

#### H-04 — Staff role edit missing after invite

| Expected | Manager promotes/demotes chef ↔ waitstaff ↔ delivery. |
| Current | Invite + remove only; `updateRestaurantStaffRole` in services unused. |
| Gap | Requires re-invite or admin intervention. |
| Priority | 🟠 High |
| Solution | Role dropdown on staff table + detail page. |
| Portals | Manager |
| Backend | `user-restaurant-role.service.ts` |
| Complexity | Small |

#### H-05 — Inventory page is fake (menu-derived)

| Expected | Stock levels, par levels, depletion on sale, low-stock alerts. |
| Current | `getInventoryFromMenu` derives display from menu; add product form is stub. |
| Gap | Cannot run real inventory ops. |
| Priority | 🟠 High |
| Solution | Wire to `ingredient` entity with quantities; or rename page to avoid false expectations. |
| Portals | Manager |
| Backend | `ingredient` module |
| Complexity | Large |

#### H-06 — `payment:recorded` not consumed by waitstaff billing

| Expected | Billing page updates instantly when payment recorded. |
| Current | WS event emitted; `useFloor` / billing hooks don't listen. |
| Gap | Stale totals during service. |
| Priority | 🟠 High |
| Solution | Add listener in billing page or extend `useFloor`. |
| Portals | Waitstaff |
| Backend | None (emit exists) |
| Complexity | Small |

#### H-07 — Driver offline sync API unwired

| Expected | Driver actions queue offline and replay (like kitchen). |
| Current | `syncDriverOfflineActions` exists; banner only; no IndexedDB queue. |
| Gap | Data loss on mobile dead zones. |
| Priority | 🟠 High |
| Solution | Port `kitchenOffline.ts` pattern to driver; call sync on reconnect. |
| Portals | Driver |
| Backend | `driver/sync` (exists) |
| Complexity | Medium |

#### H-08 — Waitstaff i18n largely hardcoded English

| Expected | Full EN/FR like chef/driver portals. |
| Current | ~15 `waitstaff:` keys; pages use literal strings. |
| Gap | FR waitstaff unusable; inconsistent product quality. |
| Priority | 🟠 High |
| Solution | i18n pass on all waitstaff pages; register keys in `en.json`/`fr.json`. |
| Portals | Waitstaff |
| Backend | None |
| Complexity | Medium |

#### H-09 — Chef notifications read-only vs other portals

| Expected | Consistent inbox with mark-read/delete. |
| Current | Bell → `/chef/alerts` with `canMutate={false}`. |
| Gap | Chefs cannot clear noise; different UX from waitstaff/driver. |
| Priority | 🟠 High |
| Solution | Use `/chef/notifications` with full inbox or enable mutations on alerts. |
| Portals | Kitchen |
| Backend | None |
| Complexity | Small |

#### H-10 — `expireSession` not exposed

| Expected | Auto or manual expire for abandoned sessions. |
| Current | Service method exists; no route; cron not verified. |
| Gap | Tables stuck occupied. |
| Priority | 🟠 High |
| Solution | Add `POST .../expire` for managers; optional scheduled job. |
| Portals | Manager, Waitstaff |
| Backend | `table-sessions.controller.ts` |
| Complexity | Small |

#### H-11 — Order analytics service is a stub

| Expected | Server-side order analytics persisted for reporting. |
| Current | `AnalyticsService` logs only; called from event listener. |
| Gap | No historical analytics backbone. |
| Priority | 🟠 High |
| Solution | Implement or remove listener calls; prefer real aggregates in manager API. |
| Portals | Manager |
| Backend | `orders/services/analytics.service.ts` |
| Complexity | Medium |

#### H-12 — Manager global search non-functional

| Expected | Search orders, customers, menu items, staff from top nav. |
| Current | `AdminTopNav` search state unused. |
| Gap | Wasted UI; users assume broken product. |
| Priority | 🟠 High |
| Solution | Implement unified search API or remove input until ready. |
| Portals | Manager |
| Backend | New search endpoint or client-side aggregate |
| Complexity | Medium |

#### H-13 — Delivery-manager-kitchen coordination

| Expected | Driver notified when order ready for pickup. |
| Current | No `pickup ready` notification wired to driver inbox/WS. |
| Gap | Drivers arrive before food ready. |
| Priority | 🟠 High |
| Solution | On kitchen `ready` for delivery orders: notify assigned driver via notification + `delivery:pickup_ready` WS. |
| Portals | Kitchen, Driver, Manager |
| Backend | `kitchen.service.ts`, `notifications.service.ts` |
| Complexity | Medium |

#### H-14 — Platform admin scope minimal

| Expected | User management, health, audit, feature flags. |
| Current | Restaurant approve/reject/suspend only. |
| Gap | Ops team needs separate tools. |
| Priority | 🟠 High (for SaaS operator) |
| Solution | Phase admin: user search, restaurant detail, migration status. |
| Portals | Admin |
| Backend | Existing admin endpoints underused |
| Complexity | Large |

---

### 🟡 NORMAL

#### N-01 — Activity history inconsistent across portals

| Expected | Chronological audit log per role (orders touched, deliveries, tickets). |
| Current | Manager: counter widgets; Customer: chart; Chef: history; Driver: history feed; Waitstaff: none. |
| Gap | No unified activity model. |
| Priority | 🟡 Normal |
| Solution | Standardize on event-sourced feed component; reuse `order_kitchen_events` / `delivery_events` patterns. |
| Portals | All |
| Complexity | Medium |

#### N-02 — Duplicate chef routes (`/activity` ≡ `/history`, `/team` ≈ `/messages`)

| Expected | Single canonical route per function. |
| Current | Duplicate pages confuse nav and bookmarks. |
| Priority | 🟡 Normal |
| Solution | Redirect duplicates; trim sidebar. |
| Portals | Kitchen |
| Complexity | Small |

#### N-03 — Customer order tracking map fallbacks

| Expected | Real restaurant → customer coordinates. |
| Current | Hardcoded defaults when coords missing (`order-history/[id]`). |
| Priority | 🟡 Normal |
| Solution | Use delivery address + restaurant address from API. |
| Portals | Customer |
| Complexity | Small |

#### N-04 — Manager reviews read-only

| Expected | Reply to reviews, flag, hide abusive content. |
| Current | List only. |
| Priority | 🟡 Normal |
| Solution | Add manager response via review PATCH if backend supports. |
| Portals | Manager |
| Complexity | Small |

#### N-05 — Manager settings thin vs customer

| Expected | Notification prefs, security, language for managers. |
| Current | Theme + restaurant profile only. |
| Priority | 🟡 Normal |
| Solution | Reuse customer settings components where applicable. |
| Portals | Manager |
| Complexity | Medium |

#### N-06 — Waitstaff shift vs driver shift model mismatch

| Expected | Consistent shift concept (clock in, hours, assignments). |
| Current | Waitstaff: two counters; Chef: load summary; Driver: full clock/break. |
| Priority | 🟡 Normal |
| Solution | Shared `ShiftSummary` component backed by role-specific APIs. |
| Portals | Waitstaff, Chef, Driver |
| Complexity | Medium |

#### N-07 — Kitchen station CRUD not in UI

| Expected | Chef/manager configures stations. |
| Current | API exists; stations page read-only; defaults seeded server-side. |
| Priority | 🟡 Normal |
| Solution | Add station edit modal for manager or chef lead role. |
| Portals | Kitchen, Manager |
| Complexity | Medium |

#### N-08 — Chat reactions/pin not surfaced

| Expected | WhatsApp-grade message actions for staff. |
| Current | Backend endpoints exist; ChatInbox doesn't call them. |
| Priority | 🟡 Normal |
| Solution | Add reaction picker + pin UI to ChatInbox. |
| Portals | All chat users |
| Complexity | Medium |

#### N-09 — Driver performance stubs (distance, ratings, trends)

| Expected | Useful fleet KPIs. |
| Current | API returns zeros/nulls for several fields. |
| Priority | 🟡 Normal |
| Solution | Integrate reviews by delivery; compute trends from `delivery_events`. |
| Portals | Driver, Manager |
| Complexity | Medium |

#### N-10 — Manager menu import button dead

| Expected | CSV/menu import for bulk onboarding. |
| Current | Button with no handler. |
| Priority | 🟡 Normal |
| Solution | Implement import or remove button. |
| Portals | Manager |
| Complexity | Medium |

#### N-11 — Help center missing on manager/waitstaff/customer nav

| Expected | In-app FAQ per portal. |
| Current | Chef, driver, customer settings/help only. |
| Priority | 🟡 Normal |
| Solution | Add `/manager/help`, `/waitstaff/help` using shared accordion component. |
| Portals | Manager, Waitstaff |
| Complexity | Small |

#### N-12 — Notifications use poll not push WS

| Expected | Instant notification delivery via socket. |
| Current | `useNotificationLive` polls 15s; backend emits `notification` WS but `useNotificationSocket` unused. |
| Priority | 🟡 Normal |
| Solution | Wire socket listener to invalidate notification queries. |
| Portals | All |
| Complexity | Small |

#### N-13 — Guest search API unused

| Expected | Fast guest lookup when seating walk-ins. |
| Current | `searchGuests` exported; seat dialog uses manual upsert only. |
| Priority | 🟡 Normal |
| Solution | Autocomplete in `SeatWalkInDialog`. |
| Portals | Waitstaff |
| Complexity | Small |

#### N-14 — Table session transfer not in UI

| Expected | Move party between tables. |
| Current | API in `floor.ts`; no UI. |
| Priority | 🟡 Normal |
| Solution | Add transfer action on table detail. |
| Portals | Waitstaff |
| Complexity | Small |

#### N-15 — Accessibility settings not wired to DOM

| Expected | High contrast / large text toggles affect UI. |
| Current | Driver settings save to localStorage only. |
| Priority | 🟡 Normal |
| Solution | Apply document classes from settings (chef/driver shared hook). |
| Portals | Driver, Chef |
| Complexity | Small |

#### N-16 — `kitchen_saved_views` orphaned

| Expected | Saved queue filters per chef. |
| Current | DB table + repo injection; no API/UI. |
| Priority | 🟡 Normal |
| Solution | Implement saved views or drop table from migration scope. |
| Portals | Kitchen |
| Complexity | Medium |

---

### 🟢 LOW

#### L-01 — Embedded maps vs external links (driver/customer)

| Expected | In-app turn-by-turn for drivers. |
| Current | Google Maps external URLs. |
| Priority | 🟢 Low |
| Solution | Map SDK integration phase 2. |
| Complexity | Large |

#### L-02 — E2EE chat device keys UI

| Expected | Optional encrypted staff channels. |
| Current | API + migration; no registration UI. |
| Priority | 🟢 Low |
| Solution | Device key onboarding in chef settings. |
| Complexity | Medium |

#### L-03 — Customer `/overview` orphan stub

| Expected | Remove dead routes. |
| Current | Placeholder page not in nav. |
| Priority | 🟢 Low |
| Solution | Delete route or implement overview. |
| Complexity | Small |

#### L-04 — Chef `/inventory` read-only menu list

| Expected | Kitchen inventory awareness or remove. |
| Current | Duplicates manager menu data loosely. |
| Priority | 🟢 Low |
| Solution | Merge into queue context or drop nav item. |
| Complexity | Small |

#### L-05 — Tasks demo API

| Expected | Remove from production surface. |
| Current | `backend/src/tasks/` unused. |
| Priority | 🟢 Low |
| Solution | Disable module in prod or delete. |
| Complexity | Small |

#### L-06 — Post-dine feedback prompt

| Expected | Review request after session close. |
| Current | Reviews exist but not tied to close event. |
| Priority | 🟢 Low |
| Solution | Modal on session close or email follow-up. |
| Complexity | Small |

#### L-07 — Driver photo/signature proof capture

| Expected | Camera capture for proof. |
| Current | OTP/notes; URL fields in API. |
| Priority | 🟢 Low |
| Solution | Firebase upload + camera component. |
| Complexity | Medium |

#### L-08 — Stacked delivery route optimization

| Expected | Multi-stop routing like Uber Eats. |
| Current | One assignment at a time in UI. |
| Priority | 🟢 Low |
| Complexity | Large |

#### L-09 — Admin chat / notifications for platform ops

| Expected | Admin alerted on restaurant applications. |
| Current | Admin checks dashboard manually. |
| Priority | 🟢 Low |
| Solution | Email already may fire; add admin notification seed. |
| Complexity | Small |

#### L-10 — Consolidate duplicate delivery/chats routes

| Expected | Clean URL space. |
| Current | `/delivery/chats` redirects to messages. |
| Priority | 🟢 Low |
| Solution | Keep redirect; update docs/links. |
| Complexity | Small |

---

## Recommended execution order

### Phase 0 — Critical fixes (before marketing "enterprise")

1. C-03 Manager revenue calculation  
2. C-01 Delivery → order status sync  
3. C-04 Delivery → kitchen flow  
4. C-06 WebSocket event alignment  
5. C-07 One delivery per order  
6. C-02 Dine-in path unification (or explicit disable)  
7. C-05 Session digital payments (market-dependent)  
8. C-08 RBAC token refresh (parallel track)

### Phase 1 — High operational value ✅ complete

H-01 through H-14 — see [`PHASE1_FINAL_IMPLEMENTATION_REPORT.md`](../PHASE1_FINAL_IMPLEMENTATION_REPORT.md)

### Phase 2 — Consistency & reporting

Normal items N-01 through N-16; inventory usage history; platform admin broadcasts; server-side manager search pagination

### Phase 3 — Polish

Low items; embedded maps; E2EE UI; gamification

---

## What CuisineEase already does well

- **Waitstaff floor model** — table sessions, billing, send-to-kitchen, realtime floor WS  
- **Kitchen portal** — queue modes, DnD, offline sync, analytics export, manager monitor  
- **Driver portal** — full shell, workflow, shift, scoped RBAC (recent ship)  
- **Chat** — cross-role inbox, realtime, attachments, delivery threads  
- **Notifications** — persistent inbox, preferences (customer), unread badges  
- **Customer ordering** — cart, MeSomb checkout, favorites, statistics, near-me  
- **Manager ops** — menu CRUD, tables/sections, staff invite, orders/deliveries tabs, kitchen monitor  
- **Security baseline** — Firebase JWT, restaurant access service, tenant isolation on staff APIs  
- **i18n architecture** — namespace pattern proven on chef/driver/customer (when registered)

---

## Document maintenance

| Trigger | Action |
|---------|--------|
| Any portal feature shipped | Update matrix + close finding |
| New WS event | Add to taxonomy doc + audit |
| Quarterly | Re-run workflow verification |

Related docs: `ai/TASKS.md`, `ai/ROADMAP.md`, `ai/features/*/GAP_ANALYSIS.md`, `ai/workflow/SYSTEM_FLOWS.md`

**Next step:** Import Critical + High findings into `ai/TASKS.md` as tracked items (Phase 0–1).
