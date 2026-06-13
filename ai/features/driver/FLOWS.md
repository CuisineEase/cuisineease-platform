# Driver Dashboard — System Flows

**Date:** 2026-06-11

---

## 1. Assignment flow

```mermaid
sequenceDiagram
  participant M as Manager
  participant API as Delivery API
  participant DB as Database
  participant WS as WebSocket
  participant D as Driver app

  M->>API: PATCH /delivery/:id { deliveryPersonId }
  API->>DB: status=assigned, assignedAt=now
  API->>WS: delivery:assigned
  WS->>D: invalidate assignments cache
  D->>API: GET .../driver/assignments
  API-->>D: scoped list
```

---

## 2. Delivery workflow

```mermaid
stateDiagram-v2
  [*] --> pending: Order created
  pending --> assigned: Manager assigns driver
  assigned --> assigned: Driver accepts
  assigned --> in_transit: Driver picks up
  in_transit --> in_transit: Driver arrives
  in_transit --> delivered: Driver completes + proof
  in_transit --> failed: Driver marks failed
  assigned --> cancelled: Cancelled
  delivered --> [*]
  failed --> [*]
  cancelled --> [*]
```

Each transition:
1. `POST .../driver/assignments/:id/status`
2. Insert `delivery_events` row
3. `ChatService.onDeliveryUpdated` (close chat on terminal states)
4. `emitDeliveryUpdated` to restaurant + driver rooms

---

## 3. Shift flow

```mermaid
flowchart LR
  A[Off shift] -->|clock-in| B[Active]
  B -->|break| C[On break]
  C -->|resume| B
  B -->|clock-out| A
  B -->|set availability| B
```

---

## 4. Chat flow

1. Manager assigns driver → `onDeliveryUpdated` creates `customer_driver` conversation
2. Driver opens Messages or taps chat on delivery card
3. `createConversation({ deliveryId })` idempotent
4. Messages via existing chat WebSocket events

---

## 5. Offline sync flow

1. Driver loses connectivity — banner shown
2. Actions queued locally (future: IndexedDB)
3. On reconnect: `POST .../driver/sync` with `idempotencyKey` per action
4. Backend replays status/proof; marks `driver_offline_actions.appliedAt`

---

## 6. Realtime subscription

```
Driver connects → joins user:{id}, delivery, restaurant:{id}
Events: delivery:*, driver:shift:updated
Frontend: useDriverLive → invalidate React Query keys
```

---

## 7. RBAC decision tree

```
Request to driver API
  → FirebaseGuard (JWT valid?)
  → resolveUser(uid)
  → assertDeliveryOrAbove(uid, restaurantId)
  → filter/query by deliveryPerson.id == user.id
```
