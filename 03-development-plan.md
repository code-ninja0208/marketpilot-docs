# MarketPilot AI: Development Plan

This plan turns `02-build-plan.md` into tasks you can execute, week by week. It uses the fix points in
`01-review-fix-points.md` and the rules from `02-build-plan.md` §1: code structure,
well-explained functions, code splitting.

- **Team assumption:** 1 full-stack developer (5 working days/week). With 2–3 people, run the
  frontend (FE) and backend (BE) tasks of the same week in parallel.
- **Estimates:** in developer-days (d). Every week is planned at ~4.5d of tasks plus 0.5d of buffer.
- **Task ID format:** `W<week>-<n>`. Use these IDs in branch names (`feat/W3-2-brand-facts`) and commits.

---

## 0. Decisions with default answers

These are the open items from review §H. The plan uses the **default** answer. If you choose
differently, only the named tasks change.

| ID | Decision | Default used in this plan | Affects |
|----|----------|---------------------------|---------|
| D1 | Launch industry | **Real estate** (listings → inquiries) | W3-3, W4-2, W5-3 |
| D2 | Auth provider | Managed provider with org support (e.g., Clerk) behind `packages/auth` | W2-1 |
| D3 | AI provider | One provider behind the `packages/ai` adapter; model chosen in W5 | W5-1 |
| D4 | Worker hosting | Always-on container host (e.g., Fly.io / Railway) | W1-6, W9 |
| D5 | Meta app | Registered by the owner in Week 0; business verification started | W8, W12 |
| D6 | Pricing | Free / $19 / $59 / $149 (PDF p.61), with the free-tier AI cost cap set in W0-3 | W11 |

---

## 1. Working method

### 1.1 Branches and pull requests
- `main` is always deployable to staging. Make short feature branches per task and merge them with a squash merge.
- Every PR must pass the CI gates (§1.3) and the per-PR checklist in `02-build-plan.md` §7.
- Database changes come in their own PR with the generated SQL reviewed (expand → migrate → contract).

### 1.2 Definition of Ready (before starting a task)
- Zod contract for any new endpoint written in `packages/contracts`.
- Table columns specified in `docs/04-architecture/database-model.md`.
- Acceptance criteria copied into the task.

### 1.3 Definition of Done (every task)
- Code in the correct module/layer; boundary lint passes.
- Exported functions documented (summary, `@param`, `@returns`, `@throws`, `@sideEffects`).
- Unit tests for pure logic; integration test for each new endpoint, including one cross-tenant denial.
- Works on a 360 px mobile viewport and on desktop (client app tasks).
- Deployed to staging; the related help/doc page is updated.

### 1.4 CI pipeline (built in W1, required for every PR)
`install → lint (eslint, jsdoc, boundaries, max-lines) → typecheck → unit → integration (Postgres + Redis containers) → build → bundle-size check`.
On `main`: also run `deploy staging → smoke → Playwright E2E`.

---

## 2. Milestones

| Milestone | End of | Exit criteria |
|-----------|--------|---------------|
| **M0 Plan locked** | Week 0 | D1–D6 decided; docs baseline merged; Meta app registered |
| **M1 Secure foundation** | Week 2 | Login, orgs, roles, tenant isolation (SEC-01/02) on staging; admin skeleton with MFA |
| **M2 Campaign preparation** | Week 6 | Goal → brief → AI content → editable revisions with media, entirely in app |
| **M3 Controlled publishing** | Week 9 | Approve → schedule → publish to Meta test accounts; PUB-01…07 pass |
| **M4 Commercial beta** | Week 11 | Status UI, analytics, billing, plan limits, admin ops page |
| **M5 Pilot launch** | Week 12 | Release gates passed; 5 pilot businesses onboarded |

---

## 3. Week-by-week tasks

Each task lists the files to create, the key functions (each one explained in one line, written out in
full as TSDoc in the code), the tests, and the estimate.

### Week 0: Plan lock (docs only, no app code)

