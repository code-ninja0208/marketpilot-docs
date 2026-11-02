# MarketPilot AI — Review of `Example.pdf`: Missing and Fix Points

Source: `Example.pdf` (86 pages, ChatGPT conversation "Fast Growing New Services", 2 Nov 2026).
Status of the project: **planning only — no code exists yet.** Both checklists in the PDF are at
0/14 (implementation) and 0/9 (documentation).

The plan was revised four times inside the conversation (v1 campaign tool → v1 engineering
blueprint → v2 "Marketing OS" → v3 professional-marketer review → v4 real-estate / sales /
dropshipping + docs + mobile). Later revisions changed earlier decisions without updating them,
so the document currently contradicts itself in several places. Severity: **High** = blocks a
correct build, **Medium** = causes rework, **Low** = clarity.

---

## A. Contradictions between plan versions

| # | Sev | Topic | Problem (PDF page) | Fix |
|---|-----|-------|--------------------|-----|
| A1 | High | Pricing | Three different price tables: $29/$79/$199 (p.10), $59/$129/$299 (p.18), Free/$19/$59/$149 (p.61). | Adopt the latest (Free/$19/$59/$149) as the only source of truth in `docs/00-project/pricing.md`. |
| A2 | High | Unit economics | The margin model (81%, $6,435 MRR, p.18–19) uses the **old** $59/$129/$299 prices and has **no free tier**. Free-plan AI cost is not modelled at all. | Recalculate with the new prices; add free-user cost per month and the free→paid conversion assumption; set a hard monthly AI-cost ceiling per free account. |
| A3 | High | Repository tree | Three different trees: `apps/web` (p.27) → `apps/client, admin, website, community` (p.70) → `client, admin, website, api, worker` (p.80). The `help` app is P0 (p.83) but missing from every tree. Packages `queues`, `config`, `social-connectors` disappear in v3 and `auth`, `marketing-core`, `messaging`, `industry` appear. | Use one canonical tree (see `02-build-plan.md` §2). |
| A4 | High | 12-week roadmap | Three different 12-week plans (p.22, p.44, p.73, p.84). Scope grew from "campaign → publish" to "+ docs, design system, industry onboarding, strategy assistant, research, Shopify, help center, admin" but the timeline stayed 12 weeks. | Keep 12 weeks only for the **core loop** (strategy → content → approval → publish → report) for **one** industry first; move the rest to phases after beta (see `02-build-plan.md` §6). |
| A5 | Medium | Target market | "All small businesses" (p.11) vs "focus real estate, sales, dropshipping" (p.75). | Decide the launch segment. Recommendation: launch with one segment (e.g., real estate), keep the other two as templates. |
| A6 | Medium | Frontend app name | `apps/web` and `apps/client` are used for the same app. | Use `apps/client`. |

## B. Database schema gaps (p.30–31, p.71)

| # | Sev | Problem | Fix |
|---|-----|---------|-----|
| B1 | High | Unique index `uq_post_placement` uses `placement_key`, but `post_targets` has no `placement_key` column. | Add `placement_key text not null`. |
| B2 | High | The publish worker needs a **claim / lease / fencing token** (p.39–40), but `post_targets` has no lease columns. | Add `lease_owner`, `lease_expires_at`, `fencing_token bigint`. |
| B3 | High | The API requires an `Idempotency-Key` header (p.33, p.36), but no table stores keys or responses. | Add `idempotency_keys (org_id, key, request_hash, response_json, status, expires_at)`, unique on `(org_id, key)`. |
| B4 | High | `GET /v1/generations/:id` exists, but no `ai_generations` table does. | Add `ai_generations (id, org_id, campaign_id, status, input_json, output_json, reserved_credits, used_credits, error_code)`. |
| B5 | High | `POST /v1/organizations/:id/invitations` exists, but no `invitations` table does. | Add `invitations (id, org_id, email, role, token_hash, expires_at, accepted_at)`. |
| B6 | High | Integration tests mention "OAuth state storage" (p.45), but no table stores it. | Add `oauth_states (state_hash, org_id, business_id, provider, pkce_verifier_ciphertext, expires_at)`. |
| B7 | Medium | `outbox_events`, `approvals`, `publish_attempts` have no `org_id`, so tenant-scoped queries need joins and are easy to get wrong. | Add `org_id` to every tenant-owned table (use RLS or composite FKs). |
| B8 | Medium | `media_assets` appears in the ERD but has no columns. | Define `(id, org_id, business_id, storage_key, mime_type, size_bytes, width, height, checksum, status)`. |
| B9 | Medium | v3 adds 13 marketing entities (MarketingStrategy, AudienceSegment, ResearchSource, …, p.71) with purpose only — no columns, keys or relations. | Specify the columns before Week 3. |
| B10 | Medium | v4 industry objects (listings, leads, pipelines, products, stores) have no schema. | Add them per industry module once the launch segment is chosen. |
| B11 | Medium | Team comments / conversations (v2–v4) have no tables. | Add `comments (id, org_id, resource_type, resource_id, author_id, body, created_at)` for MVP; postpone chat. |
| B12 | Medium | Email marketing (Phase 2) needs consent, unsubscribe and suppression (p.60), but the schema has no tables for them. | Add `contacts`, `consents`, `suppressions` before any email sending. |
| B13 | Low | Quota "reservation" (p.35) has no reservation record. | Add `usage_reservations`, or use reserve/settle rows in `usage_events`. |

