# CuisineEase Platform

**System coordination, architecture, and release control layer.**

This repository is the meta-layer for the CuisineEase multi-restaurant platform. It contains **no application source code** -- only system intelligence, governance, architecture documentation, and release artifacts.

---

## Purpose

| Concern | This repo handles it |
|---------|---------------------|
| System architecture | `docs/ARCHITECTURE_OVERVIEW.md` |
| Enterprise audit results | `ai/ENTERPRISE_PRODUCT_AUDIT.md` |
| Governance rules | `ai/GOVERNANCE.md` |
| Release gates | `docs/RELEASE_GATES.md` |
| Platform state | `ai/PLATFORM_STATE.md` |
| Feature specifications | `ai/features/` |
| Workflow & data model | `ai/workflow/` |
| Security backlog | `ai/SECURITY_BACKLOG.md` |
| Task tracking | `ai/TASKS.md` |
| Architecture decisions | `ai/DECISIONS.md` |

---

## Repository Structure

```
cuisineease-platform/
├── ai/                    <- System intelligence (source of truth)
│   ├── features/          <- Portal specs (customer, manager, kitchen, waitstaff, driver)
│   ├── security/          <- Security audits
│   └── workflow/          <- Flows, definitions, data model, API map
├── docs/                  <- Architecture + business documentation
├── release-reports/       <- Phase reports + release verification
├── deployment/            <- Deployment configuration (future)
├── README.md              <- This file
├── REPOSITORIES.md        <- Repository catalog + coordination rules
└── .gitignore
```

---

## Related Repositories

| Repository | Purpose | URL |
|------------|---------|-----|
| **Frontend** | Next.js customer/manager/staff portals | `git@github.com:CuisineEase/frontend-web.git` |
| **Backend** | NestJS API, WebSocket, migrations | `git@github.com:CuisineEase/backend.git` |
| **Platform** (this) | Architecture, AI specs, release control | `git@github.com:CuisineEase/cuisineease-platform.git` |

See [REPOSITORIES.md](./REPOSITORIES.md) for full coordination rules.

---

## Key Documents

| Document | What it tells you |
|----------|-------------------|
| [`ai/PLATFORM_STATE.md`](./ai/PLATFORM_STATE.md) | Current system status, risks, next priorities |
| [`ai/GOVERNANCE.md`](./ai/GOVERNANCE.md) | Rules for changes, deployments, documentation |
| [`docs/ARCHITECTURE_OVERVIEW.md`](./docs/ARCHITECTURE_OVERVIEW.md) | Full system architecture |
| [`docs/RELEASE_GATES.md`](./docs/RELEASE_GATES.md) | What must pass before deployment |
| [`release-reports/INITIAL_PLATFORM_REPORT.md`](./release-reports/INITIAL_PLATFORM_REPORT.md) | System maturity + RC confirmation |

---

## Current Status

**Release Candidate** -- Post Phase 1 Enterprise Audit (74/100 release score)

- Phase 0 (Critical): 8/8 complete
- Phase 1 (High): 14/14 complete
- All staff portals (waitstaff, kitchen, driver) at RC or complete
- Known risks documented in `ai/SECURITY_BACKLOG.md`
