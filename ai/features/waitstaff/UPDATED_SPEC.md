# Waitstaff Portal — Updated Specification

**Version:** 1.0 (implemented)  
**Date:** 2026-06-11  
**Supersedes:** `SPEC.md` v0.1 draft where noted

---

## Status

**Production-complete** for dine-in workflow per approved ADR-013 and Phase 8 completion.

---

## Dine-in workflow (authoritative)

```
Seat Guest (walk-in or reservation)
    ↓
Create Order(s) [draft] — multiple orders per session allowed
    ↓
Send To Kitchen [sent_to_kitchen]
    ↓
Chef: Preparing → Ready
    ↓
Mark Served [served]
    ↓
Request Bill (optional notification to managers)
    ↓
Record Payment (session-level; cash marks PAID immediately)
    ↓
Close Session (requires balance = 0, orders settled)
```

---

## Navigation (implemented)

```
/waitstaff                          Overview
/waitstaff/tables                   Floor grid
/waitstaff/tables/[tableId]         Table detail + session hub
/waitstaff/sessions/[sessionId]/orders/new   Menu picker
/waitstaff/sessions/[sessionId]/billing      Billing + payment + close
/waitstaff/orders                   Order board (all restaurant)
/waitstaff/reservations             Check-in / seat / no-show
/waitstaff/messages                 Chat
/waitstaff/notifications            Notifications
/waitstaff/shift                    Shift metrics
```

---

## Screens

### Table detail (`/waitstaff/tables/[tableId]`)

- Table metadata + status badge
- Active session: guest name, phone, party size, session status
- Order list with per-order actions (send to kitchen, mark served)
- Links: New order, Billing, Request bill
- Guest timeline (chronological events)
- Empty state: Seat walk-in dialog

### Order builder (`/waitstaff/sessions/[sessionId]/orders/new`)

- Searchable menu grid (restaurant menu via manager API)
- Cart with quantity controls
- Creates `draft` order via `POST /orders/staff`
- Returns to table detail on success

### Billing (`/waitstaff/sessions/[sessionId]/billing`)

- Order total, paid total, balance due
- Order and payment line items
- Request bill (when all orders served)
- Record payment dialog (cash, MTN, Orange, card)
- Close session (when `canClose`)
- Timeline

---

## API reference (implemented)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/restaurants/:id/tables/:tableId` | Table + session + billing + timeline |
| GET | `/restaurants/:id/floor` | Floor aggregate |
| GET | `/table-sessions/:id` | Session detail |
| GET | `/table-sessions/:id/billing` | Billing summary |
| GET | `/table-sessions/:id/timeline` | Event timeline |
| POST | `/table-sessions/:id/request-bill` | Request bill + notify staff |
| POST | `/table-sessions/:id/payments` | Record payment |
| POST | `/table-sessions/:id/close` | Close session |
| POST | `/orders/staff` | Create draft order |
| POST | `/orders/:id/send-to-kitchen` | Waitstaff |
| POST | `/orders/:id/mark-served` | Waitstaff |
| POST | `/orders/:id/kitchen-status` | Chef |

---

## Domain model (unchanged from approval)

- **Table** — physical asset + `TableStatus`
- **TableSession** — operational unit; `billRequestedAt`, `assignedWaitstaffId`, notes
- **Guest** — walk-in identity; optional `customerId` link
- **Order** — `draft` … `served` … `completed`; linked via `tableSessionId`
- **Payment** — session-scoped; `recordedBy`, `PAID` for cash

---

## Permission matrix (enforced)

| Action | Waitstaff | Chef | Manager | Customer |
|--------|-----------|------|---------|----------|
| View floor/session | ✓ | ✓ | ✓ | — |
| Seat / session CRUD | ✓ | — | ✓ | — |
| Create draft order | ✓ | — | ✓ | — |
| Send to kitchen | ✓ | — | ✓ | — |
| Kitchen status | — | ✓ | ✓ | — |
| Mark served | ✓ | — | ✓ | — |
| Request bill | ✓ | — | ✓ | — |
| Record cash payment | ✓ | — | ✓ | — |
| Close session | ✓ | — | ✓ | — |

---

## Realtime events

- `floor:updated`
- `table-session:created` / `table-session:updated`
- `order:updated`
- `payment:recorded`
- `reservation:updated`

Clients join via `join:restaurant` with membership validation.

---

## Acceptance criteria (all met)

- [x] Full workflow without leaving waitstaff portal
- [x] Table detail page
- [x] Menu picker / order builder
- [x] Billing page
- [x] Payment dialog
- [x] Session close flow
- [x] Guest timeline
- [x] RBAC audit and fixes on order read/write paths

---

## Future work (out of scope v1.0)

- Split bills
- Section-assigned floor visibility
- Staff-initiated Mobile Money completion
- Manager force-close with audit
- Guest allergy fields on profile
- In-portal table transfer UI

---

## Related documents

- `AUDIT.md` — initial state
- `GAP_ANALYSIS.md` — gap tracking
- `ARCHITECTURE_REVIEW.md` — design validation
- `WAITSTAFF_COMPLETION_REPORT.md` — Phase 8 delivery
- `SECURITY_REVIEW.md` — RBAC and risks
- `ai/DECISIONS.md` — ADR-013
