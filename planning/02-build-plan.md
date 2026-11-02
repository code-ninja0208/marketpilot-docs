# MarketPilot AI — Building Plan

This plan builds on the fix points in `01-review-fix-points.md`. It follows three rules:

1. **Code structure**: one canonical monorepo with fixed folder conventions.
2. **Well-explained functions**: every exported function has a documented contract.
3. **Code splitting**: small modules with enforced boundaries, split per layer, per feature and per route.

Stack (unchanged from the PDF): TypeScript, pnpm + Turborepo, Next.js (client/admin/website/help),
Fastify + Zod (api), PostgreSQL + Prisma, Redis + BullMQ (worker), S3-compatible storage, Stripe.

---

## 1. Rules (apply to every pull request)

### Rule 1: Code structure
- Every backend feature is a **module** under `apps/api/src/modules/<name>/` with the same file set (§3.1).
- Business logic that the API and the worker both need lives in `packages/`, never copied.
- Pure domain logic (state machines, pricing limits, validators) has **no I/O**, so it can be unit-tested exhaustively.
- One public entry per package/module: `index.ts`. Other files are internal.

### Rule 2: Well-explained functions
Every **exported** function, class and type gets a TSDoc block containing:

| Tag | Required | Content |
|-----|----------|---------|
| summary line | always | What it does, in one sentence, from the caller's view |
| `@param` | each param | Meaning and constraints, not just the type |
| `@returns` | if not void | What the value means, including the "nothing found" case |
| `@throws` | if it throws | Each `AppError` code, and when it is thrown |
| `@sideEffects` | if any | DB writes, queue jobs, external calls, audit events |
| `@example` | public package APIs | A minimal call |

Inside function bodies, comment **why**, not what. Internal helpers need only a one-line summary.
CI enforces the documentation with `eslint-plugin-jsdoc` (`require-jsdoc` on exports).

### Rule 3: Code splitting

| Level | Rule | Enforcement |
|-------|------|-------------|
| Layer | `routes → service → repository`; `policy` and `domain` are pure and called by the service | `eslint-plugin-boundaries` |
| Module | A module imports another module only through its `index.ts` service, never its repository | `dependency-cruiser` rule in CI |
| Tenant | Every repository function takes `orgId` as its first argument | Code review + cross-tenant tests (SEC-01) |
| File size | Soft limit of 300 lines per file and 50 lines per function; split when exceeded | ESLint `max-lines`, `max-lines-per-function` (warn) |
| Frontend | One route = one chunk; heavy editors, charts and the AI chat load with `next/dynamic` | Bundle budget in CI (`@next/bundle-analyzer`) |
| Worker | One queue per job type, one processor file per queue | Folder convention |

---

## 2. Canonical repository structure (replaces the three trees in the PDF)

> **Updated by ADR 0001:** each `apps/*` entry below is now its own repository (`marketpilot-<name>`),
> and `packages/*` live in `marketpilot-shared`. The folder layout *inside* each project is unchanged.

