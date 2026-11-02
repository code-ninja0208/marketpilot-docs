# ADR 0005: Staging on the owner's VPS with Docker Compose

- Status: accepted (D4)
- Date: 2026-11-02

## Decision
Staging runs on the owner's VPS: Docker Compose with PostgreSQL, Redis, API, worker and Caddy
(see `marketpilot-infra`). The worker is a normal long-running container, not a serverless
function. Production hosting is decided before the pilot (Week 11).