| ID | Task | Output | Est |
|----|------|--------|-----|
| W0-1 | Decide D1–D6 and record them as ADRs | `docs/04-architecture/architecture-decisions/0001…0006.md` | 0.5d |
| W0-2 | Merge the fixes for review items A1–A6 into vision, scope and pricing docs | `docs/00-project/{vision,scope-and-non-goals,pricing,roadmap}.md` | 1d |
| W0-3 | Recalculate unit economics with the free tier; set the free AI cost cap | `docs/00-project/unit-economics.md` | 0.5d |
| W0-4 | Schema v1: all MVP tables incl. fixes B1–B8, B11, B13 | `docs/04-architecture/database-model.md` | 1d |
| W0-5 | PRD v1 for the MVP user stories (US-01…US-15) + real-estate flows | `docs/00-project/product-requirements.md` | 1d |
| W0-6 | Register the Meta developer app; list the required permissions and test accounts | `docs/06-integrations/meta.md` | 0.5d |

### Week 1: Repository, CI, environment (→ start M1)

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W1-1 | Monorepo skeleton: pnpm, Turborepo, TS project refs, pinned Node | root configs, empty `apps/*`, `packages/*` (tree from 02 §2) | none | `turbo build` runs | 0.5d |
| W1-2 | Lint rules that enforce the 3 rules | `packages/config/eslint/` | none (`eslint-plugin-jsdoc`, `boundaries`, `max-lines`, `dependency-cruiser` config) | A deliberately bad file fails lint | 0.5d |
| W1-3 | Env validation | `packages/config/src/env.ts` | `loadEnv(schema)`: parses `process.env` with Zod and fails fast with a readable list of missing keys | unit | 0.5d |
| W1-4 | Observability | `packages/observability/src/{logger,tracing}.ts` | `createLogger(service)`: JSON logger with request id and token redaction; `initTracing(service)`: OpenTelemetry setup | unit: tokens redacted | 0.5d |
| W1-5 | API skeleton + error envelope + health | `apps/api/src/{server.ts,plugins/error-handler.ts,http/errors.ts}` | `buildServer(deps)`: creates the Fastify app with plugins, used by tests and `main`; `AppError(code, details)`: typed domain error mapped to an HTTP status in one place; `GET /health/live`, `/health/ready` (checks DB + Redis) | integration | 1d |
| W1-6 | Docker Compose (Postgres, Redis, MinIO) + CI workflow + staging deploy | `docker-compose.yml`, `.github/workflows/ci.yml`, `infrastructure/deployment/` | none | CI green; staging `/health/ready` returns 200 | 1d |
| W1-7 | Worker skeleton | `apps/worker/src/main.ts`, `packages/queues/src/index.ts` | `defineQueue(name, payloadSchema)`: one typed queue definition shared by the API (enqueue) and the worker (process) | unit | 0.5d |

### Week 2: Identity, tenancy, roles (→ M1)

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W2-1 | Auth integration | `packages/auth/src/session.ts`, `apps/api/src/plugins/auth.ts` | `verifySession(req)`: validates the provider session and returns `{ userId, authSubject }`, or throws `UNAUTHENTICATED` | integration: 401 without a session | 1d |
| W2-2 | Prisma schema: users, organizations, memberships, invitations, audit_events | `packages/database/prisma/schema.prisma`, migration | none | migration applies cleanly | 0.5d |
| W2-3 | Tenant context plugin | `apps/api/src/plugins/tenant-context.ts` | `resolveTenant(req)`: loads the membership for the org in the URL and sets `req.tenant = { orgId, userId, role }`; a user without membership gets 404, never 403, so they can't learn the org exists | **SEC-01** | 1d |
| W2-4 | Permission policy (role matrix, PDF p.43, + fix F1/F3) | `packages/auth/src/policy.ts` | `can(role, action)`: pure lookup in the role matrix; `assertCan(ctx, action)`: throws `FORBIDDEN` | unit: full matrix; **SEC-02** | 0.5d |
| W2-5 | Organizations + invitations module | `modules/organizations/*` | `createOrganization(userId, input)`: creates the org and the owner membership in one transaction; `inviteMember(ctx, email, role)`: stores a hashed token and sends an email; `acceptInvitation(userId, token)`: checks expiry and email match, then creates the membership | integration | 1d |
| W2-6 | Client app shell: auth pages, org switcher, mobile bottom nav / desktop sidebar | `apps/client/src/app/(auth)`, `(workspace)/[orgSlug]/layout.tsx`, `packages/ui` | none | Playwright: sign in → dashboard | 0.5d |
| W2-7 | Admin app skeleton with separate staff auth + MFA | `apps/admin`, `modules/admin/staff.policy.ts` | `assertPlatformStaff(req)`: separate check against `platform_staff`, never derived from customer roles | integration: customer owner is denied | 0.5d |