```
marketpilot/
├── apps/
│   ├── client/                 # Customer workspace (Next.js, PWA, mobile-first)
│   │   └── src/
│   │       ├── app/            # Routes only: thin pages that compose features
│   │       │   ├── (auth)/
│   │       │   └── (workspace)/[orgSlug]/
│   │       │       ├── home/  campaigns/  calendar/  approvals/
│   │       │       ├── assistant/  analytics/  accounts/  settings/
│   │       ├── features/       # One folder per feature (see §3.3)
│   │       ├── components/     # App-specific shared components
│   │       └── lib/            # api client, auth hooks, formatting
│   ├── admin/                  # Internal staff console (separate auth boundary)
│   ├── website/                # Public marketing site + pricing + signup (static)
│   ├── help/                   # Help center (static MDX + search)
│   ├── api/                    # Fastify backend (modular monolith)
│   │   └── src/
│   │       ├── modules/        # identity, organizations, businesses, strategy,
│   │       │                   # campaigns, content, approvals, publishing,
│   │       │                   # integrations, media, billing, usage, analytics,
│   │       │                   # comments, audit, admin
│   │       ├── plugins/        # auth, tenant-context, idempotency, error-handler, rate-limit
│   │       ├── http/           # error envelope, pagination helpers
│   │       └── server.ts
│   └── worker/                 # BullMQ processors
│       └── src/
│           ├── queues/         # dispatch, publish, ai-generate, metrics, reconcile, outbox
│           ├── processors/     # one file per queue
│           └── main.ts
├── packages/
│   ├── domain/                 # PURE logic: publication state machine, approval rules, plan limits
│   ├── contracts/              # Zod schemas + inferred types shared by api and frontends
│   ├── database/               # Prisma schema, migrations, tenant-scoped repository helpers
│   ├── ai/                     # Provider adapter, prompts, output schemas, evals/
│   ├── connectors/             # Social/email/shop adapters behind one interface each
│   ├── queues/                 # Job names, payload schemas, enqueue helpers
│   ├── industry/               # real-estate/, sales/, ecommerce/: templates, rules, compliance
│   ├── auth/                   # Session verification, permission policies
│   ├── config/                 # Env validation (Zod), per-app config loaders
│   ├── observability/          # Logger, tracing, metrics
│   └── ui/                     # Design system (shadcn/ui based)
├── docs/                       # Internal docs (structure from PDF p.80–81)
├── infrastructure/docker/, deployment/
├── .github/workflows/
├── docker-compose.yml, pnpm-workspace.yaml, turbo.json, package.json
```

**Allowed dependency directions** (anything else fails CI):

```
apps/*  ──►  packages/contracts, ui, config, observability
apps/api, apps/worker  ──►  packages/domain, database, ai, connectors, queues, industry, auth
packages/domain  ──►  (nothing: pure)
packages/database  ──►  packages/domain (types only)
```

The frontends never import `database`, `connectors` or `ai`. The worker never imports `apps/api`.

---

## 3. Splitting conventions

### 3.1 Backend module layout (example: `approvals`)

```
apps/api/src/modules/approvals/
├── index.ts               # Public surface: exports approvalService only
├── approvals.routes.ts    # HTTP: parse with contracts, call service, map result → response
├── approvals.service.ts   # Use cases: orchestrates policy, domain, repository, audit, transaction
├── approvals.repository.ts# SQL/Prisma only; every function takes orgId first
├── approvals.policy.ts    # Pure: who may do what (reads the role matrix)
├── approvals.errors.ts    # AppError codes owned by this module
└── __tests__/
    ├── approvals.service.int.test.ts
    └── approvals.policy.test.ts
```

| Layer | May do | Must not do |
|-------|--------|-------------|
| routes | Validate input, read `req.tenant`, call one service method, shape response | Business rules, DB access |
| service | Transactions, call policy/domain/repository, write audit + outbox | Know about HTTP (no `reply`, no status codes) |
| repository | Queries scoped by `orgId` | Permission checks, business decisions |
| policy / domain | Pure functions returning results or throwing `AppError` | Any I/O |

### 3.2 Worker layout
`queues/<name>.ts` defines the job name and Zod payload. `processors/<name>.processor.ts` contains
one exported `process<Name>` function that receives its dependencies as an argument, so tests can
inject fakes.

### 3.3 Frontend feature layout (example: `features/campaigns/`)

```
features/campaigns/
├── api.ts                 # Typed calls built on packages/contracts
├── hooks/                 # useCampaigns, useCreateCampaign (TanStack Query)
├── components/            # CampaignCard, CampaignForm, CampaignStatusBadge
├── screens/               # CampaignListScreen, CampaignDetailScreen (used by app/ routes)
└── index.ts
```

Route files in `app/` stay under ~30 lines and only render a screen. Heavy pieces are
lazy-loaded:

