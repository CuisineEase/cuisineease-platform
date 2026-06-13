# CuisineEase — Architecture Decision Records (ADR)

**Last updated:** 2026-06-11

Lightweight ADR log. New decisions append at the top. Superseded decisions stay for history.

---

## ADR-017 — Enterprise audit Phase 1 consistency pass

**Date:** 2026-06-11  
**Status:** Accepted (implemented)  
**Context:** Phase 0 fixed critical revenue/workflow breaks; Phase 1 High items remained for reservations, inventory honesty, search, i18n, notifications, and platform admin.  
**Decision:**
- Reservation lifecycle with validated transition matrix, staff/customer actions, and shared `ReservationActions` UI
- Ingredient-backed inventory (beta) with honest CRUD instead of menu-derived fake stock
- Manager global search command palette (Ctrl+K) across workspace entities
- Waitstaff EN/FR i18n for operational pages; reservation status labels in `common`
- Chef notification inbox parity with mark-read/delete/clear
- Platform admin module: overview KPIs, user search, activity feed, system health
- Extract `ReservationStatus` enum to `reservation-status.enum.ts` for testability
**Consequences:** `PHASE1_FINAL_IMPLEMENTATION_REPORT.md`. Normal backlog: inventory usage history, admin broadcasts, server-side search pagination.

---

## ADR-016 — Enterprise audit Phase 0 integrity pass

**Date:** 2026-06-11  
**Status:** Accepted (implemented)  
**Context:** Enterprise product audit identified 8 critical workflow and revenue integrity breaks across delivery, dine-in, payments, WebSocket, and RBAC.  
**Decision:**
- Sync order status on delivery completion from both driver and manager delivery update paths
- Require active `TableSession` for all dine-in orders (resolve via `tableId` at checkout)
- Shared revenue eligibility: `delivered`, `completed`, `served`
- Auto-route paid delivery/takeout to kitchen after customer payment
- Process session mobile money via existing MeSomb gateway; auto-close session when billing allows
- Canonical WebSocket names (`kitchen:station:updated`, `kitchen:summary:updated`, `delivery:pickup_ready`) with legacy aliases
- One delivery per order (validation + DB unique index migration `1772940000000`)
- Live RBAC via `GET /user/me/restaurant-roles` + frontend `useLiveRestaurantAccess`
**Consequences:** Reports in `PHASE0_IMPLEMENTATION_REPORT.md`, `PHASE1_IMPLEMENTATION_REPORT.md`, and `PHASE1_FINAL_IMPLEMENTATION_REPORT.md`. Phase 1 complete per ADR-017.

---

## ADR-015 — Driver portal SDD (DeliveryModule extension)

**Date:** 2026-06-11  
**Status:** Accepted (implemented)  
**Context:** Driver portal was a single list page with broken RBAC (drivers saw all deliveries). Enterprise spec required SDD parity with waitstaff/kitchen.  
**Decision:**
- New `DriverModule` at `restaurants/:restaurantId/driver/*` — scoped to assigned driver via `assertDeliveryOrAbove`
- Fix generic `/delivery` RBAC: filter by `deliveryPerson.id` for DELIVERY role; ownership on PATCH; auto `assigned` on driver assign
- Migration `1772930000000-DriverDashboard`: proof columns, `driver_shifts`, `delivery_events`, `driver_offline_actions`
- Frontend `/delivery/*` shell mirroring chef/waitstaff; i18n namespace `delivery:` EN/FR
- Realtime: `delivery:*` and `driver:shift:updated` via `RestaurantRealtimeService`
- Map workspace uses external navigation links in v1 (no embedded SDK)
**Consequences:** Full docs in `ai/features/driver/`. Production requires migration before deploy. Phase D4 backlog: IndexedDB offline, camera proof, signature UI.

---

## ADR-014 — Waitstaff production security hardening

**Date:** 2026-06-11  
**Status:** Accepted (implemented)  
**Context:** Phase 8 Waitstaff Portal complete; pre-production security audit required multi-tenant isolation and RBAC consistency.  
**Decision:**
- Centralize tenant checks in `RestaurantAccessService` (`assertWaitstaffOrAbove`, `assertRestaurantIdMatch`, `getRestaurantRole`)
- Orders: `createSecured` / `updateSecured` with role-based status transition matrix; staff order paths validate `tableSession` + `guest` belong to `restaurantId`
- Payments: list/detail scoped to payer or restaurant staff; dine-in payments remain session-scoped (`POST /table-sessions/:id/payments`)
- Notifications: user routes require self or `SystemRole.ADMIN`; broadcast/create admin-only
- Reservations: restaurant list requires waitstaff+; updates require staff or owning customer
- WebSocket: `join:restaurant` leaves previous `restaurant:*` room before joining new one
- Document frozen `tables.controller.ts` write risk in `ai/SECURITY_BACKLOG.md` (SEC-001) — not modified per ADR-013
**Consequences:** Full audit in `ai/security/WAITSTAFF_SECURITY_AUDIT.md`. Remaining non-blocking items tracked in `ai/SECURITY_BACKLOG.md`.

