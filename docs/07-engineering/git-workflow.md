# Git Workflow

Author identity for this repository: `code-ninja0208 <contributor0208@hotmail.com>` (set in the repo's local git config).

## Branches

| Branch | Purpose | Created from | Merges into |
|--------|---------|--------------|-------------|
| `main` | Released, deployable code. Only receives merges from `develop` (and hotfixes). | — | — |
| `develop` | Integration branch for ongoing work. | `main` | `main`, when a module is finished |
| `feat/<task-id>-<name>` | One module from `03-development-plan.md`, e.g. `feat/W2-3-tenant-context` | `develop` | `develop` |
| `fix/<issue-no>-<name>` | Fix for one issue (failed test), e.g. `fix/12-claim-race` | `develop` | `develop` |

## Commits

- Commit **each time a function or module is finished**, together with its tests.
- Format: [Conventional Commits](https://www.conventionalcommits.org/) with the task ID as scope.
  - `feat(W2-4): add can() and assertCan() role policy`
  - `test(W7-1): cover every state × event in transition()`
  - `fix(#12): reject claim when lease has not expired`
  - `docs(W0-4): define schema v1`
- A module's last commit leaves its tests passing.

## Module finished → merge

1. All tests for the module pass on its `feat/*` branch.
2. Merge into `develop` with `--no-ff`, so the module stays visible as one unit in the history.
3. Merge `develop` into `main` with `--no-ff` once `develop` is green.

## Failed tests → issue → fix branch

1. A failing test gets an **issue**: title `Test failed: <test name>`, with the failure output, the
   branch/commit, and the steps to reproduce.
2. Create `fix/<issue-no>-<name>` from `develop`.
3. Fix it, with commits that reference the issue (`fix(#<no>): ...`); the last commit says `Closes #<no>`.
4. Merge into `develop`; **close the issue** once the fix is merged and the tests pass.

## Release tags

Annotated tags on `main`, using semantic versioning, one per milestone in `03-development-plan.md`:

| Tag | Milestone |
|-----|-----------|
| `v0.1.0` | M1 Secure foundation (end of Week 2) |
| `v0.2.0` | M2 Campaign preparation (Week 6) |
| `v0.3.0` | M3 Controlled publishing (Week 9) |
| `v0.4.0` | M4 Commercial beta (Week 11) |
| `v1.0.0-pilot` | M5 Pilot launch (Week 12) |

Patch tags (`v0.x.y`) mark fixes released to `main` between milestones.
Command: `git tag -a v0.1.0 -m "M1: secure foundation"`.