```tsx
// app/(workspace)/[orgSlug]/campaigns/[id]/edit/page.tsx
import dynamic from "next/dynamic";

/** Content editor is ~200 KB (rich text + media); load it only on this route. */
const ContentEditorScreen = dynamic(
  () => import("@/features/content/screens/ContentEditorScreen"),
  { loading: () => <EditorSkeleton /> }
);

export default function Page({ params }: { params: { id: string } }) {
  return <ContentEditorScreen campaignId={params.id} />;
}
```

---

## 4. Reference implementations (fixes from the review, written to Rule 2)

### 4.1 Pure publication state machine: `packages/domain/src/publication/transition.ts`

This fixes review items C1–C6: every state in the diagram can be reached and left.

```ts
export type PublicationState =
  | "draft" | "pending_approval" | "approved" | "scheduled" | "queued"
  | "publishing" | "retry_waiting" | "reconciliation_required" | "manual_review"
  | "published" | "failed" | "canceled";

export type PublicationEvent =
  | "submit" | "approve" | "reject" | "edit" | "schedule" | "due" | "lease_acquired"
  | "confirmed_success" | "transient_error" | "definitive_error" | "unknown_outcome"
  | "backoff_elapsed" | "reconciled_published" | "reconciled_not_published"
  | "reconcile_unresolved" | "admin_retry" | "admin_resolve_published"
  | "admin_resolve_not_published" | "cancel";

/** Allowed transitions. Anything not listed here is invalid. */
const TABLE: Record<PublicationState, Partial<Record<PublicationEvent, PublicationState>>> = {
  draft:            { submit: "pending_approval", cancel: "canceled" },
  pending_approval: { approve: "approved", reject: "draft", edit: "draft", cancel: "canceled" },
  approved:         { schedule: "scheduled", edit: "draft", cancel: "canceled" },
  scheduled:        { due: "queued", edit: "draft", cancel: "canceled" },
  queued:           { lease_acquired: "publishing", cancel: "canceled" },
  publishing: {
    confirmed_success: "published",
    transient_error: "retry_waiting",
    definitive_error: "failed",
    unknown_outcome: "reconciliation_required",
  },
  retry_waiting:    { backoff_elapsed: "queued", definitive_error: "failed", cancel: "canceled" },
  reconciliation_required: {
    reconciled_published: "published",
    reconciled_not_published: "retry_waiting",
    reconcile_unresolved: "manual_review",
  },
  manual_review: {
    admin_resolve_published: "published",
    admin_resolve_not_published: "failed",
  },
  failed:    { admin_retry: "queued" },
  published: {},
  canceled:  {},
};

/**
 * Computes the next publication state for an event, without touching the database.
 *
 * Keep this function pure so every transition can be unit-tested exhaustively.
 * The caller persists the result with an optimistic `version` check.
 *
 * @param current - The state currently stored on the post target.
 * @param event - What happened (user action, scheduler tick or worker outcome).
 * @returns The state the post target must move to.
 * @throws AppError("APPROVAL_REQUIRED") when scheduling from a state that is not `approved`.
 * @throws AppError("INVALID_TRANSITION") for every other event the current state does not accept.
 * @example
 *   transition("approved", "schedule"); // "scheduled"
 */
export function transition(current: PublicationState, event: PublicationEvent): PublicationState {
  const next = TABLE[current][event];
  if (next) return next;
  if (event === "schedule") throw new AppError("APPROVAL_REQUIRED");
  throw new AppError("INVALID_TRANSITION", { current, event });
}
```

### 4.2 Atomic claim: `apps/worker/src/publishing/claim.ts`

Fixes B2. Only one worker can hold a target, and a stale worker's writes are rejected through the fencing token.

