# CuisineEase -- Architecture Overview

**Last updated:** 2026-06-13

---

## System Context

```
+---------------------------------------------------------+
|                      Clients                             |
|  Next.js 14 Web App (customer, manager, staff, chef,    |
|  driver, admin portals)                                 |
+-------------------------------+-------------------------+
                                |
               HTTPS + JWT      |      WSS (same host)
                                |
+-------------------------------v-------------------------+
|                  Backend -- NestJS 11                    |
|  +---------+ +----------+ +-----------------------+    |
|  | REST API| |Socket.io | | Event Emitter          |    |
|  |         | | Gateway  | | -> Notifications Svc   |    |
|  +----+----+ +----+-----+ +-----------+-----------+    |
+-------+-----------+-------------------+-----------------+
        |           |                   |
+-------v-----------v-------------------v-----------------+
|              Data & External Services                    |
|  +----------+ +-------+ +--------+ +-------+ +--------+ |
|  |PostgreSQL| | Redis | |Firebase| |MeSomb | |SendGrid| |
|  |(TypeORM) | |(cache)| |Auth+   | |Mobile | |Email   | |
|  |          | |       | |Storage | |Money  | |        | |
|  |          | |       | |+FCM    | |       | |        | |
|  +----------+ +-------+ +--------+ +-------+ +--------+ |
+---------------------------------------------------------+
```

---

## Frontend Architecture

### Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 14 App Router, React 18, TypeScript |
| Styling | Tailwind CSS, shadcn/ui (Radix primitives) |
| Server state | TanStack Query v5 |
| HTTP | Axios singleton (`src/lib/api.ts`) |
| Auth session | JWT in cookie + localStorage; Firebase for Google |
| i18n | i18next (en/fr), client-side |
| Realtime | socket.io-client |

### Layering

```
src/app/           -> Routes & page composition (by role portal)
src/components/    -> UI components (domain + shared)
src/hooks/         -> React Query wrappers, sockets, live refresh
src/services/      -> API client functions (thin Axios wrappers)
src/type/          -> TypeScript DTOs mirroring API
src/config/        -> Nav configs per role (customerNav, managerNav, etc.)
src/lib/           -> api.ts, auth helpers, wsUrl, utils, orderStatusViews
src/middleware.ts  -> JWT role gate for protected prefixes
```

### Auth & Routing Flow

1. User signs in -> `POST /auth/login` | `/auth/register` | `/auth/google`
2. `persistAuthSession()` stores tokens + role claims
3. `middleware.ts` decodes JWT, maps role -> allowed path prefix
4. Post-login routing:
   - `admin` -> `/dashboard`
   - `waitstaff` -> `/waitstaff`
   - `chef` -> `/chef`
   - `delivery` -> `/delivery`
   - `restaurantId` claim -> `/manager`
   - default `user` -> `/customer`

### Role-Specific UI Shells

| Portal | Layout | Nav Config |
|--------|--------|-----------|
| Customer | `customer/layout.tsx` | `config/customerNav.ts` |
| Manager | `manager/layout.tsx` | `config/managerNav.ts` |
| Waitstaff | `waitstaff/layout.tsx` | `config/waitstaffNav.ts` |
| Chef | `chef/layout.tsx` | `config/chefNav.ts` |
| Driver | `delivery/layout.tsx` | `config/deliveryNav.ts` |
| Admin | `dashboard/layout.tsx` | `config/platformAdminNav.ts` |

---

## Backend Architecture

### Stack

| Layer | Technology |
|-------|-----------|
| Framework | NestJS 11, Express |
| ORM | TypeORM 0.3, PostgreSQL |
| Cache | Redis (ioredis) |
| Auth | Firebase Admin JWT, global `FirebaseGuard` |
| Realtime | Socket.io via `IoAdapter` on main HTTP server |
| Events | `@nestjs/event-emitter` |
| Docs | Swagger at `/api` |

### Module Map