---

## ADR-013 — Waitstaff floor model (TableSession + Guest)

**Date:** 2026-06-11  
**Status:** Accepted (implemented)  
**Context:** Waitstaff portal SDD approved after spec review.  
**Decision:**
- `Table` keeps physical config + `TableStatus` (available, occupied, reserved, out_of_service)
- `TableSession` is the operational dine-in unit (statuses: seated → … → completed)
- `Guest` entity for walk-ins; link to `User` customer when phone matches
- Payments recorded via extended `Payment` entity (`tableSessionId`, `recordedBy`, `PAID` status)
- Order dine-in workflow: `draft` → `sent_to_kitchen` → `preparing` → `ready` → `served` → `completed`
- Waitstaff sends to kitchen; chef owns `preparing`/`ready`; legacy delivery flow unchanged (`pending` → `delivered`)
- Full floor visibility for all waitstaff in MVP; `assignedWaitstaffId` on sessions for future section scoping
- Restaurant WebSocket rooms: `restaurant:{id}` + events `floor:updated`, `table-session:*`, `order:updated`, `payment:recorded`
**Consequences:** Migration `1772880000000-WaitstaffFloor` required on deploy. Do not modify frozen `tables.controller.ts`.

---

## ADR-012 — Specification docs live in `ai/` at workspace root

**Date:** 2026-06-12  
**Status:** Accepted (pending repo placement)  
**Context:** Need a single source of truth for agents and humans before major implementations.  
**Decision:** Maintain `ai/PROJECT.md`, `ARCHITECTURE.md`, `ROADMAP.md`, `TASKS.md`, `DECISIONS.md`, `AGENT.md`.  
**Consequences:** Code changes that contradict these docs require a doc update in the same change set.  
**Open:** Which Git repo owns `ai/`?

---

## ADR-011 — Chat realtime via main API Socket.io server

**Date:** 2026-06-12  
**Status:** Accepted  
**Context:** WebSocket on port 3002 was unreachable on Render (single exposed port). Messages felt slow (30s polling fallback).  
**Decision:**
- Attach Socket.io to NestJS HTTP server via `IoAdapter`
- Remove fixed port `3002` from `@WebSocketGateway`
- Frontend `getWebSocketUrl()` uses API host (`NEXT_PUBLIC_WS_URL` optional override)
**Consequences:** `NEXT_PUBLIC_WS_URL` must not include `:3002` in production.  
**Files:** `backend/src/main.ts`, `backend/src/notifications/notifications.gateway.ts`, `app/src/lib/wsUrl.ts`

---

## ADR-010 — Default storage provider Firebase

**Date:** 2026-06-12  
**Status:** Accepted  
**Context:** Chat image upload failed with `No provider registered for type: undefined` when `STORAGE_TYPE` unset.  
**Decision:** `StorageService` defaults to `StorageType.FIREBASE` when env missing.  
**Consequences:** Production should still set `STORAGE_TYPE=FIREBASE` explicitly for clarity.  
**Files:** `backend/src/storage/storage.service.ts`

---

## ADR-009 — Restaurant staff authority before system USER role in chat

**Date:** 2026-06-12  
**Status:** Accepted  
**Context:** Managers keep `SystemRole.USER`; chat treated them as customers → "Cannot message yourself" and missing inbox threads.  
**Decision:**
- `createConversation`: check `isStaffForRestaurant` before `USER` branch
- `listConversations`: `restaurantId` query → restaurant inbox regardless of system role
- `ensureConversationAccess`: sync staff participants before access check
**Consequences:** Users who are both customer and staff see customer chats via participant join; manager inbox uses `restaurantId` filter.  
**Files:** `backend/src/chat/chat.service.ts`

---

## ADR-008 — Chat product rules (locked in product discussion)

