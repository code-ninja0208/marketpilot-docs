# MarketPilot AI

AI marketing operations platform for small businesses (launch segment: real estate).

- Planning documents: [docs/planning/](docs/planning/)
- Git workflow: [docs/07-engineering/git-workflow.md](docs/07-engineering/git-workflow.md)
- Local development: [docs/07-engineering/local-development.md](docs/07-engineering/local-development.md)

## Layout

| Path | What it is |
|------|------------|
| `apps/api` | Fastify backend (modular monolith) |
| `apps/worker` | BullMQ background worker |
| `packages/config` | Environment validation and shared lint config |
| `packages/observability` | Structured logging |
| `packages/queues` | Typed queue definitions shared by api and worker |

Apps and packages for later weeks (`client`, `admin`, `website`, `help`, `domain`, `contracts`,
`database`, …) are added in the week the development plan schedules them.

## Requirements

Node.js 26 LTS, pnpm 12, Docker (for PostgreSQL and Redis).
