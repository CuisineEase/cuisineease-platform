# CuisineEase — Tasks

**Last updated:** 2026-06-11 (Enterprise product audit)  
**Purpose:** Actionable backlog aligned with `ROADMAP.md`. Agents and humans should update this file when work starts or completes.

**Legend:** `[ ]` todo · `[~]` in progress · `[x]` done · `[-]` cancelled

---

## Active / recent (2026-06)

### Waitstaff portal (SDD)
- [x] Phase 1: Audit → `ai/features/waitstaff/AUDIT.md`
- [x] Phase 2: Specification → `ai/features/waitstaff/SPEC.md` (approved)
- [x] Phase 3: Gap analysis → `ai/features/waitstaff/GAP_ANALYSIS.md`
- [x] Phase 4: Architecture review → `ai/features/waitstaff/ARCHITECTURE_REVIEW.md`
- [x] Phase 5: Backend (FloorModule, Guest, TableSession, orders workflow, WS rooms)
- [x] Phase 6: Frontend (waitstaff layout, floor, orders, reservations, chat, shift)
- [x] Phase 7–8: Full dine-in workflow (table detail, order builder, billing, payment, close, RBAC)
- [x] Docs: `WAITSTAFF_COMPLETION_REPORT.md`, `SECURITY_REVIEW.md`, `UPDATED_SPEC.md`
- [ ] Production migrations: `1772880000000-WaitstaffFloor`, `1772890000000-TableSessionBillRequested`, `1772900000000-TableSessionExpired`

### Kitchen dashboard (SDD)
- [x] Phase K0 docs + backend prerequisites (migration, version, kitchen module)
- [x] Phase K1 Portal shell & full navigation (`/chef/*`, i18n, layout)
- [x] Phase K2 Kitchen Queue core (Kanban/List/Compact, WS, actions)
- [x] Phase K3 Stations & routing (default stations, workload, claim)
- [x] Phase K4 History & audit (`order_kitchen_events`, undo)
- [x] Phase K5 Ticket detail & partial item status
- [x] Phase K6 Realtime event taxonomy
- [x] Phase K7 Messages, team, alerts
- [x] Phase K8 Analytics & heuristic suggestions
- [x] Phase K9 Offline sync (IndexedDB + `/kitchen/sync`)
- [x] Phase K10 Virtualization, display/TV mode
- [x] Phase K11 RBAC, profile restrictions, E2EE device keys
- [x] Phase K12 Docs + Playwright smoke tests
- [x] **Final polish:** DnD reorder, manager monitor, chat reactions, analytics export
- [x] **Release verification:** builds, E2E, dependency audit — see `RELEASE_VERIFICATION.md`
- [ ] Production migrations: `1772910000000-KitchenDashboard`, `1772920000000-KitchenEnterprisePolish`
- [ ] Staging authenticated E2E: set `E2E_CHEF_TOKEN`, `E2E_MANAGER_TOKEN` in CI

### Driver dashboard (SDD)
- [x] Phase D0 docs → `ai/features/driver/` (13 documents)
- [x] Phase D1 backend — `DriverModule`, migration `1772930000000`, delivery RBAC, realtime
- [x] Phase D2 frontend — `/delivery/*` portal (layout, 12 routes, i18n EN/FR)
- [x] Phase D3 build validation + Playwright auth guards (`e2e/driver.auth.spec.ts`)
- [ ] Production migration: `1772930000000-DriverDashboard`
- [ ] Staging authenticated E2E: set `E2E_DRIVER_TOKEN` in CI
- [ ] Phase D4 polish: IndexedDB offline, camera proof, signature pad

### Chat & messaging
- [x] Backend chat module (entities, API, WebSocket events)
- [x] Frontend chat inbox (customer, manager, driver)
- [x] Fix manager "Cannot message yourself" (staff + USER role ordering)
- [x] Fix staff conversation list/access
- [x] WhatsApp-style mobile layout (notifications + chats)
- [x] Unread badges (sidebar, mobile top bar)
- [x] Socket.io on API port; faster delivery + optimistic send
- [x] Chat list last-message preview + unread styling
- [x] Image upload storage default (Firebase)
- [ ] Verify chat migration applied on production DB
- [ ] Verify `NEXT_PUBLIC_WS_URL` and `STORAGE_TYPE` in production env
- [ ] E2E test: customer messages restaurant → manager sees thread

### Notifications UX
- [x] Bell navigates to notifications page
- [x] Master-detail notification inbox (heading left, body right)
- [ ] Push notification opt-in flow audit (customer settings → FCM token)

### Checkout & orders
- [x] Payment modal (Mobile Money primary)
- [x] Delivery checkout details (address, time)
- [x] Near-me restaurants
- [ ] Confirm MeSomb payment happy path in production

---

## Enterprise product audit (2026-06-11)

Master report: [`ai/ENTERPRISE_PRODUCT_AUDIT.md`](./ENTERPRISE_PRODUCT_AUDIT.md)

### Phase 0 — Critical (before enterprise positioning)

- [x] **ENT-C01** Sync `Order.status` when driver completes delivery — `syncOrderFromDeliveryCompletion`, driver + delivery services
- [x] **ENT-C02** Unify customer dine-in cart with `TableSession` — require active session via `tableId` / `tableSessionId`
- [x] **ENT-C03** Fix manager revenue stats — `delivered`, `completed`, `served` via `orderRevenue.ts`
- [x] **ENT-C04** Auto-route paid delivery orders to kitchen queue — `autoRoutePaidOrderToKitchen` after payment
- [x] **ENT-C05** Complete non-cash session payments — MeSomb in `recordPayment`; auto-close when settled
- [x] **ENT-C06** Align WS events — canonical `kitchen:*` + legacy aliases; `payment:recorded` on floor
- [x] **ENT-C07** Enforce one delivery record per order — validation + migration `1772940000000`
- [x] **ENT-C08** Live restaurant RBAC — `GET /user/me/restaurant-roles` + `useLiveRestaurantAccess`

