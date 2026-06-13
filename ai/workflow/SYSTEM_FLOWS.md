# CuisineEase — System Flows

**Last updated:** 2026-06-11

End-to-end flows across portals. For portal-specific detail see `ai/features/*/FLOWS.md`.

---

## 1. Authentication & session

```mermaid
sequenceDiagram
  participant U as User
  participant FE as Next.js app
  participant API as NestJS API
  participant FB as Firebase

  U->>FE: Login / Google
  FE->>API: POST /auth/login or /auth/google
  API->>FB: Verify credentials
  API-->>FE: JWT + user profile
  FE->>FE: persistAuthSession cookie + localStorage
  FE->>FE: middleware routes to portal prefix
  U->>FE: Navigate protected route
  FE->>API: API calls with Bearer JWT
  API->>API: FirebaseGuard validates
```

**Logout:** `POST /auth/logout` + clear client session → redirect `/`.

---

## 2. Customer — browse to cart

```mermaid
flowchart LR
  A[Discover restaurants] --> B[View menu public GET /menu-item]
  B --> C[Add to cart POST /cart/add]
  C --> D[Cart page GET /cart]
  D --> E{Order type}
  E -->|Delivery/Takeout| F[Checkout flow]
  E -->|Dine-in| G[Select table + checkout]
```

**Key APIs:** `GET /restaurant`, `GET /menu-item`, `GET /cart`, `POST /cart/add`, `PATCH /cart/item/:id`

**Frontend:** `/customer/restaurants`, `/customer/cart`, `/customer/restaurants/[id]`

---

## 3. Customer — checkout & order (delivery/takeout)

```mermaid
sequenceDiagram
  participant C as Customer
  participant FE as Frontend
  participant API as API

  C->>FE: Confirm checkout
  FE->>API: POST /orders/from-cart { restaurantId }
  API-->>FE: Order created pending/preparing
  C->>FE: Pay optional
  FE->>API: POST /payments { orderId, amount, method }
  API-->>FE: Payment COMPLETED/FAILED
  Note over API: Notification + WS to customer
```

**Statuses (delivery legacy):** `pending` → `preparing` → `ready` → `delivered`

**Frontend:** `/customer/cart`, `/customer/order-history`, `/customer/order-history/[id]`

---

## 4. Dine-in — full restaurant flow (cross-portal)

This is the primary multi-role flow connecting customer, waitstaff, chef, and manager.

```mermaid
flowchart TB
  subgraph seating [Seating — Waitstaff]
    S1[Seat walk-in or reservation]
    S2[POST /table-sessions]
  end

  subgraph ordering [Ordering — Waitstaff]
    O1[Build order from menu]
    O2[POST /orders/staff]
    O3[POST /orders/:id/send-to-kitchen]
  end

  subgraph kitchen [Kitchen — Chef]
    K1[See sent_to_kitchen]
    K2[POST /orders/:id/kitchen-status preparing]
    K3[POST /orders/:id/kitchen-status ready]
  end

  subgraph service [Service — Waitstaff]
    V1[POST /orders/:id/mark-served]
    V2[POST /table-sessions/:id/request-bill]
  end

  subgraph payment [Payment — Waitstaff]
    P1[POST /table-sessions/:id/payments]
    P2[POST /table-sessions/:id/close]
  end

  S1 --> S2 --> O1 --> O2 --> O3
  O3 --> K1 --> K2 --> K3 --> V1 --> V2 --> P1 --> P2
```

**Realtime:** Each step emits `floor:updated` / `order:updated` to `restaurant:{id}` room.

**Alternative exits:**
- `POST /table-sessions/:id/guest-left` — when bill settled or empty session
- `POST /reservations/:id/no-show` — reservation only, frees table

---

## 5. Customer — table reservation

```mermaid
sequenceDiagram
  participant C as Customer
  participant FE as Frontend
  participant API as API

  C->>FE: Book table on restaurant page
  FE->>API: GET /reservations/table/:id/availability
  FE->>API: POST /reservations
  C->>FE: Pay deposit optional
  FE->>API: POST /reservations/:id/pay
  Note over API: Waitstaff sees reservation on /waitstaff/reservations
  Note over API: Waitstaff seats → POST /table-sessions with reservationId
```

**Frontend:** `/customer/restaurants/[id]` (BookTableDialog), `/customer/reservations`

---

## 6. Manager — restaurant onboarding