**Date:** 2026-06 (session prior to ADR formalization)  
**Status:** Accepted  
**Decisions:**
| Rule | Choice |
|------|--------|
| Customer–restaurant threads | One ongoing thread per pair |
| Who can start | Restaurant staff and customers |
| Driver chat window | Only while delivery assigned and not terminal |
| Driver chat end | Closes on delivered/cancelled/failed |
| Restaurant participation | All staff for that restaurant |
| Attachments | Images supported |

**Consequences:** Backend enums, delivery lifecycle hooks, staff participant sync.  
**Files:** `backend/src/chat/`, `app/src/components/Chat/`

---

## ADR-007 — Dual Git repositories (frontend + backend)

**Date:** Inferred from repo history  
**Status:** Accepted  
**Context:** `app/` and `backend/` are separate repos with independent remotes.  
**Decision:** No monorepo tooling; coordinate releases manually. Frontend also pushes to `TamahJustene/CuisineEase-frontend`.  
**Consequences:** API contract changes require coordinated deploys; `ai/` needs explicit home.

---

## ADR-006 — Firebase as identity provider; Postgres as source of truth

**Date:** Inferred from implementation  
**Status:** Accepted  
**Decision:** Firebase Auth issues JWTs; backend resolves Firebase UID → `users` table UUID for all domain relations.  
**Consequences:** Frontend must send Postgres UUIDs for `customerId`, cart `userId`, etc. — not Firebase UID strings unless they match.  
**Files:** `backend/src/user/user.service.ts`, `app/src/lib/apiUserId.ts`

---

## ADR-005 — API response envelope (partial adoption)

**Date:** Inferred from implementation  
**Status:** Accepted with known inconsistency  
**Decision:** Standard shape `{ success, message, data, meta, errors }` via `apiSuccess` / `HttpExceptionFilter`.  
**Consequences:** Some endpoints return legacy shapes; frontend `unwrap()` assumes envelope when present.

---

## ADR-004 — MeSomb for Mobile Money

**Date:** Inferred from implementation  
**Status:** Accepted (integration incomplete)  
**Decision:** Use `@hachther/mesomb` for MTN/Orange Mobile Money; cash gateway for COD.  
**Consequences:** Cameroon-focused payment UX; card/PayPal deprioritized in frontend modal.

---

## ADR-003 — Socket.io gateway shared for notifications and chat

**Date:** Inferred from implementation  
**Status:** Accepted  
**Decision:** Single `NotificationsGateway` handles notification events and chat events (`chat:message:new`, etc.).  
**Consequences:** Chat module depends on `NotificationsModule`; no separate chat gateway.

---

## ADR-002 — Deny-by-default Firebase guard

**Date:** Inferred from Phase A security work  
**Status:** Accepted  
**Decision:** Global `FirebaseGuard` on all routes; `@Public()` for browse/auth endpoints.  
**Consequences:** New endpoints are protected by default.

---

## ADR-001 — PostgreSQL + TypeORM migrations (no prod synchronize)

**Date:** Inferred from implementation  
**Status:** Accepted  
**Decision:** Schema changes via migrations in `src/lib/database/migrations/`; `synchronize` only in dev when explicitly enabled.  
**Consequences:** Every schema change needs a migration file and production runbook.

---

## Proposed decisions (not yet accepted)

### PDR-001 — Consolidate email modules (`emails` + `mail`)

**Status:** Proposed  
**Rationale:** TODO comments suggest duplication; simplify maintenance.

### PDR-002 — Remove legacy `tasks` module

**Status:** Proposed  
**Rationale:** Appears to be NestJS tutorial boilerplate, unused by frontend.

### PDR-003 — Standardize role guards on all mutating endpoints

**Status:** Proposed  
**Rationale:** Security consistency; tables/orders gaps.

### PDR-004 — E2EE for private staff direct messages (Kitchen)

**Status:** Accepted (implemented RC)  
**Rationale:** Kitchen spec requires encrypted chef↔staff DMs; server stores ciphertext only. Operational/manager audit channels remain non-E2EE.  
**Implementation:** `staff_device_keys`, `/user/me/device-keys`, ECDH P-256 + AES-GCM in `kitchenE2EE.ts`  
**Spec:** `ai/features/kitchen/SECURITY_REVIEW.md`, `ai/security/KITCHEN_SECURITY_AUDIT.md`  
**Consequences:** Device key management, no admin decryption of E2EE threads, separate chat conversation types. Key backup/revocation via device API.

---

## Superseded

### ADR-S1 — WebSocket on dedicated port 3002

**Superseded by:** ADR-011  
**Was:** `@WebSocketGateway(3002, ...)`  
**Why changed:** Production hosting exposes one HTTP port.
