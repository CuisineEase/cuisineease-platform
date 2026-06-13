# CuisineEase — Project Setup & Structure

**Last updated:** 2026-06-11

How the codebase is organized and how components connect at runtime.

---

## Repository layout

```
cuisineEase/                 ← local workspace (no root git)
├── app/                     ← Frontend — CuisineEase/frontend-web
├── backend/                 ← API — CuisineEase/backend
├── ai/                      ← Specs, workflow, features (docs only)
└── docs/                    ← Miscellaneous
```

| Repo | Branch | Remotes |
|------|--------|---------|
| `backend/` | `main` | `origin` → CuisineEase/backend |
| `app/` | `main` | `origin` → CuisineEase/frontend-web, `personal` → TamahJustene/CuisineEase-frontend |

---

## Runtime topology

| Component | Host | URL |
|-----------|------|-----|
| Frontend | Vercel | `https://www.cuisineease.com` |
| API + WebSocket | Render | `https://api.cuisineease.com` |
| Database | PostgreSQL | via `DATABASE_URL` |
| Auth | Firebase | Project `cuisine-ease` |
| Storage | Firebase Storage | default provider |
| Cache | Redis | optional via env |

---

## Backend structure

```
backend/src/
├── auth/              Login, register, Google, JWT
├── user/              Profiles, roles, favorites, device tokens
├── restaurant/        Restaurant CRUD, approval, schedules
├── menu-item/         Menu + categories
├── cart/              Pre-checkout cart
├── orders/            Order lifecycle + RBAC
├── floor/             TableSession, Guest, floor map, billing
├── tables/            Physical tables (⚠️ frozen controller — SEC-001)
├── reservations/      Bookings
├── payment/           Payment records + MeSomb
├── delivery/          Driver assignments
├── chat/              Messaging
├── notifications/     Inbox + Socket.io gateway
├── common/
│   ├── firebase/      FirebaseGuard
│   ├── restaurant-access/  Multi-tenant RBAC helpers
│   └── permissions/   menu.permissions.ts
└── lib/database/migrations/
```

### Request pipeline

```
HTTP Request
  → FirebaseGuard (global)
  → @Public() bypass (selective)
  → RolesGuard (selective controllers)
  → ValidationPipe
  → Controller → Service → TypeORM
  → apiSuccess({ data })
```

### Key services

| Service | Use when |
|---------|----------|
| `RestaurantAccessService.assertWaitstaffOrAbove()` | Floor, sessions, staff order reads |
| `RestaurantAccessService.assertManagerOrAbove()` | Manager-only ops |
| `RestaurantAccessService.assertRestaurantIdMatch()` | Path vs body `restaurantId` |
| `MENU_READ_ROLES` / `MENU_WRITE_ROLES` | Menu endpoint guards |

---

## Frontend structure

```
app/src/
├── app/
│   ├── (web)/         Public marketing + menu
│   ├── auth/          Login, register, verify
│   ├── customer/      Customer portal
│   ├── manager/       Restaurant manager portal
│   ├── waitstaff/     Waitstaff portal
│   ├── chef/          Kitchen display
│   ├── delivery/      Driver portal
│   └── dashboard/     Platform admin
├── components/        UI by domain
├── services/          API clients
├── hooks/             React Query + realtime
├── config/            *Nav.ts per role
├── lib/               api.ts, orderStatusViews.ts, auth helpers
└── middleware.ts      JWT role → portal redirect
```

### Auth flow

1. User signs in → `POST /auth/login` | `/auth/google`
2. Frontend `persistAuthSession()` — token in cookie + localStorage
3. `middleware.ts` reads JWT → redirects to allowed prefix
4. API calls attach Firebase JWT via Axios interceptor

### Post-login routing (`getPostLoginPath`)

| Condition | Destination |
|-----------|-------------|
| `role === admin` | `/dashboard` |
| `role === waitstaff` | `/waitstaff` |
| `role === chef` | `/chef` |
| `role === delivery` | `/delivery` |
| `restaurantId` claim or manager/owner role | `/manager` (or `/manager/pending` if not approved) |
| default `user` | `/customer` |

---

## Multi-tenancy rules

1. Every restaurant-scoped API must validate caller has a role for that `restaurantId`
2. Never trust client-only filtering — enforce on backend
3. Path `restaurantId` must match body when both present
4. ID guessing must return 403/404, not another tenant's data

---

## Database migrations (production order)

Run after deploy when schema changes:

```bash
cd backend && npm run build && npm run migration:run
```

Notable recent migrations:

| Migration | Purpose |
|-----------|---------|
| `1772870000000-ChatTables` | Chat module |
| `1772880000000-WaitstaffFloor` | TableSession, Guest, order extensions |
| `1772890000000-TableSessionBillRequested` | `billRequestedAt` |
| `1772900000000-TableSessionExpired` | Session `expired` status |

---

## Local development

```bash
# Backend
cd backend && npm install && npm run start:dev

# Frontend
cd app && npm install && npm run dev
```

Required env: see `backend/.env.example` and `app/.env.local` (API URL, Firebase keys).

Pre-commit hooks run format, lint, types, and **build** on both repos.
