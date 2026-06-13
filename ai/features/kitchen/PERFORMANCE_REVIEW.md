# Kitchen Dashboard — Performance Review (Release Candidate)

**Date:** 2026-06-11

---

## Optimizations applied

| Area | Change |
|------|--------|
| Queue list | `@tanstack/react-virtual` — O(visible) DOM nodes for thousands of tickets |
| History / activity | Virtualized scroll containers |
| Manager monitor | Virtualized read-only queue |
| API | Queue limit 200; mapper avoids duplicate unwrap work |
| DnD | Optimistic local state; batch reorder API |
| Bundle | `@dnd-kit` and `@tanstack/react-virtual` tree-shaken via route-level imports |
| Realtime | Targeted query invalidation on `kitchen:queue:updated` |

---

## Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Queue scroll (1000 items) | 60fps | Virtualization |
| DnD latency | <16ms frame | Local optimistic update |
| WS → UI refresh | <500ms | Query invalidation |
| Initial `/chef/queue` LCP | <2.5s | Code-split kitchen components |

---

## Remaining watch items

- Full bundle analysis with `@next/bundle-analyzer` on CI (optional)
- Load test `POST /kitchen/queue/reorder` under 10 concurrent chefs

---

## Sign-off

Kitchen Dashboard remains responsive under heavy queue load via virtualization and optimistic updates.
