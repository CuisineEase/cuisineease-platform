# Kitchen Dashboard — Enterprise Architecture Review

**Date:** 2026-06-11  
**Reviewer stance:** Senior product + platform team pre-production gate  
**Inputs:** [SPEC.md](./SPEC.md), [IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md) (v1), [ARCHITECTURE_REVIEW.md](./ARCHITECTURE_REVIEW.md), codebase audit  
**Output:** [IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md) v2 (revised)

---

## Executive verdict

The original K1–K6 roadmap is a **credible MVP skeleton** but is **not enterprise-complete**. It under-specifies:

- Navigation (missing Active Orders, Alerts, Shift, Performance, Help, Activity Log)
- Queue operations (no split/merge, partial completion, undo, saved views)
- Order detail (no recipe, checklist, multi-source notes)
- Realtime taxonomy (single `order:updated` is insufficient at scale)
- Backend service decomposition (monolithic `kitchen-queue.service` will not scale)
- Testing, performance, and offline (deferred too late)
- Manager/waitstaff/customer integration contracts

**Recommendation:** Expand to **K0–K12** with explicit acceptance criteria, exit gates, and a dedicated realtime + data model phase before AI/E2EE.

---

# 1. Phase-by-phase critical review (original K1–K6)

## K1 — Foundation shell

| Aspect | Current coverage | Gaps & risks |
|--------|------------------|--------------|
| **Covers** | Layout, nav config, i18n shell, WS join, profile/settings basics | — |
| **Missing** | Help center, keyboard shortcuts overlay, activity log route, error boundaries, connection status bar, default queue redirect, chef-specific hooks (`useKitchenLive`), loading skeletons, empty-state design system | Chefs land on useless stubs without queue deep-link |
| **Edge cases** | Chef with no `restaurantId` in JWT; multi-restaurant user (should not happen but must 403); session expiry mid-shift | Middleware only checks role prefix |
| **Scalability** | N/A at shell layer | — |
| **UX** | No “rush mode” layout toggle; no tablet-first default | Kitchen tablets need queue as home, not dashboard |
| **Backend** | No kitchen health endpoint | Cannot show “API degraded” banner |
| **Database** | None | — |
| **Realtime** | `join:restaurant` only | No presence registration |
| **Testing** | Not mentioned | Layout smoke E2E required |
| **Improve** | Add `/chef/help`, shortcuts modal (⌘K), connection indicator, redirect `/chef` → `/chef/queue` on tablet breakpoints, `ChefStatusGuard` mirroring manager pending states if chef role suspended |

## K2 — Kitchen Queue

| Aspect | Current coverage | Gaps & risks |
|--------|------------------|--------------|
| **Covers** | Kanban/list, ticket card basics, WS refresh, hold/re-fire, bulk, KDS stub hook, waitstaff notify | — |
| **Missing** | Virtual scrolling, drag-and-drop column moves, course/table grouping, chef claim, fire-later, split/merge, partial item completion, undo window, saved filters/views, duplicate ticket detection, allergen prominence (red banner), audio on new ticket | Rush service breaks without virtualization at 100+ tickets |
| **Edge cases** | Concurrent two chefs mark same order ready; order modified after sent_to_kitchen; session closed while ticket active; delivery + dine-in mixed queue | Race conditions unhandled |
| **Scalability** | Full list fetch `limit: 50` | Enterprise kitchens need cursor pagination + virtual list |
| **UX** | No 48px+ touch mode; no color-blind status patterns | Glove + distance viewing failures |
| **Backend** | `advanceKitchenStatus` exists; no item-level status | Partial completion impossible |
| **Database** | No `order_item_kitchen_status` | Blocks split/partial |
| **Realtime** | Generic `order:updated` | Need granular events + version field on order |
| **Testing** | Not mentioned | Concurrency integration tests mandatory |
| **Improve** | Add `GET /kitchen/queue` aggregated DTO with items, allergens, notes, timers; optimistic UI with rollback; `@tanstack/react-virtual` |

## K3 — Dashboard & stations

