# Electronics Store with PC Builder

A student team project for a single-seller electronics store in Vietnam, with a PC builder, constrained recommendations, safe inventory checkout and recoverable MoMo payments. Currency is VND; VNPAY is a later integration.

**Status:** requirements, design and implementation plans are prepared. Application implementation has not started. The twelve-week target is an integrated core milestone; accepting real customers and live payments requires the separate production launch gates.

## Documentation

- [Documentation index](docs/README.md): reading order, decisions, dependencies and requirement coverage.
- [Product requirements](docs/ELECTRONICS_STORE_PRD.md).
- [Technical requirements](docs/ELECTRONICS_STORE_TRD.md).
- [Design specification](docs/superpowers/specs/2026-10-06-electronics-store-design.md).
- [Team tasks and reviewers](docs/TEAM_TASK_ASSIGNMENTS.md).
- [Individual task guides](docs/tasks/README.md): detailed execution cards for Phat, Bảo and Huy.

## Team

| Member | Main responsibility |
| --- | --- |
| Phat | Java backend, accounts, cart, checkout and payments |
| Bảo | React frontend, integration and delivery coordination |
| Huy | Catalog, hardware rules, saved builds and Python optimizer |

The task assignment document gives exact ownership, reviewers and dependencies, including the Java services Huy implements with Phat's review.

## Repository layout

The current repository contains documentation and Git configuration:

```text
.
├── README.md
├── .gitignore
├── .gitattributes
└── docs/
    ├── README.md
    ├── ELECTRONICS_STORE_PRD.md
    ├── ELECTRONICS_STORE_TRD.md
    ├── TEAM_TASK_ASSIGNMENTS.md
    ├── tasks/
    │   ├── README.md
    │   ├── PHAT.md
    │   ├── BAO.md
    │   └── HUY.md
    └── superpowers/
        ├── specs/
        └── plans/
```

Implementation tasks will add `backend/` (Java/Spring), `frontend/` (Next.js/React), `optimizer/` (Python/FastAPI), `contracts/`, and integration scripts/configuration. Each service belongs in this shared repository. Create these directories when their first files are implemented.

## Start contributing

1. Read the documentation index and your task table.
2. Begin with F1/F2 for Phat, U1 for Bảo, and B3 for Huy; follow the recorded dependencies for subsequent tasks.
3. Work on a task branch and have its assigned reviewer review the pull request before merging into `main`.
4. Record the relevant verification command and result. Follow the implementation plan's tests; planned results are not evidence of executed checks.

Commit dependency lockfiles, migrations and safe example configuration. Local environment files, private keys and generated build artifacts are excluded by `.gitignore`. Service setup commands will be documented when the scaffolds exist.
