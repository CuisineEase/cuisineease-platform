# Kitchen Dashboard — Flows

**Last updated:** 2026-06-11  
**Spec:** [SPEC.md](./SPEC.md)

---

## 1. Chef login & portal entry

```mermaid
sequenceDiagram
  participant C as Chef
  participant FE as Frontend
  participant API as API

  C->>FE: Login (restaurant role=chef)
  FE->>API: POST /auth/login
  API-->>FE: JWT + restaurantId
  FE->>FE: middleware → /chef
  FE->>WS: join:restaurant
  FE->>API: GET /orders?statuses=sent_to_kitchen,preparing
```

---

## 2. Core ticket lifecycle (today + target)

```mermaid
stateDiagram-v2
  [*] --> sent_to_kitchen: Waitstaff send-to-kitchen
  sent_to_kitchen --> preparing: Chef start prep
  preparing --> ready: Chef mark ready
  ready --> served: Waitstaff mark served
  preparing --> on_hold: Chef pause
  on_hold --> preparing: Chef resume
  sent_to_kitchen --> cancelled: Manager/waitstaff cancel
```

**Today:** `/chef` lists `sent_to_kitchen` and `preparing`; buttons call `advanceKitchenOrderStatus`.  
**Target:** Full queue UI with timers, stations, bulk actions, WS-driven refresh.

---

## 3. Kitchen Queue — primary workflow

```mermaid
flowchart TB
  A[Open /chef/queue] --> B{View mode}
  B --> Kanban
  B --> List
  B --> Compact
  Kanban --> C[Filter station/priority]
  C --> D[Select ticket]
  D --> E{Action}
  E --> F[Start preparing]
  E --> G[Mark ready]
  E --> H[Hold / re-fire]
  E --> I[Assign station/chef]
  F --> API1[POST /orders/:id/kitchen-status]
  G --> API1
  API1 --> WS[order:updated]
  WS --> WS2[Waitstaff floor refresh]
```

---

## 4. Waitstaff → kitchen → waitstaff

```mermaid
sequenceDiagram
  participant WS as Waitstaff
  participant API as API
  participant K as Kitchen UI
  participant W as Waitstaff UI

  WS->>API: POST /orders/:id/send-to-kitchen
  API->>K: WS order:updated
  K->>API: POST /orders/:id/kitchen-status preparing
  API->>W: WS order:updated + notification
  K->>API: POST /orders/:id/kitchen-status ready
  API->>W: notification: order ready
  WS->>API: POST /orders/:id/mark-served
```

---

## 5. Modification & collaboration

```mermaid
flowchart LR
  WS[Waitstaff modification request] --> N[Order note / kitchen:request event]
  N --> Q[Kitchen queue badge]
  Q --> C[Chef accept/reject]
  C --> WS2[Waitstaff notified]
```

---

## 6. Station routing (target)

```mermaid
flowchart TB
  T[New ticket] --> R{Auto-assign?}
  R -->|yes| S[Station from menu item map]
  R -->|no| M[Expeditor manual route]
  S --> ST[Station column filter]
  ST --> CH[Assigned chef claims]
```

---

## 7. Internal messages

```mermaid
flowchart TB
  subgraph public [Operational channels]
    SC[Station chat]
    MB[Manager broadcast]
    WA[Waitstaff kitchen channel]
  end

  subgraph private [E2EE DMs]
    DM[Chef ↔ Chef encrypted]
  end

  SC --> API[Chat API + WS]
  MB --> API
  WA --> API
  DM --> E2EE[Client encrypt → ciphertext only on server]
```

---

## 8. Offline action sync

```mermaid
sequenceDiagram
  participant FE as Kitchen UI
  participant IDB as IndexedDB
  participant API as API

  FE->>FE: Network lost
  FE->>IDB: Queue mark-ready action
  FE->>FE: Optimistic UI update
  FE->>FE: Network restored
  FE->>API: Replay queued actions
  API-->>FE: ACK or conflict
  FE->>IDB: Clear synced queue
```

---

## 9. Manager monitoring

```mermaid
flowchart LR
  M[/manager/orders] --> R[Read-only queue mirror]
  M --> A[/chef/analytics read-only export]
  M --> I[Optional kitchen:intervene]
  I --> P[Priority bump / cancel with reason]
```

---

## 10. Tickets history replay

```mermaid
flowchart LR
  H[/chef/history] --> Q[Search filters]
  Q --> T[Timeline API]
  T --> E1[sent_to_kitchen @ T1 by waitstaff]
  T --> E2[preparing @ T2 by chef A]
  T --> E3[ready @ T3 by chef A]
  T --> E4[served @ T4 by waitstaff]
```

---

## 11. Inventory impact (read-only)

```mermaid
flowchart LR
  I[/chef/inventory] --> L[Low stock API]
  L --> M[Map to menu items]
  M --> Q[Highlight affected tickets in queue]
```

---

## Cross-references

- Dine-in system flow: [../../workflow/SYSTEM_FLOWS.md](../../workflow/SYSTEM_FLOWS.md) §4
- Waitstaff handoff: [../waitstaff/FLOWS.md](../waitstaff/FLOWS.md)
- API map (planned extensions): [../../workflow/API_MAP.md](../../workflow/API_MAP.md)
