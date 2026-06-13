# CuisineEase — Definitions & Glossary

**Last updated:** 2026-06-11

Canonical terms used across workflow docs, specs, and code.

---

## Platform terms

| Term | Definition |
|------|------------|
| **CuisineEase** | Multi-restaurant SaaS platform — customers order, restaurants operate, admins govern the network |
| **Tenant** | A **Restaurant** — data is isolated by `restaurantId` |
| **Portal** | Role-specific Next.js route prefix (`/customer`, `/manager`, `/waitstaff`, etc.) |
| **System role** | Global JWT role: `admin`, `user`, `delivery` |
| **Restaurant role** | Per-restaurant assignment in `user_restaurant_roles`: `owner`, `manager`, `chef`, `waitstaff`, `delivery` |

---

## Identity & auth

| Term | Definition |
|------|------------|
| **Firebase UID** | External auth identifier in JWT (`uid` / `sub`) |
| **Postgres user UUID** | Internal `user.id` — use for all domain APIs |
| **JWT claims** | May include `role`, `restaurantId` — drives middleware routing |
| **FirebaseGuard** | Global backend guard — deny-by-default unless `@Public()` |
| **RestaurantAccessService** | Preferred service-level RBAC for restaurant-scoped endpoints |

**Rule:** Restaurant managers often have `SystemRole.USER` + `RestaurantRole.MANAGER`. Never gate staff features on system role alone.

---

## Restaurant & catalog

| Term | Definition |
|------|------------|
| **Restaurant** | Business entity — status: pending, active, suspended, etc. |
| **Menu item** | Sellable dish with price, category, add-ons, availability |
| **Category** | Menu grouping (starters, mains, drinks, …) |
| **Ingredient** | Inventory/catalog ingredient scoped to restaurant |
| **Add-on** | Optional modifier on a menu item |
| **Catalog mode** | Public menu read — only active restaurants/items |
| **Manage list** | Staff menu read — includes pending restaurant menus (`GET /menu-item/manage/list`) |

### Menu permissions (ADR consistency pass)

| Permission | Roles |
|------------|-------|
| `menu:read` | waitstaff, chef, manager, owner, admin |
| `menu:write` | manager, owner, admin |

---

## Floor & dine-in

| Term | Definition |
|------|------------|
| **Table** | Physical seating asset — `Table.status`: available, occupied, reserved, out_of_service |
| **TableSession** | **Source of truth** for an active dine-in visit — lifecycle: active → closed / expired |
| **Guest** | Walk-in patron record scoped to `restaurantId` (may link to `User`) |
| **Session status** | Operational FOH state: seated, ordering, sent_to_kitchen, …, completed, expired |
| **Table.status vs TableSession** | Table = physical asset; Session = operational dine-in unit. Never use table status as session state. |

### TableSession lifecycle

| State | Meaning |
|-------|---------|
| **Active** | `closedAt` is null, status not cancelled/expired |
| **Closed** | Normal completion — guest paid, session closed |
| **Expired** | No-show / stale — cron or explicit expiry frees table |

---

## Orders

| Term | Definition |
|------|------------|
| **Order** | Transaction line items tied to `restaurantId`, optional `tableSessionId` |
| **Order type** | `dine-in`, `takeout`, `delivery` |
| **Raw status** | Backend enum — never show directly in UI |
| **View layer** | Role-specific label mapping — see `app/src/lib/orderStatusViews.ts` |

### Order statuses (backend — canonical)

`draft` → `sent_to_kitchen` → `preparing` → `ready` → `served` → `completed`  
Legacy delivery: `pending` → `preparing` → `ready` → `delivered`  
Terminal: `cancelled`, `on_hold`

### Role-specific UI labels

| Raw status | Waitstaff label | Manager bucket |
|------------|-----------------|----------------|
| draft | Draft | Pending |
| sent_to_kitchen | In Kitchen | Pending |
| preparing | Preparing | Pending |
| ready | Ready | Pending |
| served | Served | Completed |
| completed | Completed | Completed |
| delivered | — | Completed |
| cancelled | Cancelled | Cancelled |

---

## Reservations

| Term | Definition |
|------|------------|
| **Reservation** | Table booking — `reservationTime`, `durationMinutes`, `partySize` |
| **Status** | pending, confirmed, checked_in, seated, no_show, completed, cancelled |
| **No-show** | `POST /reservations/:id/no-show` — frees table if no active session |

---

## Payments

| Term | Definition |
|------|------------|
| **Customer payment** | `POST /payments` — MeSomb mobile money or gateway |
| **Session payment** | `POST /table-sessions/:id/payments` — waitstaff-recorded cash/POS |
| **Payment entity** | Links to `order` and/or `tableSession` + `restaurant` |

---

## Chat & notifications

| Term | Definition |
|------|------------|
| **Conversation** | Thread — `customer_restaurant` or `customer_driver` |
| **User room** | WebSocket `user:{uuid}` — private notifications |
| **Restaurant room** | WebSocket `restaurant:{uuid}` — floor realtime (staff only after `join:restaurant`) |
| **Notification** | Persisted row + optional FCM push |

---

## Frontend conventions

| Term | Definition |
|------|------------|
| **Service** | `app/src/services/*.ts` — thin Axios wrappers |
| **Hook** | `app/src/hooks/*` — React Query + socket wrappers |
| **Nav config** | `app/src/config/*Nav.ts` — sidebar links per portal |
| **Envelope unwrap** | Most APIs return `{ success, data }` — services extract `data` |
| **getUuidBackedUserId()** | Resolve Postgres UUID from profile for API calls |

---

## Environment & deployment

| Term | Definition |
|------|------------|
| **app/** | Frontend repo (Next.js) |
| **backend/** | API repo (NestJS) |
| **ai/** | Specification docs (not yet in git remote) |
| **Migrations** | `backend/src/lib/database/migrations/` — run before deploy |

See [SETUP.md](./SETUP.md) for full environment variable list.