| Module | Responsibility |
|--------|---------------|
| `auth` | Login, register, Google, refresh, email verify, password reset |
| `user` | Profiles, favorites, device tokens, notification prefs, restaurant roles |
| `restaurant` | CRUD, status, schedules, onboarding |
| `menu-item`, `ingredient`, `add-on` | Catalog per restaurant |
| `cart` | Pre-checkout cart |
| `orders` | Order lifecycle, event listeners, secured create/update |
| `reservations` | Table bookings, lifecycle transitions |
| `tables` | Physical tables + sections (frozen controller -- SEC-001) |
| `floor` | TableSession, Guest, floor map, billing, session payments |
| `payment` | Payment records, gateway resolver (MeSomb, cash) |
| `delivery` | Delivery assignment; hooks chat open/close |
| `driver` | Driver-scoped assignments, shifts, offline sync, performance |
| `kitchen` | Queue, stations, analytics, monitor, reorder, sync |
| `chat` | Conversations, messages, image attachments, reactions, E2EE |
| `notifications` | Persist, WebSocket emit, FCM push |
| `review` | Restaurant reviews |
| `storage` | Upload abstraction (Firebase default, Google Drive optional) |
| `emails`, `mail` | Transactional email |

### Request Pipeline

```
Request
  -> FirebaseGuard (global, deny-by-default)
  -> @Public() bypass where marked
  -> RolesGuard (selective controllers)
  -> RestaurantAccessService (service-level RBAC)
  -> ValidationPipe
  -> Controller -> Service -> Repository
  -> apiSuccess() envelope
  -> HttpExceptionFilter on errors
```

---

## Authorization Model (Dual)

1. **System role** -- `admin` | `user` | `delivery` (in JWT)
2. **Restaurant role** -- per `user_restaurant_roles` table: `owner`, `manager`, `chef`, `waitstaff`, `delivery`

**Critical rule:** Restaurant managers often retain `SystemRole.USER`. Features must check restaurant staff membership, not only system role.

**Preferred guard:** `RestaurantAccessService` -- `assertWaitstaffOrAbove`, `assertChefOrAbove`, `assertManagerOrAbove`, `assertRestaurantIdMatch`, `getRestaurantRole`.

---

## WebSocket Architecture

### Connection

- Socket.io attached to NestJS HTTP server via `IoAdapter` (shared port, not separate :3002)
- Frontend `getWebSocketUrl()` uses API host (`NEXT_PUBLIC_WS_URL` optional override)
- Auth: Firebase JWT via `auth: { token }` on client connect

### Rooms

| Room | Members | Purpose |
|------|---------|---------|
| `user:{uuid}` | Individual user | Private notifications, chat messages |
| `restaurant:{uuid}` | Staff with verified role | Floor realtime events |
| `admin` | Platform admins | Admin notifications |
| `delivery` | Drivers | Delivery broadcasts |
| `notifications:*` | All connected | General notification channels |

### Restaurant Realtime Events

| Event | Trigger | Subscribers |
|-------|---------|-------------|
| `floor:updated` | Session change | Waitstaff, chef, manager |
| `order:updated` | Order PATCH | All staff portals |
| `table-session:*` | Session lifecycle | Waitstaff floor |
| `payment:recorded` | Payment recorded | Waitstaff billing |
| `reservation:updated` | Reservation PATCH | Waitstaff arrivals |
| `kitchen:queue:updated` | Kitchen changes | Chef queue |
| `kitchen:station:updated` | Station changes | Chef stations |
| `delivery:*` | Delivery lifecycle | Driver portal |
| `driver:shift:updated` | Shift changes | Driver shift |

### Room Join Protocol

1. Client connects with JWT
2. Client emits `join:restaurant` with `restaurantId`
3. Server verifies `user_restaurant_roles`
4. Server leaves prior `restaurant:*` room (prevents cross-tenant leak)
5. Server joins `restaurant:{id}` room

---

## Payment Flow

### Methods

| Gateway | Status | Use case |
|---------|--------|----------|
| **MeSomb** (MTN/Orange Mobile Money) | Partial (gateway exists, service TODOs) | Customer checkout, session payments |
| **Cash** | Complete | COD, waitstaff-recorded session payments |

### Customer Payment

```
POST /payments { orderId, amount, method }
  -> PaymentService.processPayment
  -> MeSomb gateway or cash marker
  -> On success: autoRoutePaidOrderToKitchen (delivery/takeout)
  -> Notification + WS to customer
```