| Aspect | Current coverage | Gaps & risks |
|--------|------------------|--------------|
| **Covers** | Station entity, station views, dashboard metrics, auto-routing | — |
| **Missing** | Station capacity enforcement, shift-bound assignments, expeditor role, station overload alerts, manager station CRUD UI, batch groups per station | Auto-routing without capacity causes overload |
| **Edge cases** | Menu item mapped to multiple stations; station offline mid-shift | — |
| **Scalability** | Metrics computed client-side | Must be server aggregates with Redis cache |
| **UX** | Dashboard before queue splits attention | Tablet should default queue; dashboard for expeditor/manager monitor |
| **Backend** | No `KitchenStationService` capacity logic | — |
| **Database** | Missing `kitchen_station_assignments`, `station_capacity`, indexes on `(restaurantId, stationId, status)` | Slow queries at scale |
| **Realtime** | `kitchen:station:updated` planned | Need `station:overload` |
| **Testing** | Not mentioned | Load test station filter with 500 tickets |
| **Improve** | Split “Shift Summary” and “Performance” from dashboard; manager configures stations in `/manager/kitchen` |

## K4 — History, messages, inventory

| Aspect | Current coverage | Gaps & risks |
|--------|------------------|--------------|
| **Covers** | Event timeline, operational chat, inventory read-only, waitstaff notes | — |
| **Missing** | Team chat vs DMs separation, station channels, announcements, pinned prep instructions, inventory “affected tickets” cross-link, full-text search on history | Messages lumped into one route |
| **Edge cases** | E2EE key rotation; chat while offline | — |
| **Scalability** | History unbounded | Partition by date + pagination |
| **UX** | Chat inbox not optimized for one-handed tablet | Reuse waitstaff mobile drawer pattern |
| **Backend** | Chat module lacks `station` conversation type | Schema extension needed |
| **Database** | `order_kitchen_events` without indexes on `(restaurantId, createdAt)` | — |
| **Realtime** | `message:new` only | Need `typing`, `presence` |
| **Testing** | Not mentioned | Chat permission matrix tests |
| **Improve** | Split `/chef/messages` (DMs) and `/chef/team` (channels); defer E2EE to K11 |

## K5 — Analytics & manager monitor

| Aspect | Current coverage | Gaps & risks |
|--------|------------------|--------------|
| **Covers** | Chef analytics page, manager read-only, intervene permission, smart notifications | — |
| **Missing** | Heatmaps, export CSV, comparative periods, SLA thresholds, alert rules engine, manager `/manager/kitchen` monitor view | Analytics stub mirrors broken `analytics.service.ts` |
| **Edge cases** | Timezone boundaries for “peak hour” | Use restaurant timezone from settings |
| **Scalability** | Aggregating 2000 orders client-side (manager pattern) | Materialized views or nightly rollups |
| **UX** | Analytics on same nav weight as queue | Secondary nav group |
| **Backend** | No `KitchenAnalyticsService` | — |
| **Database** | No `kitchen_metrics_daily` rollup table | Heavy queries on raw events |
| **Realtime** | `metrics:update` missing | Dashboard stale |
| **Testing** | Not mentioned | Snapshot tests on metric definitions |
| **Improve** | Add K8 dedicated to analytics + AI; manager monitor in K7 |

## K6 — Advanced

| Aspect | Current coverage | Gaps & risks |
|--------|------------------|--------------|
| **Covers** | Offline, E2EE, AI, TV mode, WCAG, completion report | — |
| **Missing** | WCAG should be **continuous** from K1; offline needed **before** production in many kitchens; stress tests; large-restaurant simulation; security audit doc | Bolting a11y at end fails compliance |
| **Edge cases** | Offline conflict on same order from two devices | Last-write-wins loses data |
| **Scalability** | AI on hot path | Async worker / batch |
| **Improve** | Split into K9 (offline), K10 (performance), K11 (security/E2EE), K12 (QA sign-off); embed a11y in every phase gate |

---

# 2. Navigation completeness

## Recommended navigation (enterprise)

