# ADR 0007: PostgreSQL with Prisma, shared through marketpilot-shared

- Status: accepted
- Date: 2026-11-02

## Decision
The schema and migrations live in `marketpilot-shared/packages/database` (Prisma). The API and the
worker import the generated client from there. Every generated migration's SQL is reviewed in its
pull request; constraints Prisma cannot express (partial or composite unique indexes) are added
as raw SQL in the migration.

Every tenant-owned table carries `org_id` (review item B7), and every repository function takes
`orgId` as its first parameter.