```ts
/**
 * Atomically claims a due post target for one worker.
 *
 * Uses a single conditional UPDATE, so two workers can never both succeed (test PUB-01).
 * The returned fencing token must accompany every later write; writes carrying an older
 * token are ignored, which neutralises a worker that resumes after its lease expired.
 *
 * @param orgId - Tenant that owns the target.
 * @param targetId - Post target to claim.
 * @param workerId - Stable identifier of the calling worker process.
 * @param leaseMs - How long the claim stays valid without renewal.
 * @returns The claim with attempt id and fencing token, or `{ acquired: false }` if the target
 *          is not `queued` or another worker holds a live lease.
 * @sideEffects Updates `post_targets` (state → publishing, lease columns) and inserts a
 *              `publish_attempts` row in the same transaction.
 */
export async function claimTarget(
  db: Db, orgId: string, targetId: string, workerId: string, leaseMs: number
): Promise<Claim> { /* ... */ }
```

### 4.3 Publish processor: `apps/worker/src/processors/publish.processor.ts`

Fixes review item D: the error scopes are separate, so a recording failure can never trigger a second publish.

```ts
/**
 * Publishes one approved post target to its social provider, at most once.
 *
 * Flow: claim → re-check eligibility → record attempt → call provider → record outcome.
 * Each step has its own error handling, because the safe reaction differs:
 *  - failure before the provider call  → safe to retry;
 *  - provider error                    → classified (transient / definitive / unknown);
 *  - failure after provider success    → NEVER retry; mark for reconciliation.
 *
 * @param job - BullMQ job whose payload is `{ orgId, targetId }`.
 * @param deps - Injected services (publication, connectors, clock) so tests can fake them.
 * @returns Nothing; all outcomes are persisted on the post target and its attempt.
 * @sideEffects External post creation at the provider; DB state changes; audit events;
 *              metrics `publication_*`.
 */
export async function processPublish(job: PublishJob, deps: PublishDeps): Promise<void> {
  const { orgId, targetId } = job.data;

  const claim = await deps.publication.claim(orgId, targetId);
  if (!claim.acquired) return; // Already taken or no longer queued: nothing to do.

  const snapshot = await deps.publication.getSnapshot(orgId, targetId);
  const eligibility = deps.publication.checkEligible(snapshot); // pure: approval, account, quota
  if (!eligibility.ok) {
    // No external call happened, so this is a definitive failure, not a retry candidate.
    await deps.publication.recordOutcome(claim, { kind: "definitive_error", reason: eligibility.reason });
    return;
  }

  let result: PublishResult;
  try {
    result = await deps.connectors.for(snapshot.provider).publish({
      account: snapshot.account,
      revision: snapshot.approvedRevision,
      attemptId: claim.attemptId,
    });
  } catch (error) {
    // Timeouts classify as "unknown_outcome" → reconciliation, never a blind resend.
    await deps.publication.recordOutcome(claim, classifyProviderError(error));
    return;
  }

  try {
    await deps.publication.recordOutcome(claim, {
      kind: "confirmed_success",
      externalPostId: result.externalPostId,
    });
  } catch {
    // The post exists at the provider but we could not save that fact.
    // If this write fails too, the reconcile sweep finds an attempt with no outcome
    // after the lease expires and handles it the same way.
    await deps.publication.markReconciliationRequired(claim, result.externalPostId);
  }
}
```

### 4.4 Thin route → service split (pattern for all modules)

```ts
// approvals.routes.ts: HTTP only
app.post("/v1/post-targets/:id/approve", { schema: ApproveRequestSchema }, async (req) => {
  const data = await approvalService.approve(req.tenant, req.params.id, req.body);
  return { data };
});

// approvals.service.ts: use case only
/**
 * Approves one exact revision of a post target.
 *
 * @param ctx - Verified tenant context (orgId, userId, role) from the auth plugin.
 * @param targetId - Post target being reviewed.
 * @param input - Revision id and the `expectedVersion` the reviewer saw.
 * @returns The updated target summary with its new version.
 * @throws AppError("FORBIDDEN") if the role may not approve (policy matrix, test SEC-02).
 * @throws AppError("STALE_VERSION") if the target or revision changed since it was read (409, APP-01/02).
 * @sideEffects Inserts `approvals` and `audit_events` rows in one transaction.
 */
export async function approve(ctx: TenantContext, targetId: string, input: ApproveInput) { /* ... */ }
```

