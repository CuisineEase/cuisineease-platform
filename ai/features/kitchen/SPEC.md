# Kitchen Dashboard — Specification

**Version:** 1.0  
**Date:** 2026-06-11  
**Status:** Implemented — see [KITCHEN_COMPLETION_REPORT.md](./KITCHEN_COMPLETION_REPORT.md)

**Baseline today:** Single page `/chef` with incoming/preparing lists and two status buttons. See [AUDIT.md](./AUDIT.md) and [GAP_ANALYSIS.md](./GAP_ANALYSIS.md).

---

## 1. Vision — complete kitchen workspace

The Kitchen Dashboard is the **operating system for chefs and kitchen staff**, not a read-only ticket list.

| Principle | Requirement |
|-----------|-------------|
| Minimize movement | Primary actions on Kitchen Queue; no context switching for routine work |
| Reduce mistakes | Allergies, modifications, and priorities surfaced on every ticket |
| Improve communication | Internal messages, station chats, waitstaff/manager integration |
| Increase efficiency | Stations, timers, bulk actions, intelligent routing (phased) |
| Production quality | Equal polish to `/manager` and `/customer` — **no placeholder screens** |

Path prefix: **`/chef`** (existing middleware role `chef`). Shell mirrors waitstaff/manager: sidebar + top nav + mobile drawer.

---

## 2. Navigation structure

Config target: `app/src/config/chefNav.ts` (to be created).

### 2.1 Dashboard — `/chef`

Live kitchen overview:

- Orders waiting (`sent_to_kitchen`)
- Orders preparing
- Orders ready
- Delayed orders (timer threshold exceeded)
- Kitchen performance snapshot
- Active stations summary
- Alerts (critical/urgent grouped)
- Staff currently online (kitchen role, WS presence)
- Average preparation time (shift window)
- Throughput statistics (tickets/hour)

### 2.2 Kitchen Queue — `/chef/queue` *(primary working screen)*

| Mode | Layout |
|------|--------|
| Kanban | Columns: Incoming → Preparing → Ready (+ On Hold) |
| List | Dense sortable table |
| Compact | High-volume single-screen |

Features:

- Station grouping filter
- Priority grouping (rush, VIP, allergy-flagged)
- Search (order id, table, item name)
- Filters (status, station, chef, time range)
- Bulk actions (start all, mark ready batch, re-fire)
- Live timers per ticket (since sent_to_kitchen / since preparing)
- Auto refresh (WS `order:updated` + configurable poll fallback)

### 2.3 Stations — `/chef/stations`

Per-station view (Grill, Fryer, Pizza, Bakery, Dessert, Salad, Drinks, Expeditor, custom):

- Assigned chefs
- Capacity / max concurrent tickets
- Current workload count
- Delay indicators
- Estimated completion (computed)
- Station performance metrics

Manager configures station list; chef assigns default station in settings.

### 2.4 Tickets History — `/chef/history`

Searchable archive:

| Filter | Field |
|--------|-------|
| Order ID | `orders.id` |
| Table | session / table number |
| Customer / guest | linked ids |
| Waitstaff | `waitstaffId` |
| Date range | createdAt |
| Station | ticket routing metadata |
| Chef | assigned chef id |
| Status | terminal + in-progress |

Include **replay timeline** (status transitions with timestamps, actor, notes).

### 2.5 Internal Messages — `/chef/messages`

Full staff communication (extends chat module):

| Feature | MVP | Full |
|---------|-----|------|
| 1:1 messaging | ✅ | ✅ |
| Group / station chats | Phase 2 | ✅ |
| Kitchen announcements | Phase 2 | ✅ |
| Manager broadcasts | ✅ (read + reply) | ✅ |
| Attachments / images | ✅ | ✅ |
| Reply threads | Phase 2 | ✅ |
| Read receipts | Phase 2 | ✅ |
| Typing indicators | Phase 2 | ✅ |
| Online status | Phase 2 | ✅ |
| Search / pins / favorites / drafts | Phase 3 | ✅ |
| Voice notes | Phase 3 | ✅ |
| Reactions | Phase 3 | ✅ |

**E2EE (private DMs):** Direct chef↔chef (or staff↔staff) private threads use **client-side encryption**. Server stores ciphertext + routing metadata only. Key exchange and device verification per [SECURITY_REVIEW.md](./SECURITY_REVIEW.md). Manager **audit channels** remain separate, non-E2EE, for operational policy.