### Week 3: Businesses and real-estate onboarding

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W3-1 | Businesses module | `modules/businesses/*` | `createBusiness(ctx, input)`: validates the IANA timezone and industry; `getBusiness(ctx, id)`: scoped by `orgId` | integration + cross-tenant | 1d |
| W3-2 | Brand profile + approved facts | `modules/businesses/brand-profile.*` | `updateBrandProfile(ctx, businessId, input)`: stores tone, audience, restrictions; `upsertFact(ctx, businessId, fact)`: stores a fact the user verified (price, hours, address) with an id the AI must cite | integration | 1d |
| W3-3 | Industry module: real estate | `packages/industry/real-estate/{templates,rules,compliance}.ts` | `realEstateGoals()`: starting goals for onboarding; `checkFairHousing(text)`: flags discriminatory wording (pure, rule list + tests) | unit with a phrase list | 1d |
| W3-4 | Listings (minimal) | `modules/listings/*`, table `listings` | `createListing(ctx, businessId, input)`: stores address, price, features and photos as approved facts | integration | 0.5d |
| W3-5 | Mobile onboarding flow (Step 1 business type → Step 2 goal → first task, PDF p.77) | `features/onboarding/*` | none (autosave draft with `useAutosave`) | Playwright on a 360 px viewport | 1d |

### Week 4: Strategy, objectives, campaigns

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W4-1 | Schema: marketing_strategies, objectives, campaigns, campaign_briefs (fix B9, minimal columns) | migration | none | migration | 0.5d |
| W4-2 | Strategy module (guided mode) | `modules/strategy/*` | `createStrategy(ctx, businessId, input)`: stores the goal, audience segment, channels and KPI; `defineObjective(ctx, strategyId, kpi)`: validates that the target is measurable (metric + number + date) | unit + integration | 1.5d |
| W4-3 | Campaigns module + idempotency plugin (fix B3) | `modules/campaigns/*`, `plugins/idempotency.ts` | `createCampaign(ctx, input)`: links to an objective and validates the date range; `withIdempotency(key, hash, fn)`: replays the stored response for the same key + payload, and returns 409 for the same key with a different payload | integration: replay + conflict | 1.5d |
| W4-4 | Campaign list/detail screens (cards on mobile) | `features/campaigns/*` | none | Playwright | 1d |

### Week 5: AI content generation

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W5-1 | AI adapter + structured output | `packages/ai/src/{adapter,providers/*}.ts` | `generateStructured(schema, prompt, opts)`: calls the provider, parses the output with Zod, retries within a bounded budget, and returns usage | unit with a fake provider | 1d |
| W5-2 | Credits reserve/settle (fix B13) | `modules/usage/*` | `reserveCredits(ctx, amount)`: fails with 429 if it would exceed the plan; `settleCredits(reservationId, actual)`: records the real usage and releases the rest | integration: concurrent reserve | 1d |
| W5-3 | Generation job (fix B4, E5) | `modules/content/generation.service.ts`, `worker/processors/ai-generate.processor.ts` | `queueGeneration(ctx, campaignId, input)`: reserves credits, inserts `ai_generations`, enqueues the job, returns 202; `processGeneration(job, deps)`: builds the prompt from approved facts only, then validates against fact ids, length, platform rules and the fair-housing check | **SEC-04** prompt injection | 1.5d |
| W5-4 | Eval suite (fix G3) | `packages/ai/evals/*` | `runEvals(cases)`: golden cases; fails if any output contains a price/claim not in the facts | runs in CI (nightly) | 0.5d |
| W5-5 | Generation progress UI (polling) | `features/content/components/GenerationStatus.tsx` | none | component test | 0.5d |