---

## 5. Pre-build decisions (Week 0, before any code)

Resolve every item in review section H, and adopt the fixes for A1–A6 in `docs/00-project/`.
Without the launch industry and the final pricing, the onboarding and plan-limit code
would have to be rewritten later.

---

## 6. Phased build plan

The scope is cut to make 12 weeks realistic (review item A4): **one industry, Facebook + Instagram only,
no chat, no community, no email sending.**

| Week | Deliverable | Main packages / modules | Definition of done |
|------|-------------|-------------------------|--------------------|
| 0 | Decisions (H1–H6), fixed docs: vision, pricing, PRD v1, schema v1 | `docs/` | Documents reviewed; Meta developer app registered |
| 1 | Monorepo, CI (lint, typecheck, test, jsdoc, boundaries), Docker Compose, `config`, `observability`, health endpoints | all skeletons | `pnpm turbo build test lint` green in CI; staging deploy works |
| 2 | Identity, organizations, memberships, invitations, roles, tenant-context plugin, admin skeleton + staff auth | `identity`, `organizations`, `auth`, `apps/admin` | SEC-01 cross-tenant test passes; admin requires MFA |
| 3 | Business profile + industry onboarding (launch segment), brand facts | `businesses`, `industry/<segment>` | Mobile onboarding flow under 5 minutes in a usability test |
| 4 | Strategy + objectives + campaign briefs (guided mode only) | `strategy`, `campaigns`, `domain` | User can go from goal to saved campaign brief |
| 5 | AI generation: adapter, structured output, fact validation, credits reserve/settle, evals | `ai`, `usage`, `ai_generations` | Eval suite: no invented prices/claims on golden set; quota enforced |
| 6 | Content revisions, media upload intents, comments | `content`, `media`, `comments` | Edits create immutable revisions; media validated |
| 7 | Approvals + state machine (`domain/publication`) + audit | `approvals`, `audit`, `domain` | 100% transition-table test coverage; APP-01/02 pass |
| 8 | OAuth connections (Meta), token encryption and refresh, idempotency plugin | `integrations`, `connectors/meta` | SEC-03 passes; tokens never in logs |
| 9 | Scheduler, dispatcher, claim, publish processor, reconcile sweep | `worker`, `queues`, `publishing` | PUB-01…07 and OPS-01 pass |
| 10 | Status UI, calendar, basic analytics (metric snapshots) | `analytics`, client features | US-10 acceptance criteria pass |
| 11 | Billing (Stripe), plan limits, free-tier caps, admin operations page | `billing`, `admin` | PAY-01/02 pass; free-tier cost cap enforced |
| 12 | Security review, backup-restore drill, help center (getting started), pilot with 5 businesses | `apps/help`, runbooks | All release quality gates (PDF p.48) met |

**After beta**: second and third industry modules, Shopify (GraphQL Admin API, read-only first),
email with consent/suppression (review items B12, G1), research sources with provenance,
professional workspace mode, team chat, and later the community app.

---

## 7. Per-pull-request checklist

- [ ] Code sits in the correct layer and module; no forbidden imports (CI).
- [ ] Every exported function has a TSDoc summary, `@param`, `@returns`, `@throws`, `@sideEffects`.
- [ ] Repository functions take `orgId` first; a cross-tenant test exists for new endpoints.
- [ ] New state changes go through `packages/domain` pure functions.
- [ ] New routes are lazy-loaded if they include editors, charts or chat.
- [ ] Files ≤ 300 lines, functions ≤ 50 lines, or a comment explains the exception.
- [ ] Contract (Zod) updated in `packages/contracts`; API docs regenerated.