| Route | Purpose | Primary users | Permission | Mobile | Tablet | Desktop |
|-------|---------|---------------|------------|--------|--------|---------|
| `/chef` | Live overview, alerts summary | Expeditor, head chef | `chef:read` | Scroll cards; alerts top | 2-col metrics + alert strip | Full dashboard grid |
| `/chef/queue` | **Primary workspace** — ticket processing | All kitchen staff | `chef:write` status | Single column; bottom actions | 2-col Kanban | 4-col Kanban |
| `/chef/active` | Subset: in-progress only (fast lane) | Line cooks | `chef:write` | Default home option | Split list + detail | Narrow list + detail pane |
| `/chef/stations` | Per-station workload | Station cooks | `chef:read` | Station picker → list | Station tabs | Multi-station grid |
| `/chef/history` | Searchable archive + replay | Head chef, manager | `chef:read` / manager read | Search-first | Filters drawer | Full filter bar |
| `/chef/messages` | DMs (future E2EE) | All staff | chat participant | WhatsApp-style | Same | Master-detail |
| `/chef/team` | Station + kitchen channels | All staff | staff chat | Channel list | Split view | Split view |
| `/chef/alerts` | Priority inbox (kitchen-specific) | All | `chef:read` | Full screen list | Split | Split |
| `/chef/shift` | Shift summary, clock, handoff notes | All | `chef:read` | Single column | Cards | Dashboard widgets |
| `/chef/analytics` | Trends, heatmaps | Head chef, manager | `chef:read` / manager | Simplified charts | Charts | Full analytics |
| `/chef/performance` | Personal + station KPIs | Chefs | `chef:read` self | Cards | Cards | Tables + charts |
| `/chef/inventory` | Read-only stock impact | Prep, head chef | `chef:read` | List | List + detail | Table + impact panel |
| `/chef/activity` | Audit log (actions today) | Head chef, manager | `chef:audit` / manager | Timeline | Timeline | Filterable table |
| `/chef/profile` | Identity (restricted), photo, MFA | Self | self | Full page | Full page | Modal or page |
| `/chef/settings` | Preferences | Self | self | Grouped sections | Same | Sidebar settings |
| `/chef/help` | Shortcuts, workflows, support | All | public within portal | FAQ accordion | Same | Searchable help |
| `/chef/shortcuts` | Keyboard map (or modal) | Desktop cooks | self | Hidden | Optional | ⌘K modal |

**Nav grouping (desktop sidebar):**

1. **Operations:** Queue, Active, Stations, Alerts  
2. **Communication:** Messages, Team  
3. **Insight:** Dashboard, Shift, Analytics, Performance, History, Activity  
4. **Resources:** Inventory  
5. **Account:** Profile, Settings, Help  

**Missing from v1 roadmap:** Active Orders, Alerts, Team Chat, Shift Summary, Performance, Help, Activity Log, Shortcuts.

---

# 3. High-pressure kitchen UX

## Current weaknesses

- Two-button list UI — no allergy surfacing, no timer, no priority color
- Hardcoded English strings — cognitive load under stress
- No audio/haptic on critical events
- No “expeditor mode” vs “line cook mode”

## Recommendations

| Area | Standard |
|------|----------|
| Touch targets | Min **48px** height; primary actions **56px** on queue |
| Color | Status + icon + text (never color alone); allergy = high-contrast red border + icon |
| Timers | Monospace numerals; pulse only when threshold exceeded; configurable warn at 80%/100%/120% |
| Priority | Rush = amber stripe; VIP = purple; allergy = red badge always visible |
| Typography | Min 16px body on tablet; ticket title 20–24px; distance-readable mode in settings |
| Animations | ≤200ms; respect reduced motion; no blocking animations |
| Alerts | Critical = sound + persistent banner; urgent = sound once; normal = visual only |
| Audio | New ticket chime, ready bell, delay alarm — per-category mute |
| Haptic | `navigator.vibrate` on critical (where supported) |
| Auto focus | New rush ticket scrolls into view without stealing focus from active prep |
| Auto sort | Default: priority desc → time asc; user override saved per view |
| Clicks | Start prep = 1 tap; ready = 1 tap from card (no detail required for simple tickets) |
| Scrolling | Virtual list; sticky column headers; compact mode eliminates scroll on 15-ticket stations |

---

# 4. Responsiveness matrix