### Week 6: Content revisions, media, comments (→ M2)

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W6-1 | Drafts + immutable revisions | `modules/content/*` | `createRevision(ctx, draftId, input)`: inserts `revision_no + 1` and never updates an old revision; it also emits an `edit` event for affected targets, so approval is invalidated (fix C3) | integration: APP-01 | 1d |
| W6-2 | Media upload intents (fix B8) | `modules/media/*` | `createUploadIntent(ctx, file)`: validates the mime type and size, then returns a short-lived signed URL; `confirmUpload(ctx, assetId)`: checks the checksum and dimensions | integration | 1d |
| W6-3 | Comments (fix B11) | `modules/comments/*` | `addComment(ctx, resourceType, resourceId, body)`: checks that the resource belongs to the tenant before inserting | cross-tenant test | 0.5d |
| W6-4 | Content editor screen (lazy-loaded) + device previews | `features/content/screens/ContentEditorScreen.tsx` | none | Playwright; bundle budget | 1.5d |
| W6-5 | M2 demo + fix list | none | none | E2E: goal → brief → AI → edit | 0.5d |

### Week 7: Approvals and state machine

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W7-1 | Pure state machine (02 §4.1) | `packages/domain/src/publication/transition.ts` | `transition(current, event)`: returns the next state or throws `INVALID_TRANSITION` / `APPROVAL_REQUIRED` | unit: **every** state × event pair | 1d |
| W7-2 | Post targets (fix B1, B2 columns) | `modules/publishing/targets.*` | `createTarget(ctx, draftId, accountId, placementKey)`: checks that the account belongs to the same business (tenant consistency) | integration | 0.5d |
| W7-3 | Approvals module | `modules/approvals/*` | `submit(ctx, targetId, revisionId)`; `approve(ctx, targetId, input)`: applies the role policy, the revision match and the `expectedVersion` check, and writes the audit entry in the same transaction; `reject(ctx, targetId, reason)` | **APP-01, APP-02** concurrency | 1.5d |
| W7-4 | Audit module + outbox writer | `modules/audit/*`, `packages/database/src/outbox.ts` | `recordAudit(tx, event)`: always called inside the caller's transaction; `writeOutbox(tx, event)`: the worker delivers it later | integration | 0.5d |
| W7-5 | Approvals screen (mobile approve/reject) | `features/approvals/*` | none | Playwright | 1d |

### Week 8: Meta connection and tokens

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W8-1 | Token encryption | `packages/database/src/crypto/tokens.ts` | `encryptToken(plain)` / `decryptToken(cipher)`: envelope encryption with a key from the secret store; the key never appears in logs | unit; log redaction test | 0.5d |
| W8-2 | OAuth start/callback (fix B6) | `modules/integrations/oauth.*` | `startOAuth(ctx, provider, businessId)`: stores a hashed state + PKCE verifier and returns the redirect; `completeOAuth(state, code)`: checks that the state is unused and unexpired, exchanges the code, and stores the encrypted token | integration: replayed state rejected | 1.5d |
| W8-3 | Connector interface + Meta adapters | `packages/connectors/src/{types.ts,meta/facebook.ts,meta/instagram.ts}` | `SocialConnector.validateContent / prepareMedia / publish / reconcile / fetchMetrics`; `reconcile()` may return `unknown` | contract tests with recorded fixtures | 1.5d |
| W8-4 | Token refresh + expiry job | `worker/processors/token-refresh.processor.ts` | `refreshExpiringTokens(deps)`: marks an account `reconnect_required` on failure | **SEC-03** | 0.5d |
| W8-5 | Accounts screen (connect, status, disconnect) | `features/accounts/*` | none | Playwright with a mocked provider | 0.5d |

