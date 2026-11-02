# MarketPilot AI: Product requirements v1 (MVP)

Scope: the 12-week beta for **real-estate agents and agencies** (ADR 0002), Facebook + Instagram
only, human approval before anything is published. Stories are numbered as in the engineering
blueprint; acceptance criteria (AC) are what tests and reviews check.

## Personas

| Persona | Goal | Typical role |
|---------|------|-------------|
| Solo agent | Market listings without hiring help, from a phone | owner |
| Agency principal | Consistent marketing across agents, control over what is published | owner / admin |
| Agent in an agency | Create listing campaigns quickly | editor |
| Compliance / brand reviewer | Approve or reject before anything goes public | reviewer |
| MarketPilot staff | Support customers, watch system health | platform staff |

## Core flow

Goal → strategy & objective → campaign brief → AI drafts → edit → approve → schedule → publish → results.
A beginner completes it in **under 15 minutes** for the first campaign (PDF success metric).

## Stories and acceptance criteria

### Identity and team (done in M1)

- **US-01 Register and create an organization.** AC: signing in for the first time creates the
  user; the creator becomes owner; the slug is unique and URL-safe.
- **US-03 Invite a team member with limited permissions.** AC: invitation by email, single use,
  expires (7 days default), bound to the invited email; roles owner/admin/editor/reviewer;
  admins cannot create or change owners; the last owner cannot be demoted; users outside the
  organization get 404 for all its URLs.

### Business setup (Week 3)

- **US-02 Create and configure a business profile.** AC: name, industry (real estate), IANA
  timezone (validated); tone, audience and restrictions (words/claims never to use) are editable;
  only owner/admin can change them; changes are audited.
- **US-02a Maintain approved facts.** AC: facts have a kind (price, hours, address, offer,
  service, claim, other), a label, a value and an optional validity period; each fact records who
  verified it and when; the AI may use only active facts (Week 5).
- **US-02b Add a listing (real estate).** AC: address, price and currency, bedrooms, bathrooms,
  area, features and status (draft, active, under offer, sold, withdrawn); a listing's price and
  address are available as facts for its campaigns.
- **US-02c Guided onboarding on mobile.** AC: business type → starting goal → first suggested
  task (PDF p.77), at most three screens, usable at 360 px width, progress autosaved.

### Strategy and campaigns (Week 4)

- **US-05 Create a campaign from a business goal.** AC: a campaign links to a measurable
  objective (metric + target + date); dates validated (end ≥ start); repeated create requests with
  the same idempotency key create one campaign.

### Content (Weeks 5–6)

- **US-06 Generate platform-specific content.** AC: uses only the business profile, active facts
  and the brief; separate drafts per platform; flags missing facts; never invents prices,
  discounts, guarantees or opening hours; real-estate copy passes the fair-housing check; usage
  is metered and limited by plan; nothing is published automatically.
- **US-07 Edit content and upload media.** AC: every edit creates a new immutable revision;
  uploads are type- and size-checked; an edit to approved content invalidates the approval.

### Approval and publishing (Weeks 7–9)

- **US-08 Approve or reject a specific revision.** AC: approval names the exact revision and
  destination; concurrent reviewers cannot produce two effective outcomes; every decision is audited.
- **US-09 Schedule an approved post.** AC: only approved, unchanged content; future time in the
  business timezone, sent as UTC; duplicate requests do not create duplicate posts; a
  disconnected account makes the post ineligible.
- **US-10 See whether a post succeeded.** AC: shows scheduled, publishing, published, failed or
  uncertain; links to the published post; never shows success based only on a finished job.
- **US-11 Cancel a pending publication.** AC: allowed until publishing starts; the answer is
  honest if publishing already began.
- **US-14 Inspect and recover failed jobs (staff).** AC: stuck and uncertain publications are
  listed; staff can retry or resolve them; a global "pause publishing" switch exists.

### Results and billing (Weeks 10–11)

- **US-12 Review basic campaign metrics.** AC: results are compared with the objective's target;
  estimates are labelled as estimates.
- **US-13 Subscribe to a plan and see usage.** AC: plans and limits as in ADR 0006; usage shown
  per month; limits enforced server-side; the free plan has a hard AI cost cap.
- **US-15 Disconnect accounts and delete eligible data.** AC: disconnecting revokes stored
  tokens; deletion requests are audited.

## Real-estate specifics

- **Listing campaign templates:** new listing, open house, price change, just sold, buyer/seller lead magnet.
- **Fair housing:** copy and targeting must not express preference or limitation based on protected
  characteristics (race, colour, religion, sex, disability, familial status, national origin, and
  local additions). The checker flags phrases such as "perfect for young couples", "no kids",
  "Christian neighbourhood", "exclusive community" and requires human review. It assists the
  reviewer; it is not legal advice.
- **Accuracy:** price, address, room counts and area come from the listing record, never from the AI.

## Non-functional requirements

- Mobile-first: works from 320 px; touch targets ≥ 44 px; WCAG 2.2 AA target.
- Every API error uses the standard envelope with a request id.
- Tenant isolation is tested for every endpoint (SEC-01).
- No secrets or tokens in logs (enforced by the logger).

## Out of scope for the MVP

Paid ads, AI video, social inbox, CRM sync, email campaigns, public community, agency multi-client
access, sales and dropshipping modules (see `planning/03-development-plan.md` §7).