| Device | Layout behavior |
|--------|-----------------|
| Desktop (≥1280px) | 4-col Kanban; sidebar always visible; optional order detail split pane |
| Laptop (1024–1279px) | 3-col Kanban; collapsible sidebar |
| iPad landscape | 2-col Kanban; bottom nav; queue as default route |
| iPad portrait | 1-col cards; swipe actions (start/ready/hold) |
| Android tablet | Same as iPad; test Chrome WebView |
| Kitchen touchscreen | Full-screen queue; 56px buttons; no hover-only affordances |
| Phone | Queue only + alerts; hide analytics; hamburger nav |
| Foldable | Breakpoint at inner width; dual-pane when unfolded ≥840px |
| TV / kiosk | `/chef/display` — station columns only, no chrome, auto-rotate ready column |
| Landscape kiosk | Horizontal station strip + large timers |

**Anti-pattern:** Scaling down desktop grid — use **mode switch** (Kanban ↔ List ↔ Compact) per breakpoint defaults.

---

# 5. Accessibility (continuous requirement)

| Requirement | Implementation |
|-------------|------------------|
| Keyboard | Full queue traversal; `j/k` move ticket; `Enter` open; `s` start; `r` ready; `?` shortcuts |
| Screen reader | `aria-live="polite"` on queue updates; ticket announces table, items, allergies |
| Focus | Trap modals; restore focus on close; skip link to queue |
| Large fonts | `rem` scaling to 125% without layout break |
| Reduced motion | CSS `prefers-reduced-motion`; disable pulse |
| Color blind | Patterns + labels on all status chips |
| WCAG | Target **2.1 AA**; audit each phase gate |
| Touch a11y | 48px targets; spacing between destructive actions |
| Voice | Optional Web Speech API read-aloud for new tickets (settings) |

**Gate:** No phase closes without axe-core scan on new routes.

---

# 6. Kitchen Queue — feature matrix

| Feature | Priority | Phase | Notes |
|---------|----------|-------|-------|
| Kanban / List / Compact | P0 | K2 | — |
| Virtual scrolling | P0 | K2 | Required for enterprise |
| Station grouping | P0 | K3 | — |
| Course grouping | P1 | K4 | Requires course metadata on items |
| Table grouping | P1 | K2 | Group header in Kanban |
| Chef assignment / claim | P0 | K3 | Prevents double-work |
| Priority assignment | P0 | K2 | Rush/VIP flags from waitstaff |
| Fire later | P1 | K4 | Scheduled `sent_to_kitchen` |
| Hold / Resume | P0 | K2 | Maps to `on_hold` |
| Re-fire | P0 | K2 | New event + notification |
| Duplicate detection | P1 | K4 | Same table duplicate draft warn |
| Merge tickets | P2 | K6 | Same table, same course |
| Split tickets | P2 | K6 | By item or station |
| Partial completion | P1 | K5 | Item-level ready |
| Undo (30s window) | P1 | K5 | Audit revert |
| Est. completion | P1 | K8 | AI/heuristic |
| Prep timeline | P1 | K5 | Per ticket drawer |
| Visual dependencies | P2 | K8 | Item B after item A |
| Auto batching | P2 | K8 | AI |
| Smart ordering | P2 | K8 | AI |
| Filters + saved views | P1 | K4 | Persist per user |
| Bulk actions | P0 | K2 | — |
| Search | P0 | K2 | ID, table, item |
| Drag-and-drop | P1 | K4 | Column + priority reorder |
| Offline queue | P1 | K9 | IndexedDB + sync |
| Optimistic updates | P0 | K2 | With rollback |

---

# 7. Order / ticket detail experience

**Route:** `/chef/queue/[orderId]` or slide-over panel (tablet/desktop).

| Content | Source | Phase |
|---------|--------|-------|
| Line items + mods | Order API | K2 |
| Allergens | Menu item + order notes | K2 |
| Special requests | Order item notes | K2 |
| Recipe instructions | Menu item `preparationNotes` (new field) | K5 |
| Plating guide / photos | Menu media (Firebase) | K5 |
| Cooking video | Optional URL on menu item | K6 |
| Ingredient substitutions | Inventory + menu | K6 |
| Waitstaff / manager / customer notes | Order + session timeline | K4 |
| Internal prep notes | `order_kitchen_notes` | K4 |
| Station progress | Item-level station status | K5 |
| Preparation checklist | Template per menu item | K5 |
| Timers | Per item + ticket | K2 |
| Audit history | `order_kitchen_events` | K4 |
| Pinned prep instructions | Station/menu pin table | K4 |