### 2.6 Analytics — `/chef/analytics`

- Average cook time by item / station
- Preparation trends (day/week)
- Peak hours
- Most delayed items
- Station efficiency
- Chef productivity (tickets completed)
- Order completion time distribution
- Rush hour analysis
- Heatmaps (hour × station)
- Forecasting (Phase 3 — ML/heuristic)

### 2.7 Inventory Status — `/chef/inventory` *(read-only)*

Quick kitchen view (data from manager inventory module):

- Low stock / out of stock
- Ingredient shortages
- Menu items affected
- Alternative suggestions (manager-configured)
- Kitchen impact summary

### 2.8 Settings — `/chef/settings`

| Area | Options |
|------|---------|
| Theme | light / dark / high contrast |
| Language | EN / FR (instant switch, no reload) |
| Notifications | category toggles, sound |
| Display | density, tablet mode, large text |
| Accessibility | reduced motion, color-blind mode |
| Timers | warning thresholds, sound |
| Shortcuts | keyboard map (desktop) |
| Default station | pre-filter queue |
| Timezone | display only |
| Auto refresh | interval when WS disconnected |

### 2.9 Profile — `/chef/profile`

**Editable by chef (self):**

- Profile picture
- Password / MFA
- Active sessions & devices
- Login history (read-only)
- Notification preferences
- Language

**NOT editable by chef** (manager/admin only):

- Full name
- Email
- Employee ID
- Restaurant assignment
- Job role

Same rule applies to waitstaff, delivery, and other non-manager staff portals.

---

## 3. Language support (i18n)

- Namespace: `chef:` (+ shared `common:`)
- Every label, button, alert, timestamp format localized
- Instant switch via `i18next` — no full page reload
- Persist preference via `/user/notification-preferences` or dedicated user prefs
- RTL-ready layout (logical properties, no hardcoded `ml`/`mr` only)
- **Never hardcode user-facing strings** in `/chef/*`

---

## 4. Responsiveness

| Viewport | Layout |
|----------|--------|
| Desktop | 4-column Kanban, sidebar visible |
| Tablet / kitchen touchscreen | 2-column queue, 44px+ touch targets (glove-friendly) |
| Mobile | Single-column stacked cards, bottom nav |
| Large TV / kiosk | Station overview grid, minimal chrome |
| Foldable | Reflow at breakpoints; test landscape kiosk |

Responsive ≠ shrink-only — queue modes switch by breakpoint defaults.

---

## 5. Offline resilience

| Capability | Behavior |
|------------|----------|
| Local cache | IndexedDB: active queue snapshot, draft messages |
| Offline queue | Status actions queued with client timestamp |
| Sync | Replay on reconnect; server validates version |
| Conflicts | Last-write-wins for status with server audit log |
| UI | Connection banner; degraded read-only history |
| No data loss | Pending actions persisted until ACK |

---

## 6. Accessibility

WCAG 2.1 AA target:

- Full keyboard navigation on queue actions
- Screen reader labels on tickets (table, items, allergies)
- High contrast theme
- Reduced motion respects `prefers-reduced-motion`
- Large font scaling
- Color-blind safe status colors (not color-only)
- Focus trap in modals; skip links

---

## 7. Smart notifications

Categories with priority:

| Priority | Examples |
|----------|----------|
| Critical | Allergy on new ticket, fire suppression |
| Urgent | Rush order, delay > threshold, station overload |
| Normal | New ticket, modification accepted |
| Informational | Shift summary, inventory heads-up |

Grouping + escalation: repeated delays escalate to manager channel. Avoid alert fatigue — batch "3 tickets ready" when appropriate.

Integrate `useNotificationLive` on kitchen layout (same as waitstaff/manager).

---

## 8. Advanced kitchen intelligence (phased)

| Feature | Phase |
|---------|-------|
| Auto station assignment (menu item → station map) | 2 |
| Load balancing across stations | 2 |
| Prep time prediction (historical avg) | 2 |
| Delay prediction | 3 |
| Suggested batching / sequencing | 3 |
| Rush optimization | 3 |
| Chef workload balancing | 3 |
| Auto escalation to manager | 2 |
| Capacity forecasting | 3 |