### Week 9: Scheduling and publishing (→ M3)

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W9-1 | Schedule / cancel endpoints | `modules/publishing/schedule.*` | `schedule(ctx, targetId, input)`: requires approval, an active account and quota, and a time in the future (UTC instant); `cancel(ctx, targetId)`: allowed in the states from fix C4, rejected while `publishing` | **PUB-05, PUB-06** | 1d |
| W9-2 | Dispatcher | `worker/processors/dispatch.processor.ts` | `dispatchDue(deps, now)`: selects due targets from Postgres (the source of truth), moves them `scheduled → queued` and enqueues them with jobId = targetId | **OPS-01** Redis restart | 0.5d |
| W9-3 | Claim with lease + fencing (02 §4.2) | `worker/publishing/claim.ts` | `claimTarget(db, orgId, targetId, workerId, leaseMs)` | **PUB-01** two workers | 0.5d |
| W9-4 | Publish processor (02 §4.3, fix D) | `worker/processors/publish.processor.ts` | `processPublish(job, deps)`; `classifyProviderError(err)`: maps provider errors to transient / definitive / unknown (a timeout is unknown) | **PUB-02, PUB-03, PUB-04, PUB-07** | 1.5d |
| W9-5 | Reconcile sweep + retry backoff (fix C1, C2) | `worker/processors/reconcile.processor.ts` | `reconcileStale(deps, now)`: handles expired leases and attempts with no outcome; it calls `connector.reconcile` and moves the target to published, retry or manual review | integration | 1d |

### Week 10: Status, calendar, analytics

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W10-1 | Publication status API + UI | `modules/publishing/status.*`, `features/campaigns/components/PublicationStatus.tsx` | `getStatus(ctx, targetId)`: returns the DB state with the external URL; it never reads the queue state | US-10 criteria | 1d |
| W10-2 | Calendar (cursor pagination, fix E2) | `modules/publishing/calendar.*`, `features/calendar/*` (lazy) | `listCalendar(ctx, businessId, range, cursor)` | integration | 1d |
| W10-3 | Metrics job + snapshots | `worker/processors/metrics.processor.ts`, `modules/analytics/*` | `collectMetrics(deps)`: calls `fetchMetrics` for recently published targets; `campaignReport(ctx, campaignId)`: compares results with the objective KPI | integration | 1.5d |
| W10-4 | Analytics screen (KPI cards, lazy charts) | `features/analytics/*` | none | Playwright; bundle budget | 1d |

### Week 11: Billing, limits, admin ops (→ M4)

| ID | Task | Files | Key functions | Tests | Est |
|----|------|-------|---------------|-------|-----|
| W11-1 | Plan limits as pure data | `packages/domain/src/plans.ts` | `limitsFor(plan)`: the D6 price table as code; `checkLimit(plan, usage, resource)`: pure | unit | 0.5d |
| W11-2 | Stripe checkout + webhooks | `modules/billing/*` | `createCheckout(ctx, plan)`; `handleStripeWebhook(rawBody, signature)`: verifies the signature, deduplicates on `webhook_events`, and re-fetches the current subscription from Stripe before updating, so the order events arrive in doesn't matter | **PAY-01, PAY-02** | 1.5d |
| W11-3 | Free-tier abuse + cost cap | `modules/usage/free-tier.ts` | `enforceFreeCap(ctx)`: blocks AI calls above the W0-3 cap; rate-limits signups by IP and email domain | integration | 0.5d |
| W11-4 | Admin ops page | `apps/admin/src/features/ops/*`, `modules/admin/*` | `listStuckPublications()`, `retryTarget(staff, id)` (fix C5), `resolveManualReview(staff, id, outcome)` (fix C6), `pauseAllPublishing(staff)`: a global kill switch that the worker checks | integration + staff-only | 1.5d |
| W11-5 | Billing + usage screens | `features/settings/billing/*` | none | Playwright | 0.5d |