**UX:** Detail panel must not block queue on tablet — 50/50 split or bottom sheet.

---

# 8. Collaboration (expanded)

| Capability | Channel type | E2EE | Phase |
|------------|--------------|------|-------|
| Private DM | `staff_dm` | Yes (K11) | K7 ops / K11 E2EE |
| Station channel | `kitchen_station` | No | K7 |
| Kitchen-wide | `kitchen_broadcast` | No | K7 |
| Manager broadcast | `staff_operational` | No | K7 |
| Waitstaff request | Order-scoped note + notify | No | K4 |
| @mentions | Parse in chat body | No | K7 |
| Threads | Reply-to message id | No | K7 |
| Reactions | Emoji on message | No | K8 |
| Voice notes | Upload + transcode | No | K8 |
| Images / files | Existing chat attachments | No | K7 |
| Pinned / search / drafts | Chat extensions | No | K8 |
| Task assignment | Link message → ticket claim | No | K6 |
| Request assistance | Ping + optional DM | No | K6 |
| Escalation | Auto-notify manager on SLA breach | No | K8 |
| Typing / presence / unread | WS events | No | K6 |

**Policy:** E2EE DMs inaccessible to admins; operational channels auditable by manager.

---

# 9. Settings (expanded)

All in `/chef/settings` with sections:

| Section | Keys |
|---------|------|
| Appearance | theme, density, high contrast, distance mode |
| Language | instant switch, persist to user prefs |
| Sound | per notification category volumes |
| Notifications | push, in-app, critical override |
| Queue defaults | default view, filters, station, sort |
| Timers | warn thresholds, sound on overdue |
| Accessibility | reduced motion, large text, screen reader hints |
| Tablet mode | full-screen queue, swipe actions |
| Keyboard shortcuts | customizable where safe |
| Privacy | presence visibility, read receipts opt-out |
| Security | password, MFA, sessions, devices |
| Auto refresh | interval when WS disconnected |
| Timezone | display (inherit restaurant default) |

**Identity fields:** read-only on profile — enforced FE + BE (see `.cursor/rules/staff-profile-rbac.mdc`).

---

# 10. Internationalization

| Requirement | Gate |
|-------------|------|
| `chef:` namespace 100% coverage | K1 exit |
| Instant switch via i18next | K1 |
| `Intl.DateTimeFormat` / `Intl.NumberFormat` | K2 |
| XAF currency on ticket totals | K2 |
| Pluralization keys for ticket counts | K2 |
| RTL logical properties audit | K10 |
| No hardcoded strings — CI grep rule | K1 |

---

# 11. Realtime architecture (expanded)

## Event taxonomy (target)

| Event | When | Payload (IDs only) |
|-------|------|---------------------|
| `order:new` | Ticket hits kitchen | orderId, restaurantId, priority |
| `order:accepted` | Chef claims | orderId, chefId |
| `order:started` | → preparing | orderId |
| `order:paused` | → on_hold | orderId, reason |
| `order:ready` | → ready | orderId |
| `order:completed` | Removed from KDS | orderId |
| `order:refire` | Re-fired | orderId |
| `order:delay` | SLA breach | orderId, minutesLate |
| `order:modified` | Items changed post-send | orderId, version |
| `station:update` | Workload change | stationId |
| `alert:new` | Kitchen alert | alertId, priority |
| `message:new` | Chat | conversationId |
| `presence:update` | Chef online/offline | userId, status |
| `typing` | Chat | conversationId, userId |
| `notification:new` | Existing | userId |
| `inventory:update` | Stock change | ingredientId |
| `metrics:update` | Dashboard rollup | restaurantId, period |

## Client behavior

- Subscribe via `join:restaurant` + `join:kitchen:{restaurantId}` (new room for high-volume fanout optimization)
- Reconnect: exponential backoff; replay missed events via `GET /kitchen/sync?since=timestamp`
- Optimistic updates with `order.version` conflict detection
- Offline: queue actions in IndexedDB (K9)

**Backward compat:** Keep emitting `order:updated` as alias during migration.

