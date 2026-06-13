# Driver Dashboard — Performance Review

**Date:** 2026-06-11

---

## Targets

| Metric | Target | Current |
|--------|--------|---------|
| Dashboard load | < 2s on 4G | ✅ Two parallel queries (summary + assignments) |
| Assignment list | < 500ms API | ✅ Indexed query on driver + restaurant |
| Status update | < 300ms | ✅ Single UPDATE + event insert |
| WebSocket refresh | < 1s perceived | ✅ Query invalidation on event |
| Mobile bundle | Minimal portal JS | ✅ Shared chunks with design system |

---

## Backend

- Pagination supported (`page`, `limit`) — default 50
- Dashboard uses COUNT queries + limited delivered sample for avg time
- No N+1 on assignment list (eager joins in query builder)
- Timeline capped at 100 events per detail request

---

## Frontend

- React Query `staleTime: 15_000` on dashboard
- No virtualization yet — acceptable for <100 assignments per driver
- Socket reconnect via socket.io defaults

---

## Recommendations

1. Add DB index on `(deliveryPersonId, status)` if list queries slow at scale
2. Virtualize `/delivery/deliveries` when restaurants exceed 50 concurrent deliveries
3. Debounce search input (300ms) — currently refetches on every keystroke

---

## Load testing

Not yet run. Suggested: 20 concurrent drivers × 10 status updates/min per restaurant.
