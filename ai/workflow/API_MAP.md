# CuisineEase — API Map

**Last updated:** 2026-06-11

High-level map of REST endpoints by domain. For full request/response schemas use Swagger at `{API_URL}/api`.

**Legend:** 🔒 = Firebase JWT required · 🌐 = `@Public()` · 👤 = self-scoped · 🏪 = restaurant-scoped RBAC

---

## Auth — `/auth`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| POST | `/auth/login` | 🌐 | Email/password login |
| POST | `/auth/google` | 🌐 | Google OAuth |
| POST | `/auth/register` | 🌐 | Customer/manager signup |
| POST | `/auth/refresh-token` | 🌐 | Refresh JWT |
| POST | `/auth/logout` | 🔒 | Logout |
| POST | `/auth/reset-password` | 🌐 | Password reset |
| POST | `/auth/verify-email` | 🌐 | Email verification |

---

## User — `/user`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/user/me` | 🔒 👤 | Current profile |
| PATCH | `/user/:id` | 🔒 👤 | Update profile |
| GET/PATCH | `/user/notification-preferences` | 🔒 👤 | Notification prefs |
| GET/POST/DELETE | `/user/favorite-*` | 🔒 👤 | Favorites |
| GET/POST/DELETE | `/user/restaurant-roles/*` | 🔒 🏪 | Staff role management |
| POST | `/user/restaurant-staff/invite` | 🔒 🏪 | Invite staff |

---

## Restaurant — `/restaurant`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/restaurant` | 🌐/🔒 | List/discover |
| GET | `/restaurant/:id` | 🌐 | Public detail |
| POST | `/restaurant` | 🔒 | Create |
| POST | `/restaurant/with-manager` | 🔒 | Onboarding application |
| PATCH | `/restaurant/:id/status` | 🔒 admin | Approve/suspend |
| PATCH | `/restaurant/:id` | 🔒 manager | Update profile |

---

## Menu — `/menu-item`, `/menu-item/category`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/menu-item` | 🌐 | Public catalog |
| GET | `/menu-item/manage/list` | 🔒 🏪 read | Staff menu (waitstaff+) |
| GET | `/menu-item/:id` | 🌐 | Item detail |
| POST/PATCH/DELETE | `/menu-item/*` | 🔒 🏪 write | Manager catalog CRUD |
| GET | `/menu-item/category` | 🌐 | Categories |

---

## Cart — `/cart`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/cart` | 🔒 👤 | Get cart |
| POST | `/cart/add` | 🔒 👤 | Add item |
| PATCH/DELETE | `/cart/item/:id` | 🔒 👤 | Update/remove |
| DELETE | `/cart/clear` | 🔒 👤 | Clear |

---

## Orders — `/orders`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| POST | `/orders` | 🔒 | Create (secured) |
| POST | `/orders/staff` | 🔒 🏪 waitstaff+ | Dine-in staff order |
| POST | `/orders/from-cart` | 🔒 👤 | Checkout cart |
| GET | `/orders` | 🔒 | List (secured, supports `statuses`) |
| GET | `/orders/:id` | 🔒 | Detail (secured) |
| PATCH | `/orders/:id` | 🔒 | Update (role transitions) |
| POST | `/orders/:id/send-to-kitchen` | 🔒 waitstaff+ | Draft → kitchen |
| POST | `/orders/:id/mark-served` | 🔒 waitstaff+ | Ready → served |
| POST | `/orders/:id/kitchen-status` | 🔒 chef+ | Kitchen advance |

---

## Floor & sessions

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/restaurants/:id/floor` | 🔒 waitstaff+ | Floor map |
| GET | `/restaurants/:id/tables/:tableId` | 🔒 waitstaff+ | Table detail |
| GET | `/restaurants/:id/guests/search` | 🔒 waitstaff+ | Guest lookup |
| POST | `/restaurants/:id/guests` | 🔒 waitstaff+ | Create/find guest |
| POST | `/table-sessions` | 🔒 waitstaff+ | Seat / start session |
| GET | `/table-sessions/:id` | 🔒 waitstaff+ | Session detail |
| GET | `/table-sessions/:id/billing` | 🔒 waitstaff+ | Bill summary |
| POST | `/table-sessions/:id/payments` | 🔒 waitstaff+ | Record payment |
| POST | `/table-sessions/:id/request-bill` | 🔒 waitstaff+ | Request bill |
| POST | `/table-sessions/:id/guest-left` | 🔒 waitstaff+ | Guest departed |
| POST | `/table-sessions/:id/close` | 🔒 waitstaff+ | Close session |
| POST | `/table-sessions/:id/transfer` | 🔒 waitstaff+ | Move table |

---

## Tables — `/tables` ⚠️

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/tables/restaurant/:id` | 🌐 | Public table list |
| POST/PATCH/DELETE | `/tables/*` | ⚠️ Unguarded | **SEC-001 backlog** |

