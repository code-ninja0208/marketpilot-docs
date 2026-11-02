# Deviations from the development plan

Every place where the implementation differs from `planning/03-development-plan.md`, why, and
when it is resolved. Updated at each milestone.

| Task | Plan | What was built | Why | Resolve by |
|------|------|----------------|-----|-----------|
| W0-2, W0-3, W0-5, W0-6 | Vision/scope/pricing docs, unit economics, PRD, Meta app | Not done (W0-1 ADRs and W0-4 schema are done) | Development started first by request; W0-6 needs the owner's Meta account | Before Week 3 (W0-5 PRD) and Week 8 (W0-6) |
| W1-1 | One monorepo | One repository per project (ADR 0001) | Owner's request | — (accepted) |
| W1-4 | Logger + OpenTelemetry tracing | Logger only | Disk and scope; nothing to trace across yet | W12 |
| W1-6 | GitHub Actions CI | Local `pnpm check` gate before every merge | No GitHub remote yet | When repositories are on GitHub |
| W2-1 | Managed identity provider | Provider-neutral JWT/JWKS verification + **development sign-in** (any email, no password) | No provider account yet; keeps every flow testable | Before any non-team user gets access |
| W2-5 | Invitation emails | `logMailer` writes the link to the API log | No email provider yet | Before pilot (W12); `main.ts` refuses production without a real mailer |
| W2-6 | Playwright E2E | End-to-end flow scripted with HTTP requests against the full Docker stack | Browser binaries do not fit the build machine's disk | W12 (run on CI or the VPS) |
| W2-7 | Admin MFA via provider | MFA read from the token's `amr` claim; development staff sign-in simulates it | Same as W2-1 | With W2-1 |
| — | TypeScript 7 | TypeScript 6.0 | typescript-eslint does not support 7 yet | When supported |
| — | Lean images | API/worker images ~1 GB | Prisma 7 CLI (needed for migrations) is large | Separate migrate image, W11 |
