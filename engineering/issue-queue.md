# Issue queue (until GitHub is connected)

Failed tests and checks are logged here with a local number `L<n>`. When the GitHub repositories
exist, each open or closed entry is created as a GitHub issue in its repository (closed ones are
closed immediately with a link to the fix commit), and this file is retired.

| No. | Repository | Title | Found on | Fix branch | Status |
|-----|-----------|-------|----------|-----------|--------|
| L1 | marketpilot-shared | Check failed: `@marketpilot/config` build: TS2591 cannot find name `process` (Node types not loaded) | `develop` @ ab37fc7 | `fix/L1-node-types` | open |

## L1 details

- **Command:** `pnpm --filter @marketpilot/config build`
- **Output:** `src/env.ts(40,48): error TS2591: Cannot find name 'process'.`
- **Cause:** `tsconfig.base.json` does not load `@types/node`; Vitest passed because it runs without type checking.
- **Also:** the merge went ahead because the build result was piped through `grep`; merges must run checks with `set -e`.
