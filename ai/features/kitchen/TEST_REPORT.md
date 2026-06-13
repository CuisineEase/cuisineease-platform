# Kitchen Dashboard — Test Report (Release Candidate)

**Date:** 2026-06-11

---

## Automated

| Suite | Status | Notes |
|-------|--------|-------|
| Backend `npm run build` | Pass | TypeScript compile |
| Frontend `npm run build` | Pass | Next.js production build |
| Frontend `npm run check-types` | Pass | Zero TS errors |
| Frontend `npm run lint` | Pass | No kitchen errors |
| `kitchen.service.spec.ts` | Pass | Service bootstrap |
| Playwright `e2e/` (chromium) | **17 pass, 4 skipped** | Auth guards, regression, responsive; authenticated tests need `E2E_*_TOKEN` |

Run: `cd app && npm run test:e2e`

---

## Manual / staging checklist

- [ ] Chef queue DnD — reorder within column, move between columns, rollback on conflict
- [ ] Multi-select drag (shift+click)
- [ ] Keyboard drag via @dnd-kit keyboard sensor
- [ ] Touch drag on tablet viewport
- [ ] Offline action enqueue → sync banner → `/kitchen/sync`
- [ ] Manager `/manager/kitchen` read-only (no chef action buttons)
- [ ] E2EE device key register → public key fetch → encrypt/decrypt round-trip
- [ ] Chat reaction toggle, message pin
- [ ] Analytics CSV export
- [ ] TV display auto-rotate + high-contrast theme
- [ ] EN/FR locale switch on kitchen + manager monitor

---

## Concurrency / realtime

- WS `kitchen:queue:updated` invalidates React Query cache
- Optimistic version conflict returns 400 → UI rollback verified in DnD mutation

---

## Sign-off

Core automated gates green. Staging manual checklist recommended before production cutover.
