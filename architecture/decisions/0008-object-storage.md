# ADR 0008: S3-compatible object storage, SeaweedFS on staging

- Status: accepted
- Date: 2026-11-13

## Decision
Uploaded images go to S3-compatible object storage through one interface,
`@marketpilot/storage` (`ObjectStore`). Browsers upload and download directly with short-lived
signed URLs; the API never relays file bytes except to verify each upload once (size, SHA-256,
real type from the bytes, dimensions) before marking it `ready` (review item B8).

- **Staging:** SeaweedFS 4.51 in the Compose stack, served through Caddy at
  `/marketpilot-media/*` on the app's own address. Same origin means no CORS setup, and the
  path-style URL reaches storage unchanged, so signatures stay valid.
- **Production:** any S3 service (AWS S3, Cloudflare R2, Backblaze B2), chosen with production
  hosting in Week 11. Only environment variables change (`S3_ENDPOINT`, `S3_PUBLIC_ENDPOINT`,
  `S3_BUCKET`, keys).
- **Tests:** an in-memory store with the same interface.

## Why not MinIO
The development plan assumed MinIO. Its community edition stopped publishing Docker images in
October 2025 and the repository was archived in 2026, so it no longer gets security fixes.
SeaweedFS is Apache-2.0, actively released and runs as one container. Garage (AGPL, needs
cluster layout commands at setup) and RustFS (young) were the other candidates.

## Consequences
- Object keys start with `orgs/<orgId>/businesses/<businessId>/`, so one tenant's files can be
  listed or deleted by prefix.
- Signed upload URLs carry no body checksum: the AWS SDK's default adds one and breaks browser
  uploads, so the API verifies the SHA-256 itself on confirm.
- The `seaweedfs-data` volume must be backed up with the database.
- Without `S3_BUCKET` the API runs with uploads switched off (media routes answer 503).
