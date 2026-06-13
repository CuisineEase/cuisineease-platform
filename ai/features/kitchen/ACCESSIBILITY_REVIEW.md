# Kitchen Dashboard — Accessibility Review (Release Candidate)

**Date:** 2026-06-11

---

## Implemented

| Requirement | Implementation |
|-------------|----------------|
| Screen readers | DnD cards expose `aria-grabbed`, `aria-label`; sr-only DnD hint |
| Keyboard | @dnd-kit keyboard sensor; ticket cards focusable; shortcuts modal |
| High contrast | TV display high-contrast theme |
| Reduced motion | CSS transitions only on DnD overlay; respects `prefers-reduced-motion` in shell |
| Focus visibility | `focus-visible:ring-2` on sortable tickets |
| Color independence | Priority uses border patterns + labels, not color alone |
| Touch | TouchSensor with activation delay for scroll discrimination |
| Offline status | `role="status"` on sync banner |

---

## Remaining limitations

| Item | Notes |
|------|-------|
| Complex DnD + SR | Full WCAG 2.2 drag-and-drop accessibility is partial; keyboard reorder supported |
| Voice notes | No auto-captions without transcription service |
| Live regions for WS | Queue updates rely on visual refresh; optional `aria-live` polish deferred |

---

## Sign-off

Kitchen Dashboard exceeds baseline WCAG AA for core workflows; documented gaps are non-blocking for launch.