```mermaid
flowchart LR
  A[Register manager POST /auth/register] --> B[Apply POST /restaurant/with-manager]
  B --> C[Status pending]
  C --> D[Admin approves PATCH /restaurant/:id/status]
  D --> E[Manager /manager full access]
  C --> F[Manager /manager/pending limited]
```

---

## 7. Manager — menu management

```mermaid
flowchart LR
  M[Manager /manager/menu] --> C[POST /menu-item]
  M --> U[PATCH /menu-item/:id]
  M --> I[Ingredients / add-ons]
  M --> PH[POST /menu-item/:id/upload-photo]
```

**RBAC:** `menu:write` — manager, owner, admin only.  
**Waitstaff read:** `GET /menu-item/manage/list?restaurantId=` for order building.

---

## 8. Manager — orders oversight

Manager uses **view-layer buckets** (not raw statuses):

| Tab | Backend filter (`statuses` param) |
|-----|-----------------------------------|
| All | none |
| Pending | draft, sent_to_kitchen, preparing, pending, ready, on_hold |
| Completed | served, completed, delivered |
| Cancelled | cancelled |

**Frontend:** `/manager/orders`, `/manager/orders/[id]`  
**Mapping:** `app/src/lib/orderStatusViews.ts`

---

## 9. Chat

```mermaid
flowchart TB
  CR[Customer starts chat] --> API1[POST /chat/conversations]
  API1 --> WS[join user room]
  MSG[POST /chat/conversations/:id/messages]
  MSG --> WS2[emit chat:message:new to participants]
  DRV[Driver chat] -->|active delivery only| MSG
```

**Types:** `customer_restaurant`, `customer_driver`  
**Frontend:** `/customer/chats`, `/manager/messages`, `/waitstaff/messages`, `/delivery/chats`

---

## 10. Notifications

```mermaid
flowchart LR
  E[Domain event order/reservation/etc] --> L[Event listener]
  L --> N[NotificationsService.create]
  N --> DB[(user_notifications)]
  N --> WS[emitToUser user:id]
  N --> FCM[Firebase push]
```

**Frontend:** Bell component + `/customer/notifications`, `/manager/notifications`, etc.

---

## 11. Realtime — restaurant floor

```mermaid
sequenceDiagram
  participant FE as Waitstaff UI
  participant WS as Socket.io
  participant API as API

  FE->>WS: connect with Firebase JWT
  FE->>WS: emit join:restaurant { restaurantId }
  WS->>WS: verify user_restaurant_roles
  WS-->>FE: join restaurant:uuid room
  API->>WS: floor:updated / order:updated
  WS-->>FE: event with ids only
  FE->>API: fetch secured REST detail
```

---

## 12. Delivery driver flow

```mermaid
flowchart LR
  O[Order ready/delivery type] --> D[Manager assigns delivery]
  D --> DR[Driver /delivery dashboard]
  DR --> CHAT[Chat with customer]
  DR --> DONE[Delivery status terminal]
  DONE --> X[Close driver chat thread]
```

---

## 13. Platform admin

```mermaid
flowchart LR
  A[/dashboard/restaurants] --> B[List pending restaurants]
  B --> C[Approve or suspend]
  C --> D[Restaurant status active/suspended]
```

---

## 14. Table availability maintenance (background)

**Cron:** `FloorMaintenanceService` every 15 minutes

- Finds expired reservations (past `reservationTime + duration`)
- No active `TableSession` on table
- Sets reservation `no_show`, table `available`
- Emits `reservation:updated`, `floor:updated`

---

## Flow index by portal

| Portal | Primary flows |
|--------|---------------|
| Customer | Browse, cart, checkout, reservation, track, chat, profile |
| Manager | Onboarding, menu, orders, tables, staff, reservations, reviews, chat |
| Waitstaff | Floor, seat, order, kitchen handoff, serve, bill, payment, close |
| Chef | Kitchen queue, stations, analytics, staff messaging — see kitchen spec |
| Delivery | Assigned orders, customer chat |
| Admin | Restaurant approval |

See also:

- [features/customer/FLOWS.md](../features/customer/FLOWS.md)
- [features/manager/FLOWS.md](../features/manager/FLOWS.md)
- [features/waitstaff/FLOWS.md](../features/waitstaff/FLOWS.md)
- [features/kitchen/FLOWS.md](../features/kitchen/FLOWS.md)
