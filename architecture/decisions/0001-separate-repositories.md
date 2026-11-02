# ADR 0001: One repository per project

- Status: accepted
- Date: 2026-11-02
- Replaces: the single monorepo tree in `planning/02-build-plan.md` §2

## Decision

Every project is its own folder and its own git repository, with its own branches, tags and issues.
All repositories sit side by side in one parent folder:

| Repository | Contents | Created in |
|------------|----------|-----------|
| `marketpilot-docs` | Planning, architecture, engineering docs | W0 |
| `marketpilot-shared` | Shared packages: `config`, `observability`, `queues`; later `contracts`, `domain`, `database`, `auth`, `ui`, … | W1 |
| `marketpilot-api` | Fastify backend (modular monolith) | W1 |
| `marketpilot-worker` | BullMQ background worker | W1 |
| `marketpilot-infra` | Docker Compose, Caddy, VPS deploy scripts, daily progress page | W1 |
| `marketpilot-client` | Customer workspace (Next.js PWA) | W2 |
| `marketpilot-admin` | Internal staff console | W2 |
| `marketpilot-website` | Public site, pricing, signup | W2 |
| `marketpilot-help` | Help center | W12 |

## How shared code is consumed

`marketpilot-shared` is a pnpm workspace. Other repos depend on its packages with a sibling link,
for example `"@marketpilot/config": "link:../marketpilot-shared/packages/config"`.
The VPS deploy clones every repository into the same parent folder, so the links resolve there too.
Each release of `marketpilot-shared` is tagged; consumers record the shared tag they were tested
against in their own release notes.

## Consequences

- The dependency rules from the build plan still apply, now across repositories:
  frontends may use only `config`, `contracts`, `ui`, `observability`; they never use `database`,
  `connectors` or `ai`.
- A change to a shared package that breaks consumers needs a coordinated change in each consumer
  repository before `marketpilot-shared` is merged into its `main`.
- Each repository runs its own CI.