---

# 12. Backend services (decomposition)

| Service | Responsibility |
|---------|----------------|
| `KitchenQueueService` | Aggregated queue DTOs, filters, pagination, grouping |
| `KitchenStationService` | CRUD stations, capacity, assignments, routing rules |
| `KitchenTimerService` | SLA thresholds, delay detection, cron emit `order:delay` |
| `KitchenAssignmentService` | Claim ticket, assign chef/station, auto-balance |
| `KitchenNotificationService` | Kitchen-specific alert rules + fanout |
| `KitchenAuditService` | Append-only events, activity log, undo support |
| `KitchenAnalyticsService` | Rollups, heatmaps, export |
| `KitchenMetricsService` | Real-time counters for dashboard |
| `KitchenPerformanceService` | Per-chef/station KPIs |
| `KitchenForecastService` | Peak prediction (heuristic) |
| `KitchenAIService` | Batching, sequencing, delay prediction (async worker) |
| `KitchenOfflineSyncService` | Batch replay, idempotency, conflict resolution |
| `KitchenDisplayService` | **Upgrade existing stub** — emit WS, archive completed |

All services live under `backend/src/kitchen/` subfolder; orders remain source of truth for status.

---

# 13. Database design

## New / extended entities

| Entity | Purpose |
|--------|---------|
| `kitchen_stations` | Station config, capacity, sort order |
| `kitchen_station_assignments` | Chef ↔ station for shift window |
| `menu_item_stations` | Routing rules |
| `menu_item_prep` | Recipe, checklist, media refs |
| `order_kitchen_events` | Audit timeline (indexed) |
| `order_kitchen_notes` | Internal notes |
| `order_item_kitchen_status` | Partial completion, station progress |
| `kitchen_alerts` | Alert inbox rows |
| `kitchen_saved_views` | User filter presets |
| `kitchen_shift_logs` | Shift handoff notes |
| `kitchen_metrics_daily` | Rollup aggregates |
| `kitchen_offline_actions` | Idempotency keys for sync |
| `staff_device_keys` | E2EE public keys |
| `chat_conversations` (extend) | Types: `kitchen_station`, `staff_dm`, etc. |

## Indexes (critical)

- `order_kitchen_events (restaurant_id, created_at DESC)`
- `orders (restaurant_id, status, updated_at)` — partial index active kitchen statuses
- `order_item_kitchen_status (order_id, station_id)`

## Migration strategy

- One migration per phase (K2–K11)
- Never break `/orders/:id/kitchen-status` during rollout
- Feature flags via restaurant settings jsonb

---

# 14. Integration matrix

| System | Integration | Phase |
|--------|-------------|-------|
| Waitstaff | send-to-kitchen, ready notify, mod requests, mark served | K2–K4 |
| Manager | station config, monitor, intervene, analytics read | K3, K7 |
| Customer | Indirect via order status / notifications | K2 |
| Reservations | VIP flag on reservation → ticket priority | K4 |
| Inventory | Low stock → alerts + affected tickets | K4 |
| Notifications | Kitchen categories + FCM | K2 |
| Table sessions | Table number, guest allergies on ticket | K2 |
| Delivery | Separate queue filter or column | K4 |
| Payments | Read-only paid/unpaid on ticket (dine-in billing) | K5 |
| Chat | Station channels + DMs | K7 |
| WS gateway | Extended events | K6 |
| i18n | `chef:` namespace | K1 |
| Design system | shadcn + waitstaff shell | K1 |
| RBAC | `RestaurantAccessService` | K0 |
| Reports | Export CSV/PDF from analytics | K8 |

---

# 15. Security (expanded)

| Control | Phase |
|---------|-------|
| Role + restaurant on every route | K0 |
| `order.version` optimistic locking | K2 |
| Rate limit kitchen sync endpoint | K9 |
| Audit all status changes | K4 |
| E2EE DM ciphertext only on server | K11 |
| Session/device revoke on profile | K1 |
| JWT refresh handling on long shifts | K1 |
| Idempotency-Key header on mutations | K2 |
| Security event log on RBAC 403 | K0 |
| Pen test before K12 sign-off | K12 |

---

# 16. Testing strategy

