# CuisineEase — Feature Documentation Index

**Last updated:** 2026-06-11

Each portal or major product area has a folder under `ai/features/` with a **SPEC** (what it is, routes, RBAC, components) and **FLOWS** (user journeys and API sequences).

Cross-cutting setup and definitions live in [../workflow/README.md](../workflow/README.md).

---

## Portals

| Portal | Spec | Flows | Status |
|--------|------|-------|--------|
| **Customer** | [customer/SPEC.md](./customer/SPEC.md) | [customer/FLOWS.md](./customer/FLOWS.md) | Production |
| **Manager** | [manager/SPEC.md](./manager/SPEC.md) | [manager/FLOWS.md](./manager/FLOWS.md) | Production |
| **Waitstaff** | [waitstaff/SPEC.md](./waitstaff/SPEC.md) | [waitstaff/FLOWS.md](./waitstaff/FLOWS.md) | Phase 8 complete |
| **Kitchen** | [kitchen/SPEC.md](./kitchen/SPEC.md) | [kitchen/FLOWS.md](./kitchen/FLOWS.md) | **Implemented** — see [KITCHEN_COMPLETION_REPORT.md](./kitchen/KITCHEN_COMPLETION_REPORT.md) |

---

## Other surfaces

| Surface | Path prefix | Notes |
|---------|-------------|-------|
| Delivery | `/delivery` | Driver assignments + customer chat |
| Platform admin | `/dashboard` | Restaurant approval, network oversight |
| Public | `/`, `/menu`, `/contact` | Marketing, public menus |

See [../workflow/SYSTEM_FLOWS.md](../workflow/SYSTEM_FLOWS.md) for end-to-end diagrams.

---

## Waitstaff supporting docs

| Document | Purpose |
|----------|---------|
| [waitstaff/WAITSTAFF_COMPLETION_REPORT.md](./waitstaff/WAITSTAFF_COMPLETION_REPORT.md) | Phase 8 sign-off |
| [waitstaff/UPDATED_SPEC.md](./waitstaff/UPDATED_SPEC.md) | Revised spec after implementation |
| [waitstaff/GAP_ANALYSIS.md](./waitstaff/GAP_ANALYSIS.md) | Pre-build gap analysis |
| [waitstaff/AUDIT.md](./waitstaff/AUDIT.md) | Implementation audit |
| [waitstaff/ARCHITECTURE_REVIEW.md](./waitstaff/ARCHITECTURE_REVIEW.md) | Architecture review |
| [waitstaff/SECURITY_REVIEW.md](./waitstaff/SECURITY_REVIEW.md) | Security review |
| [../security/WAITSTAFF_SECURITY_AUDIT.md](../security/WAITSTAFF_SECURITY_AUDIT.md) | Production security audit |

---

## Kitchen supporting docs

| Document | Purpose |
|----------|---------|
| [kitchen/AUDIT.md](./kitchen/AUDIT.md) | Current `/chef` baseline |
| [kitchen/GAP_ANALYSIS.md](./kitchen/GAP_ANALYSIS.md) | Current vs target |
| [kitchen/ARCHITECTURE_REVIEW.md](./kitchen/ARCHITECTURE_REVIEW.md) | Module design |
| [kitchen/SECURITY_REVIEW.md](./kitchen/SECURITY_REVIEW.md) | RBAC + E2EE plan |
| [kitchen/IMPLEMENTATION_ROADMAP.md](./kitchen/IMPLEMENTATION_ROADMAP.md) | Phased delivery **K0–K12** (v2 enterprise) |
| [kitchen/ENTERPRISE_REVIEW.md](./kitchen/ENTERPRISE_REVIEW.md) | Senior architectural review + industry comparison |

---

## When to add a feature folder

Create `ai/features/<name>/` when:

- A portal gains enough routes and domain logic to warrant its own spec
- RBAC or state machines are non-trivial
- Multiple agents will work on the area in parallel

Minimum contents: `SPEC.md` + `FLOWS.md`. Link from this index and [../workflow/README.md](../workflow/README.md).
