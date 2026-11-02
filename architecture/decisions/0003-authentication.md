# ADR 0003: Provider-neutral authentication with JWT + JWKS

- Status: accepted (D2)
- Date: 2026-11-02

## Decision
Sign-in is delegated to a managed identity provider (default candidate: Clerk; Auth0 or
Supabase Auth also fit). The API **does not depend on a provider SDK**: it verifies the
provider's signed JWT (RS256/ES256) against the provider's JWKS URL, issuer and audience, then
maps the token subject to `users.auth_subject`.

Organizations, memberships and roles are stored in **our** database, not in the provider, so
authorization stays under our control and tenant tests run without the provider.

## Consequences
- Configuration: `AUTH_JWKS_URL`, `AUTH_ISSUER`, `AUTH_AUDIENCE`.
- Tests and local development use a local key pair and JWKS (no provider account needed).
- Switching provider is a configuration change plus a sign-in UI change in the client app.