Manager/waitstaff should use `/restaurants/:id/floor` for secured reads.

---

## Reservations — `/reservations`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| POST | `/reservations` | 🔒 👤 | Book |
| GET | `/reservations/me` | 🔒 👤 | My reservations |
| GET | `/reservations/restaurant/:id` | 🔒 waitstaff+ | Restaurant list |
| PATCH | `/reservations/:id` | 🔒 staff/owner | Update |
| POST | `/reservations/:id/pay` | 🔒 👤 | Pay deposit |
| POST | `/reservations/:id/no-show` | 🔒 waitstaff+ | Mark no-show |
| GET | `/reservations/table/:id/availability` | 🔒 | Check slot |

---

## Payments — `/payments`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| POST | `/payments` | 🔒 👤 | Customer pay order |
| GET | `/payments` | 🔒 👤 | My payments |
| GET | `/payments/:id` | 🔒 👤/staff | Payment detail |

---

## Chat — `/chat`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/chat/conversations` | 🔒 | Inbox |
| POST | `/chat/conversations` | 🔒 | Start thread |
| GET/POST | `/chat/conversations/:id/messages` | 🔒 | Messages |
| POST | `/chat/conversations/:id/attachments` | 🔒 | Image upload |

---

## Notifications — `/notifications`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/notifications/user/:userId` | 🔒 self/admin | Inbox |
| PATCH | `/notifications/user/:userId/*` | 🔒 self/admin | Mark read |

---

## Delivery — `/delivery`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET/POST/PATCH | `/delivery/*` | 🔒 manager/driver | Assignments |

---

## Reviews — `/review`

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| POST | `/review` | 🔒 👤 | Submit review |
| GET | `/review` | 🌐/🔒 | List by restaurant |

---

## WebSocket events (Socket.io)

| Event | Room | Purpose |
|-------|------|---------|
| `join:restaurant` | → `restaurant:{id}` | Staff floor subscribe |
| `floor:updated` | restaurant | Floor refresh trigger |
| `order:updated` | restaurant | Order refresh trigger |
| `table-session:*` | restaurant | Session lifecycle |
| `payment:recorded` | restaurant | Payment recorded |
| `notification:new` | `user:{id}` | User notification |
| `chat:message:new` | `user:{id}` | Chat message |

---

## Frontend service map

| Service file | Backend domain |
|--------------|----------------|
| `auth.ts` | `/auth` |
| `user.ts` | `/user` |
| `restaurant.ts` | `/restaurant` |
| `menu.ts`, `category.ts` | Menu |
| `cart.ts` | Cart |
| `order.ts` | Orders |
| `floor.ts` | Floor, sessions, guests |
| `reservation.ts` | Reservations |
| `payment.ts` | Payments |
| `chat.ts` | Chat |
| `notification.ts` | Notifications |
| `staff.ts` | Staff roles |
| `tables.ts`, `tableSections.ts` | Tables (manager) |
| `delivery.ts` | Delivery |
| `review.ts` | Reviews |
| `dashboard.ts` | Manager stats |
| `kitchen.ts` *(planned)* | Kitchen queue, stations, analytics |

---

## Kitchen — `/kitchen` *(planned — see `ai/features/kitchen/SPEC.md`)*

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| GET | `/kitchen/queue` | 🔒 🏪 chef+ | Aggregated ticket queue |
| GET | `/kitchen/stations` | 🔒 🏪 chef+ read | Station list + workload |
| POST/PATCH | `/kitchen/stations` | 🔒 🏪 manager write | Configure stations |
| GET | `/kitchen/history` | 🔒 🏪 chef+ | Searchable ticket archive |
| GET | `/orders/:id/timeline` | 🔒 🏪 chef+ | Status replay |
| POST | `/orders/:id/kitchen-status` | 🔒 🏪 chef+ | **Exists today** — advance prep |
| POST | `/kitchen/sync` | 🔒 🏪 chef+ | Offline action replay *(P3)* |
| GET | `/kitchen/analytics` | 🔒 🏪 chef+ / manager read | Kitchen metrics *(planned)* |

**WebSocket (existing + planned):** `order:updated`, `kitchen:alert`, `kitchen:station:updated`
