# Customer Portal — Specification

**Version:** 1.0  
**Date:** 2026-06-11  
**Status:** Production

---

## Business goals

Customers discover restaurants, build orders, pay, track delivery or dine-in progress, reserve tables, and communicate with restaurants and drivers — all from a single authenticated web portal.

| Problem | Outcome |
|---------|---------|
| Fragmented ordering experience | Unified cart, checkout, and order history |
| No visibility after ordering | Order tracking, notifications, chat |
| Table booking is offline | In-app reservations with availability check |
| Language barrier | EN/FR i18n across customer UI |

---

## Personas

| Persona | Primary goals |
|---------|---------------|
| **Customer** | Browse, order (delivery/takeout/dine-in), pay, track, chat |
| **Returning customer** | Favorites, order history, saved addresses |
| **Guest diner** | May book/reserve; dine-in at table may be handled by waitstaff with guest record |

---

## Access & routing

| Item | Value |
|------|-------|
| **System role** | `user` (no restaurant staff role) |
| **Path prefix** | `/customer` |
| **Middleware** | `app/src/middleware.ts` — redirects non-customer paths to `/customer` |
| **Auth** | Firebase JWT in cookie; `FirebaseGuard` on API |

Users with `restaurantId` JWT claim route to `/manager`, not customer — even if system role is `user`.

---

## Navigation

Config: `app/src/config/customerNav.ts`

| Route | Sidebar label key | Purpose |
|-------|-------------------|---------|
| `/customer` | `sidebar.menu` | Home / menu discovery |
| `/customer/restaurants` | `sidebar.restaurants` | Restaurant list |
| `/customer/reservations` | `sidebar.reservations` | My reservations |
| `/customer/order-history` | `sidebar.orderHistory` | Past orders |
| `/customer/favorites` | `sidebar.favorites` | Saved restaurants |
| `/customer/chats` | `sidebar.chats` | Chat inbox |
| `/customer/notifications` | `sidebar.notifications` | Notification inbox |
| `/customer/cart` | `sidebar.cart` | Cart & checkout |
| `/customer/statistics` | `sidebar.statistics` | Spending / activity stats |
| `/customer/settings` | `sidebar.settings` | Profile & preferences |

**Layout:** `app/src/app/customer/layout.tsx` — sidebar (desktop), top nav, live notification + chat hooks.

**Nested routes (no sidebar entry):**

| Route | Purpose |
|-------|---------|
| `/customer/restaurants/[id]` | Restaurant detail, menu, book table |
| `/customer/order-history/[id]` | Single order detail + tracking |
| `/customer/[id]` | Legacy/alternate menu item route |
| `/customer/overview` | Overview dashboard |
| `/customer/settings/*` | Security, notifications, policy, help, preferences |

---

## Key components

| Area | Location |
|------|----------|
| Cart & checkout | `components/CustomerDashboard/Cart/*` |
| Restaurants | `components/CustomerDashboard/Restaurants/*` |
| Book table | `components/CustomerDashboard/Restaurant/BookTableDialog.tsx` |
| Order history | `components/CustomerDashboard/OrderHistory/*` |
| Settings / profile | `components/CustomerDashboard/CustomerSetting/*` |
| Menu browsing | `components/CustomerDashboard/CustomerMenu/*` |
| Reviews | `components/CustomerDashboard/Reviews/*` |
| Shell | `app/customer/(components)/CustomerSideBar.tsx`, `CustomerTopNavBar.tsx` |

---

## Frontend services

| Service | Backend domain | Primary use |
|---------|----------------|-------------|
| `auth.ts` | `/auth` | Login session |
| `user.ts` | `/user` | Profile, favorites, addresses |
| `restaurant.ts` | `/restaurant` | Discovery, detail |
| `menu.ts` | `/menu-item` | Public catalog |
| `cart.ts` | `/cart` | Add/update/clear cart |
| `order.ts` | `/orders` | Checkout, history, tracking |
| `payment.ts` | `/payments` | MeSomb / cash payment |
| `reservation.ts` | `/reservations` | Book, list, pay deposit |
| `chat.ts` | `/chat` | Restaurant & driver threads |
| `notification.ts` | `/notifications` | Inbox |
| `review.ts` | `/review` | Submit reviews |
| `address.ts` | `/user` addresses | Delivery addresses |
| `delivery.ts` | `/delivery` | Delivery record on checkout |
| `tables.ts` | `/tables` | Dine-in table picker (public list) |

Hooks: `useCart`, `useCurrentUser`, `useMenuItems`, `useNotificationLive`, `useChatLive`, `useTablesByRestaurant`, `useAppFees`.

---

## Backend modules (customer-facing)

| Module | Customer operations |
|--------|---------------------|
| `auth` | Register, login, Google |
| `user` | Profile, favorites, notification prefs |
| `restaurant` | List, detail (public/active) |
| `menu-item` | Public menu read |
| `cart` | User-scoped cart CRUD |
| `orders` | `POST /orders/from-cart`, list own orders |
| `payments` | Pay for order |
| `reservations` | Book, `GET /reservations/me`, pay deposit |
| `chat` | Start/view conversations |
| `notifications` | User inbox |
| `review` | Submit review |
| `delivery` | Created on delivery checkout |

**RBAC pattern:** Self-scoped — customer can only access own cart, orders, reservations, payments. Restaurant staff endpoints are not used from customer UI.

---

## Order types

Checkout supports three modes (`app/src/app/customer/cart/page.tsx`):

| Type | UI | Backend |
|------|-----|---------|
| **Delivery** | Address + time window | `POST /orders/from-cart` + optional `POST /delivery` |
| **Takeout** | Pickup time | Same checkout path |
| **Dine-in** | Table selection | Order linked to table; waitstaff may take over session flow |

Delivery uses legacy status path: `pending` → `preparing` → `ready` → `delivered`.

---

## Realtime

| Hook | Events |
|------|--------|
| `useNotificationLive` | `notification:new` → refetch inbox |
| `useChatLive` | `chat:message:new` → refetch conversations |

Customer does **not** join restaurant floor WebSocket rooms.

---

## i18n

Namespace: `customer:` (layout titles), plus `cart:`, shared keys.  
Switcher: `CustomerLanguageSwitcher.tsx`.

---

## Permissions summary

| Action | Requirement |
|--------|-------------|
| Browse active restaurants | Public or authenticated |
| Cart / checkout | Authenticated customer |
| Chat with restaurant | Authenticated; thread type `customer_restaurant` |
| Chat with driver | Active delivery on order |
| Pay order | Authenticated; order belongs to user |
| Book reservation | Authenticated; table availability check |

---

## Known gaps & notes

| Item | Notes |
|------|-------|
| MeSomb payments | Partial — see `payment.service.ts` TODOs |
| Dine-in after checkout | Full session lifecycle is waitstaff-driven post-order |
| Revenue stats on `/customer/statistics` | Client-side aggregation from order history |
| Chef/waitstaff visibility | Customer sees simplified order status in tracking UI |

---

## Related docs

- Flows: [FLOWS.md](./FLOWS.md)
- System flows: [../../workflow/SYSTEM_FLOWS.md](../../workflow/SYSTEM_FLOWS.md)
- API map: [../../workflow/API_MAP.md](../../workflow/API_MAP.md)
- Data model: [../../workflow/DATA_MODEL.md](../../workflow/DATA_MODEL.md)