| Layer | Scope | Phase gate |
|-------|-------|------------|
| Unit | Services, state transitions, timer logic | Each backend phase |
| Integration | RBAC, restaurant isolation, WS emit | K2, K6, K11 |
| E2E | Waitstaff → chef → waitstaff loop | K2 exit |
| Realtime | Two clients, same ticket race | K2 |
| Offline | Queue actions offline → sync | K9 |
| Concurrency | 10 chefs, 1 order | K2 |
| Permission | Chef cannot access other restaurant | K0 |
| Tablet | Playwright viewport 1024×768 | K2 |
| a11y | axe on all routes | Every phase |
| Performance | 1000 tickets virtual scroll FPS | K10 |
| Stress | 100 WS clients, 50 events/sec | K10 |
| Network | Drop WS mid-action | K9 |

---

# 17. Performance targets

| Metric | Target |
|--------|--------|
| Queue initial load | < 800ms p95 (50 tickets) |
| WS event to UI update | < 300ms |
| Virtual scroll | 60fps with 2000 cached tickets |
| Analytics query | < 2s p95 (use rollups) |
| Re-renders | Ticket card memoized; invalidate by id |

Techniques: cursor pagination, Redis cache for dashboard metrics, `@tanstack/react-virtual`, React Query `select`, websocket debounce batching (100ms).

---

# 18. AI opportunities (phased)

| Feature | Input | Output | Phase |
|---------|-------|--------|-------|
| Delay prediction | Historical prep times, current load | ETA + alert | K8 |
| Smart batching | Same item across tickets | Batch suggestion card | K8 |
| Prep sequence | Station queues | Recommended order | K8 |
| Station balancing | Load per station | Reassignment suggestion | K8 |
| Peak forecast | Historical orders by hour | Staffing hint | K8 |
| Ingredient warning | Inventory + menu | Affected tickets highlight | K4 |
| Auto priority | VIP + wait time | Priority score | K8 |

All AI outputs are **suggestions** — chef confirms action (no auto status change without setting).

---

# 19. Industry comparison

| Capability | Toast KDS | Square KDS | TouchBistro | CuisineEase target |
|------------|-----------|------------|-------------|-------------------|
| Multi-station routing | ✅ | ✅ | ✅ | K3 |
| Item-level firing | ✅ | ✅ | ✅ | K5 |
| Allergy alerts | ✅ | ✅ | ✅ | K2 |
| Bump bar / keyboard | ✅ | Partial | ✅ | K1 shortcuts |
| Expo screen | ✅ | ✅ | ✅ | K3 `/chef/active` |
| Online ordering merge | ✅ | ✅ | ✅ | K4 delivery filter |
| Prep times | ✅ | ✅ | Partial | K8 |
| Multi-location | ✅ | ✅ | Partial | Single tenant per restaurant (OK) |
| Offline mode | Partial | Partial | ❌ | K9 (differentiator) |
| Staff messaging | Limited | Limited | ❌ | K7 (differentiator) |
| E2EE staff chat | ❌ | ❌ | ❌ | K11 (optional differentiator) |
| Integrated reservations | Partial | Partial | ✅ | K4 VIP flags |
| Analytics | ✅ | ✅ | Partial | K8 |

**Gaps to close for parity:** item-level status, expo mode, prep times, station capacity, keyboard workflow.

**Differentiators to keep:** full CuisineEase dine-in session integration, waitstaff realtime loop, optional E2EE, offline sync.

---

# 20. Revised roadmap

See **[IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md) v2** — phases **K0–K12** with acceptance criteria, dependencies, risks, and documentation gates.

**Summary of phase changes:**

| Old | New |
|-----|-----|
| K1 shell only | K0 prerequisites + K1 expanded shell |
| K2 queue basic | K2 queue enterprise + K5 ticket detail/partial |
| K3 stations | K3 stations + K4 queue advanced |
| K4 history/messages | K7 collaboration + K4 history/audit |
| K5 analytics | K8 analytics + AI |
| K6 advanced blob | Split: K6 realtime, K9 offline, K10 perf, K11 security/E2EE, K12 QA |

**Estimated delivery:** K0–K4 = production MVP; K5–K8 = enterprise parity; K9–K12 = enterprise hardening + sign-off.