### Week 12: Hardening and pilot (→ M5)

| ID | Task | Output | Est |
|----|------|--------|-----|
| W12-1 | Security pass: OWASP multi-tenant checklist, dependency audit, secrets scan | `docs/07-engineering/security-review.md`; zero high-severity issues left | 1d |
| W12-2 | Load + failure drills: worker kill, Redis restart, backup restore (OPS-02) | `docs/runbooks/*` (6 runbooks from PDF p.52) | 1d |
| W12-3 | Help center: getting started, connect accounts, first campaign, troubleshooting | `apps/help/content/*.mdx`; in-app Help links | 1d |
| W12-4 | Real-device mobile test (iOS Safari, Android Chrome), WCAG 2.2 AA check | test report | 0.5d |
| W12-5 | Pilot onboarding of 5 businesses; feedback form; metrics dashboard | pilot log | 1d |

---

## 4. Test plan: where each required test is built

| Test | Week | Test | Week |
|------|------|------|------|
| SEC-01 cross-tenant | W2-3 (+ every module) | PUB-01 double claim | W9-3 |
| SEC-02 role denial | W2-4 | PUB-02/03 crash before/after the provider call | W9-4 |
| SEC-03 expired OAuth | W8-4 | PUB-04 rate limit | W9-4 |
| SEC-04 prompt injection | W5-3 | PUB-05/06 cancel + race | W9-1 |
| APP-01 edit after approval | W6-1, W7-3 | PUB-07 IG container not ready | W9-4 |
| APP-02 concurrent approve | W7-3 | PAY-01/02 Stripe dupes + order | W11-2 |
| OPS-01 Redis restart | W9-2 | OPS-02 backup restore | W12-2 |

Coverage targets: `packages/domain` 100% of branches; services ≥ 80%; every endpoint has ≥ 1 cross-tenant test.

---

## 5. Risk register

| Risk | Likelihood | Impact | Mitigation | Trigger to act |
|------|-----------|--------|------------|----------------|
| Meta app review not approved by W12 | High | High | Start in W0; pilot with test/permitted accounts; manual-export fallback | No approval by W8 |
| Scope creep (chat, email, community) | High | High | Only Week 0 docs define scope; anything new goes to the post-beta backlog | Any unplanned task over 0.5d |
| AI invents facts | Medium | High | Fact-id validation, evals, mandatory approval | Eval failure rate > 0 |
| Solo developer falls behind | Medium | Medium | 0.5d buffer each week; drop W10-4 charts or W11-5 polish first | 2 weeks behind at M2 |
| AI cost above plan revenue | Medium | Medium | Reserve/settle credits, free cap, cheaper model for drafts | Cost/user > 30% of plan price |
| Duplicate publishing | Low | High | Claim + fencing + reconcile; never retry after an unknown outcome | Any PUB test flaky |

**What to cut first if late:** analytics charts (keep the numbers), the calendar view (keep the list), admin
UI polish (keep the endpoints). **Never cut:** tenant isolation, approval invalidation, or the
publish-once safeguards.

---

## 6. Tracking

- Use one GitHub Project board with the columns `Backlog → Ready → In progress → Review → Done`; one card per task ID.
- Every Friday: demo on staging, update the burndown, and check risk triggers (§5).
- At each milestone: a short review in `docs/08-release/changelog.md`; go/no-go for the next phase.

---

## 7. After the pilot (Phase 2, ordered)

1. Sales industry module: leads, pipeline, follow-up tasks (needs consent tables first, fix B12/G1).
2. Dropshipping module: Shopify GraphQL Admin API, read-only catalog → product campaigns.
3. Email sending through a delivery provider with consent, unsubscribe and suppression.
4. Research sources with provenance (fix G2) → market intelligence dashboard.
5. Professional workspace mode (advanced strategy controls).
6. Agency multi-client access (fix F2).
7. Team chat (WebSockets), then the community app (P2).