### Session Payment (Dine-in)

```
POST /table-sessions/:id/payments { amount, method }
  -> Payment linked to tableSession + restaurant
  -> Cash: immediate COMPLETED
  -> MeSomb: gateway processing (ENT-C05)
  -> Auto-close session when billing allows
```

---

## Chat System

### Conversation Types

| Type | Participants | Lifecycle |
|------|-------------|-----------|
| `customer_restaurant` | Customer + restaurant staff | Ongoing (one per pair) |
| `customer_driver` | Customer + assigned driver | Active delivery only; closes on terminal status |

### Access Rules

- Customers start restaurant chats; staff start with `customerId`
- Staff synced as participants on access (ADR-009)
- Driver chat only while delivery active with assigned driver
- Image upload via backend storage (Firebase default)

### Realtime

- `chat:message:new`, `chat:conversation:updated` via `NotificationsGateway.emitToUser`
- Frontend: `useChatSocket` patches message/conversation caches

### E2EE (Kitchen)

- `staff_device_keys` table -- public keys only on server
- ECDH P-256 + AES-GCM session keys in `kitchenE2EE.ts`
- Server stores ciphertext only for private DMs
- Operational/manager channels remain non-E2EE

---

## Notification System

```
Domain event (order, reservation, etc.)
  -> Event listener
  -> NotificationsService.create (persists row)
  -> WebSocket to user:{id} room
  -> FCM to registered device tokens
```

### Frontend

- Bell component in each portal's top nav
- Full inbox at `/[role]/notifications`
- `useNotificationLive` -- polling + focus refresh (15s)
- `useChatLive` -- socket + 5s polling for chat badges

---

## Database & Migration Structure

### Database

- PostgreSQL via TypeORM 0.3
- SSL via `DB_SSL` env var
- `synchronize` locked to non-prod (ADR-001)

### Migrations

- Path: `backend/src/lib/database/migrations/`
- Run: `npm run build && npm run migration:run`
- 21+ migration files with monotonic timestamps
- All use `IF NOT EXISTS` / `IF EXISTS` for idempotency

### Key Entities

```
Restaurant ----+-- MenuItem ------ Ingredient, AddOn
               |-- Table --------- TableSession ---- Guest, Reservation
               |-- UserRestaurantRole
               |-- Order --------- OrderItem, Payment
               |-- Delivery
               +-- Reservation

User -----------+-- Order, Cart, Address, Reservation
                |-- ChatParticipant ---- ChatMessage
                +-- Notification

ChatConversation ---- ChatParticipant, ChatMessage
```

### State Machines

**Order (dine-in):** `draft -> sent_to_kitchen -> preparing -> ready -> served -> completed`
**Order (delivery):** `pending -> preparing -> ready -> delivered`
**TableSession:** `seated -> ... -> completed | expired`
**Reservation:** `pending -> confirmed -> checked_in -> seated -> completed | no_show | cancelled`
**Delivery:** `pending -> assigned -> in_transit -> delivered | failed | cancelled`

---

## Deployment Topology

| Service | Platform | URL |
|---------|----------|-----|
| Frontend | Vercel | `https://www.cuisineease.com` |
| Backend API + WebSocket | Render | `https://api.cuisineease.com` |
| Database | External PostgreSQL | via `DATABASE_URL` |
| Auth | Firebase | Project `cuisine-ease` |
| Storage | Firebase Storage | Default provider |
| Cache | Redis | Optional via env |
| Push | Firebase Cloud Messaging | Via device tokens |

### Key Environment Variables

| Variable | Location | Purpose |
|----------|----------|---------|
| `NEXT_PUBLIC_API_URL` | Vercel | Backend API base URL |
| `NEXT_PUBLIC_WS_URL` | Vercel | WebSocket URL (should match API host) |
| `DATABASE_URL` | Render | PostgreSQL connection |
| `STORAGE_TYPE` | Render | `FIREBASE` (default) |
| `CORS_ORIGINS` | Render | Allowed frontend origins |
| `FRONTEND_URL` | Render | Production frontend URL |
