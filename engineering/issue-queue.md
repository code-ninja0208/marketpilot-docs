# Issue queue (until GitHub is connected)

Failed tests and checks are logged here with a local number `L<n>`. When the GitHub repositories
exist, each open or closed entry is created as a GitHub issue in its repository (closed ones are
closed immediately with a link to the fix commit), and this file is retired.

| No. | Repository | Title | Found on | Fix branch | Status |
|-----|-----------|-------|----------|-----------|--------|
| L1 | marketpilot-shared | Check failed: `@marketpilot/config` build: TS2591 cannot find name `process` (Node types not loaded) | `develop` @ ab37fc7 | `fix/L1-node-types` | closed (marketpilot-shared 0c1776c) |
| L2 | marketpilot-shared | Check failed: Docker build: clean `pnpm install` rejects `msgpackr-extract` build script | `main` @ c14a2b4 | `fix/L2-allow-builds` | closed (marketpilot-shared 553f5c8, marketpilot-worker 85e2ec9) |
| L3 | marketpilot-worker | Integration test failed once: `prisma-store.int.test.ts` both tests threw `PrismaClientKnownRequestError` | `feat/W5-3-generation-processor` (uncommitted) | — (not reproducible) | closed: passed 3 reruns; no code change |

## L1 details

- **Command:** `pnpm --filter @marketpilot/config build`
- **Output:** `src/env.ts(40,48): error TS2591: Cannot find name 'process'.`
- **Cause:** `tsconfig.base.json` does not load `@types/node`; Vitest passed because it runs without type checking.
- **Also:** the merge went ahead because the build result was piped through `grep`; merges must run checks with `set -e`.

## L2 details

- **Command:** `docker compose build` (marketpilot-infra, W1-6)
- **Output:** `ERR_PNPM_IGNORED_BUILDS: Ignored build scripts: msgpackr-extract@3.0.4`
- **Cause:** BullMQ's optional native add-on wants to run an install script; `pnpm-workspace.yaml`
  `allowBuilds` lists only `esbuild`, and a clean install treats unlisted build scripts as an error.
  The local install had passed because the dependency was added incrementally.
- **Affects:** marketpilot-shared and marketpilot-worker (both depend on bullmq).

## L3 details

- **Command:** `pnpm test:integration` (marketpilot-worker), 2026-11-03 ~13:14
- **Output:** both tests failed with `PrismaClientKnownRequestError`; the error text was not captured
  (the session was interrupted while reading it).
- **Cause (likely):** a second Claude session was running Week 5 in the same working trees and
  against the same development database at the same time. Passed on three reruns with only one
  session active.
- **Prevention:** only one session works in the `marketpilot-*` repositories at a time.
