# Meta (Facebook and Instagram) integration

How MarketPilot connects a business's Facebook Page and Instagram professional account, what the
owner has to set up in Meta, and what is still open. Code: `marketpilot-shared/packages/connectors`,
`marketpilot-api/src/modules/integrations`, `marketpilot-worker/src/processors/account-check.processor.ts`.

## What the owner sets up (W0-6)

1. Create an app at developers.facebook.com (type **Business**).
2. Add the product **Facebook Login for Business**. Under its settings add the valid OAuth redirect
   URI `<PUBLIC_URL>/api/v1/integrations/meta/callback` (staging: the VPS address or domain).
3. Request these permissions (all are needed to publish):
   `pages_show_list`, `pages_read_engagement`, `pages_manage_posts`, `instagram_basic`,
   `instagram_content_publish`.
4. While the app is in **development mode** only people with a role on the app (admin, developer,
   tester) can connect. Add the pilot team as testers, plus a test Facebook Page with an Instagram
   professional account linked to it.
5. Before real customers connect, submit the permissions for **App Review** and complete Business
   Verification. Meta asks for a screencast of the connect and publish flow.
6. Put the app id and secret in the server's `.env` (`META_APP_ID`, `META_APP_SECRET`), never in git.

## How connecting works

1. *Connect Facebook & Instagram* (owners and admins) creates a single-use handshake: the SHA-256 of
   a random `state` and the encrypted PKCE verifier, valid 10 minutes (review item B6).
2. Facebook Login sends the browser back to the callback. The handshake must belong to the
   signed-in user and is used exactly once; a replayed or late callback is refused.
3. The code is exchanged for a long-lived user token (about 60 days) and the user's Pages with
   their Page tokens. Page tokens obtained this way do not expire while the user keeps their role
   on the Page. Pages are kept encrypted for 15 minutes until the user chooses one.
4. The chosen Page and its linked Instagram account are stored with the Page token encrypted
   (envelope encryption, `TOKEN_ENCRYPTION_KEYS`). One account per platform per business; choosing
   another Page replaces the previous one. The other Pages and their tokens are deleted.
5. Every hour the worker verifies tokens not checked in the last day. A refused, expired or
   unreadable token marks the account `reconnect_required`; posts to it pause until the owner
   reconnects (SEC-03).

## Publishing facts the connectors rely on

- Graph API host `graph.facebook.com`, version from `META_GRAPH_VERSION` (default `v26.0`).
- Facebook: text to `/{page}/feed`, one photo to `/{page}/photos` (`url`, `caption`), several photos
  uploaded unpublished then attached to one feed post. Page photos may be at most 4 MB.
- Instagram: container `POST /{ig}/media` (`image_url`, `caption`; carousels from child containers),
  wait for `status_code` FINISHED, then `POST /{ig}/media_publish`. JPEG only, up to 8 MB, at least
  320 px wide, aspect ratio between 4:5 and 1.91:1. 100 API posts per 24 hours per account.
- Meta downloads images from their URLs, so signed storage URLs must be reachable from the internet
  (staging exempts `/marketpilot-media/*` from the basic-auth login).
- Errors are classified as transient (rate limits, outages, media processing), definitive, unknown
  (a publishing request that failed or timed out — reconciled, never resent) or auth (reconnect).

## Open points

- PKCE: Meta documents PKCE for its OIDC code flow and marks it as in testing. The authorize URL
  sends a challenge and the exchange sends the verifier; the single-use hashed state and the app
  secret protect the flow even if Meta ignores PKCE. Confirm with the real app.
- The connectors' contract tests use answers shaped like Meta's reference documentation, not
  recordings of live calls. Record real answers with the test Page once the app exists.
- Metrics (`fetchMetrics`) arrive with analytics in Week 10.
