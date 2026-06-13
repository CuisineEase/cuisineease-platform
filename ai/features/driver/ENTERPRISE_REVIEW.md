# Driver Dashboard — Enterprise Review

**Date:** 2026-06-11  
**Benchmark:** Uber Eats Driver · DoorDash Dasher · Bolt Food Courier · Glovo · Deliveroo Rider

---

## Feature comparison

| Capability | Enterprise apps | CuisineEase Driver |
|------------|-----------------|-------------------|
| Shift clock in/out | ✅ | ✅ |
| Availability toggle | ✅ | ✅ available/busy/offline |
| Assignment accept | ✅ | ✅ |
| Turn-by-turn navigation | ✅ In-app | 🟡 External Maps links |
| Proof photo | ✅ Camera | 🟡 URL field (API ready) |
| OTP verification | ✅ | ✅ |
| Customer chat | ✅ | ✅ ChatInbox |
| Earnings / performance | ✅ | ✅ Basic KPIs |
| Delivery history | ✅ | ✅ |
| Offline mode | ✅ | 🟡 Backend sync; partial UI |
| Stacked orders | ✅ | ⬜ Future |
| Live GPS to customer | ✅ | ⬜ Future |
| Heat map / demand zones | ✅ | ⬜ N/A (single restaurant) |
| Ratings | ✅ | ⬜ Stub |

---

## CuisineEase-fit improvements implemented

1. **Restaurant-scoped drivers** — not marketplace gig workers; simpler RBAC
2. **Integrated chat** — same infrastructure as customer/manager/waitstaff
3. **Manager assignment** — dispatch from `/manager/orders` not auto-offer model
4. **Audit timeline** — `delivery_events` for dispute resolution
5. **Bilingual EN/FR** — required for Cameroon market

---

## Recommended phase 2 (enterprise polish)

| Priority | Feature |
|----------|---------|
| P1 | Camera proof upload via Firebase storage |
| P1 | Frontend offline queue (IndexedDB) |
| P2 | Customer live ETA via WebSocket |
| P2 | Signature pad component |
| P3 | Route ordering for multiple active deliveries |
| P3 | Push notification on new assignment |

---

## Verdict

CuisineEase Driver Portal meets **internal restaurant fleet** enterprise bar. Marketplace-scale features (heat maps, auto-dispatch, earnings payouts) are intentionally out of scope for v1.