## C. Publishing state machine gaps (p.37–38)

The diagram shows states that the transition table never reaches or leaves:

| # | Sev | Missing transition | Fix |
|---|-----|--------------------|-----|
| C1 | High | `Retry waiting → Queued` (backoff elapsed) is not defined. | Add it, with `attempt_no < maxAttempts`, otherwise `→ Failed`. |
| C2 | High | `Reconciliation required → Published / Failed / Manual review` is not in the table. | Add three transitions: confirmed published / confirmed not published / unresolved. |
| C3 | High | "Edit content" invalidates approval (p.32), but `Approved → Draft` and `Scheduled → Draft` are not in the table. | Add `edit` from `pending_approval`, `approved` and `scheduled` → `draft` (approval invalidated). |
| C4 | Medium | Cancel is only allowed from `Scheduled`, but US-11 says "cancel a pending publication" and `Queued` is also cancelable before the lease. | Allow `cancel` from `draft`, `pending_approval`, `approved`, `scheduled`, `queued`, and from `retry_waiting`; reject it from `publishing`. |
| C5 | Medium | `Failed` is described as needing a "controlled retry" (p.38), but no transition or endpoint exists. | Add an admin `retry` (`failed → queued`) that creates a new attempt; also `POST /v1/admin/post-targets/:id/retry` (US-14). |
| C6 | Low | `Manual review` has no exit. | Add admin `resolve_published` / `resolve_not_published`. |

## D. Publish worker pseudocode bug (p.39–40)

The pseudocode wraps **everything** in one `try/catch`:

- If `connector.publish()` **succeeds** and then `confirmPublished()` **throws** (DB blip), the catch
  calls `classifyAndRecordFailure`. That can schedule a retry, and the post gets **published twice**.
  The text below the code warns about this, but the code does not handle it.
- An eligibility failure (`assertStillEligible`) goes through the same failure classifier as a
  provider error, so an ineligible post can enter the retry path.
- The code never writes a `publish_attempts` row **before** the external call, which the
  "required safeguards" section asks for (p.39).

The corrected version, with separate error scopes, is in `02-build-plan.md` §4.3.

## E. API contract gaps (p.32–36)

| # | Sev | Problem | Fix |
|---|-----|---------|-----|
| E1 | High | v3/v4 features have no endpoints: strategy, objectives, audiences, research sources, comments, help, admin console, invitation accept, Shopify. | Add contracts per module before building each one. |
| E2 | Medium | No pagination / filtering / sorting convention for list endpoints. | Use a standard cursor: `?cursor=&limit=` → `{ data, nextCursor }`. |
| E3 | Medium | Auth is shown as `Authorization: Bearer <session-token>`, but Next.js apps usually use httpOnly cookies. The auth provider is not chosen. | Choose the provider (e.g., Clerk / Auth0 / Supabase Auth); use cookie sessions in the browser and Bearer tokens only for server-to-server calls. |
| E4 | Medium | The admin console needs a separate authorization boundary (p.69), but no `/admin/v1` API surface or platform-staff role exists. | Add a `platform_staff` table, `/admin/v1/*` routes, enforced MFA. |
| E5 | Low | The `GeneratedContent.platform` type allows only `"facebook" \| "instagram"`; email/blog/listing content (v2+) does not fit. | Use a discriminated union per content type. |

## F. Roles and permissions

| # | Sev | Problem | Fix |
|---|-----|---------|-----|
| F1 | Medium | US-09 uses a "publisher" role, but only owner/admin/editor/reviewer exist. | Rename US-09 to "editor", or add a `publisher` permission. |
| F2 | Medium | Agency multi-client access (p.69) is described but not modelled. | Add `agency_grants (agency_org_id, client_org_id, role, granted_by, revoked_at)`. |
| F3 | Low | Self-approval policy is "later" but affects the approval logic now. | Add an `org.require_separate_reviewer` boolean (default false) from day one. |

## G. Compliance and safety gaps

| # | Sev | Problem | Fix |
|---|-----|---------|-----|
| G1 | High | Sales outreach (v4) touches cold email/SMS rules (CAN-SPAM, TCPA, GDPR/ePrivacy); only fair housing and product claims are covered. | Add a compliance section per industry and block sending without a consent record. |
| G2 | Medium | Competitor research data sources are not defined. Scraping and platform ToS limits are unaddressed. | Limit research to user-provided URLs + licensed APIs; store `ResearchSource` with retrieval date. |
| G3 | Medium | No AI evaluation plan (accuracy of facts, refusal of invented claims, prompt-injection tests beyond SEC-04). | Add a golden-set eval suite run in CI (`packages/ai/evals`). |
| G4 | Low | PWA push notifications on iOS only work for installed web apps. | Note it in `known-limitations.md`; provide an email notification fallback. |

## H. Missing decisions (must be made before coding)

1. Launch industry (one of: real estate / sales / dropshipping).
2. Auth provider.
3. AI model provider(s) and the per-plan credit → token mapping.
4. Hosting for the always-on worker (Fly.io / Railway / ECS / VM).
5. Meta developer app owner and business verification (external blocker, start in Week 1).
6. Final price table (A1) and the free-tier cost cap (A2).