### Phase 1 — High (post-critical)

- [x] **ENT-H01** Manager driver picker — staff dropdown in deliveries dialog
- [x] **ENT-H02–H03** Reservation lifecycle — transitions, confirm/cancel/check-in/no-show, customer reschedule, shared `ReservationActions`
- [x] **ENT-H04** Staff role edit after invite — manager staff detail PATCH role
- [x] **ENT-H05** Inventory beta + ingredient CRUD — migration `1772950000000`, real API wiring, beta banner
- [x] **ENT-H06** Consume `payment:recorded` WS on waitstaff billing — `useFloor`
- [x] **ENT-H07** Wire driver offline sync — `driverOffline.ts` + sync banner
- [x] **ENT-H08** Waitstaff i18n pass (EN/FR) — overview, orders, billing, shift, reservations
- [x] **ENT-H09** Chef notifications full inbox — `/chef/alerts` with mutate parity
- [x] **ENT-H10** Expose `expireSession` API route — floor controller
- [x] **ENT-H11** Replace stub `AnalyticsService` — structured metrics + logging
- [x] **ENT-H12** Manager global search — `ManagerGlobalSearch` Ctrl+K palette
- [x] **ENT-H13** Notify driver when delivery order is kitchen-ready — `delivery:pickup_ready`
- [x] **Release verification (2026-06-11):** builds pass, enterprise unit tests 6/6 — see [`RELEASE_READINESS_REPORT.md`](../RELEASE_READINESS_REPORT.md). **Push blocked:** uncommitted changes in `backend/` + `app/`; workspace `ai/` not in either Git repo.

## Backlog — Reliability & ops

- [ ] **OPS-001** Document production env checklist (`ai/` or `docs/deploy.md`)
  - `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`, `CORS_ORIGINS`, `FRONTEND_URL`
  - `STORAGE_TYPE=FIREBASE`, Firebase service account on Render
  - Run pending TypeORM migrations
- [ ] **OPS-002** Align `PRODUCTION_FRONTEND_URL` in backend with `www.cuisineease.com`
- [ ] **OPS-003** Decide where `ai/` folder is versioned (app repo, backend repo, or meta-repo)
- [ ] **SEC-001** Audit `RolesGuard` coverage; fix tables controller (guards commented out)
- [ ] **SEC-002** Audit orders/reservations endpoints for restaurant scoping
- [ ] **SEC-003** Remove or secure Postman `connectionType` bypass in WebSocket gateway (dev only?)

---

## Backlog — Payments

- [ ] **PAY-001** Complete `payment.service.ts` gateway processing TODOs
- [ ] **PAY-002** Document supported payment methods in production (COD vs MeSomb)
- [ ] **PAY-003** Frontend error handling for failed MeSomb transactions
- [ ] **PAY-004** Reservation payment flow end-to-end test

---

## Backlog — Quality

- [ ] **QA-001** Integration tests: auth login + refresh
- [ ] **QA-002** Integration tests: create order from cart
- [ ] **QA-003** Integration tests: chat create + send message
- [ ] **QA-004** Add `npm test` to pre-commit or CI pipeline
- [ ] **QA-005** Remove/replace boilerplate `should be defined` specs with behavior tests

---

## Backlog — Restaurant operations

- [ ] **OPS-REST-001** Kitchen display: implement KDS service or remove stub
- [ ] **OPS-REST-002** Chef portal: real-time order updates (socket or polling)
- [ ] **OPS-REST-003** Waitstaff portal: mark served / table assignment
- [ ] **OPS-REST-004** Order analytics service or remove from roadmap

---

## Backlog — Platform admin

- [ ] **ADMIN-001** Expand dashboard beyond restaurant approval
- [ ] **ADMIN-002** User management for platform admins
- [ ] **ADMIN-003** System health / metrics view

---

## Backlog — Documentation hygiene

- [ ] **DOC-001** Replace `backend/README.md` with project-specific setup
- [ ] **DOC-002** Replace `app/README.md` with project-specific setup
- [ ] **DOC-003** Update or archive `backend/mindmap.md` (stale next steps)
- [ ] **DOC-004** Clean `app/.env.example` (remove unused MongoDB/Cloudinary vars or mark legacy)
- [ ] **DOC-005** Keep `ai/*.md` updated when shipping features

---

## Backlog — Specification-driven development

- [x] **SDD-001** Create initial `ai/` operating system (this audit)
- [ ] **SDD-002** Product owner answers open questions in `PROJECT.md`
- [ ] **SDD-003** Define "major implementation" threshold in team practice
- [ ] **SDD-004** Add PR template referencing `ai/TASKS.md` + `DECISIONS.md`

---

## Completed archive (reference)

| ID | Summary | Approx date |
|----|---------|-------------|
| — | Chat module initial ship | 2026-06 |
| — | Chat staff role fix (`914fabe`) | 2026-06 |
| — | Notification + chat mobile UX (`47e3fee`) | 2026-06 |
| — | Chat realtime + storage fix (`dd07b89`, `93926f4`) | 2026-06 |

---

## How to add tasks

```markdown
- [ ] **AREA-NNN** Short title
  - Context: why
  - Acceptance: how we know it's done
  - Files: likely touch points
```

Agents: update status in this file **in the same PR** as the code change.