All intelligence must be **explainable** in UI ("Suggested: Grill — avg 8m for this item").

---

## 9. Kitchen collaboration

- Task assignment between chefs (ticket claim / handoff)
- Request assistance (ping + optional message)
- Hand off ticket to another station/chef
- Request ingredients (notify manager/inventory)
- Notify waitstaff (ready, delay, pause, re-fire)
- Notify managers (escalation)
- @mention users and @mention stations in notes
- Shared notes on ticket (visible on queue card)
- Pinned prep instructions per menu item
- Temporary broadcasts (shift huddle)

---

## 10. Waitstaff integration

Waitstaff receives realtime updates when kitchen:

| Event | Waitstaff UI |
|-------|--------------|
| Preparation starts | Order card → "Preparing" |
| Delayed | Alert + badge |
| Paused / on hold | Status + reason |
| Ready | Notification + floor board |
| Re-fired | Updated ticket |
| Cancelled | Remove from active |
| Modification accepted/rejected | Order detail note |

Waitstaff → kitchen:

- Modification request
- Rush table flag
- Allergy clarification
- Fire now / hold

API: extend order notes + WS `order:updated`; optional `kitchen:request` event.

---

## 11. Manager visibility

Default: **read-only monitoring** in `/manager` (existing orders view + future kitchen monitor).

Optional supervisor permission (`kitchen:intervene`):

- Bump priority
- Reassign station
- Cancel ticket (with reason)

Managers must **not** accidentally change prep state without explicit intervene mode.

---

## 12. Security

- Every endpoint: `FirebaseGuard` + `RestaurantAccessService.assertChefOrAbove`
- Absolute `restaurantId` isolation
- Audit log: status changes, handoffs, message sends (metadata for E2EE)
- E2EE for private staff DMs — see [SECURITY_REVIEW.md](./SECURITY_REVIEW.md)
- Replay protection on queued offline actions (nonce + timestamp window)
- Ownership validation on order/ticket ids
- Security event logging for failed RBAC
- No privilege escalation via client role claims

---

## 13. UX principles

- Speed over decoration
- Clarity over complexity
- Large touch targets (min 44px, prefer 48px on queue)
- Minimal clicks: one tap start, one tap ready
- Minimal scrolling on primary queue
- Fast loading: skeleton + stale-while-revalidate
- Predictable interactions
- Low cognitive load under rush
- Immediate feedback (optimistic UI where safe)
- Consistent motion with reduced-motion fallback

---

## 14. Completion definition

Kitchen Dashboard is **complete** when every item in this spec has:

- [ ] Designed UI (all routes in §2)
- [ ] Backend API + RBAC
- [ ] WebSocket events documented
- [ ] i18n EN + FR
- [ ] Responsive layouts tested (desktop, tablet, mobile, TV)
- [ ] Offline path for queue actions
- [ ] Accessibility audit passed
- [ ] Integration with waitstaff + manager
- [ ] Security review signed off
- [ ] Documented in [FLOWS.md](./FLOWS.md) and [../../workflow/API_MAP.md](../../workflow/API_MAP.md)

---

## Technical anchors (existing)

| Item | Location |
|------|----------|
| Role routing | `app/src/middleware.ts` → `/chef` |
| Order kitchen advance | `POST /orders/:id/kitchen-status` |
| Order list | `GET /orders?statuses=&restaurantId=` |
| Status labels | `app/src/lib/orderStatusViews.ts` |
| WS room | `join:restaurant` → `order:updated` |
| KDS stub | `backend/src/orders/services/kitchen-display.service.ts` |
| Menu read | `GET /menu-item/manage/list` (`menu:read` includes chef) |

---

## Related documents

| Doc | Purpose |
|-----|---------|
| [FLOWS.md](./FLOWS.md) | User journeys |
| [GAP_ANALYSIS.md](./GAP_ANALYSIS.md) | Current vs target |
| [ARCHITECTURE_REVIEW.md](./ARCHITECTURE_REVIEW.md) | Module design |
| [SECURITY_REVIEW.md](./SECURITY_REVIEW.md) | RBAC + E2EE |
| [AUDIT.md](./AUDIT.md) | Current code baseline |
| [IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md) | Phased delivery |
