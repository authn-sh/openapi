# Changelog

## [0.2.0] — 2026-05-10

### Added

- **Organization domain** — `Organization`, `OrganizationMembership`, `OrganizationInvitation`, `OrganizationDomain`, `OrganizationMembershipRequest`, `Role`, `Permission` schemas. BAPI + FAPI endpoints for CRUD, memberships, invitations, domains, roles/permissions, and active-organization switching. Default seeded roles `org:admin` / `org:member` and the 13 system permission keys.
- **Email magic-link strategy** (`email_link`) for sign-in + sign-up, including the `transferable` cross-flow handling and cross-device polling story.
- **`Challenge` sub-resource** — unified verification dance hangs off `SignIn`, `SignUp`, `EmailAddress`, and `OrganizationDomain`. Replaces the per-factor `prepare-*` / `attempt-*` endpoint pairs and the standalone `/me/email-addresses/{id}/{prepare,attempt}-verification` and `/organizations/{org}/domains/{did}/verify` endpoints.
- **`domain_dns_txt` strategy** — DNS TXT verification for organization domains now goes through the standard challenge dance, which gives clients a real polling story (the old `/verify` action had no clean way to learn when the DNS lookup completed).
- **Webhook event catalogue** for `organization.*`, `organizationMembership.*`, `organizationInvitation.*`, `organizationDomain.*`, `role.*`, `permission.*`, and the `organization` block on `session.created` / `session.touched`.
- **Auto-applied contract validation** consumers (e.g. authn-sh/authn) now run every test response through the bundled spec; spec drift fails the offending test loudly with operation + status + JSON-pointer.

### Changed

- **URL paths use kebab-case** for every BAPI + FAPI segment. Path parameter names (`{user_id}`, `{organization_id}`, etc.), query parameters, and JSON request/response field names stay snake_case.
- `SignIn` / `SignUp` resources drop `factor_one_verification`, `factor_two_verification`, `supported_first_factors[]`, `supported_second_factors[]`, and the per-attribute `verifications` map. They now expose `current_challenge_id` and `supported_strategies[]`. Server narrows `supported_strategies[]` based on the parent's current state — clients never name a factor in the URL.
- `EmailAddress` / `OrganizationDomain` drop their nested `verification` blob. Both expose flat `verified: bool` + `current_challenge_id: ?Id`. The standalone `Verification` schema is removed — `Challenge` is the only verification-state shape.
- `ClientResponseEnvelope.client` is now `Client | null` so `DELETE /v1/client` can sign the device out cleanly.
- `Environment` exposes `object`, `id`, `paths`, `sessions`, `created_at`, `updated_at`, plus `display_config.brand_color` (the SDK already needed these — they were de-facto contract).
- FAPI organization create + membership create + invitation create now declare `201` (the controllers already shipped 201; the spec was the stale party).
- `OrganizationInvitation.url` is nullable (revoked / accepted / expired invitations don't carry a redeemable ticket URL).
- `WebhookEndpoint` exposes `rotation_window_expires_at` and `disabled_at`. `signing_secret` is write-only (only present on create / rotate).
- System permission catalogue corrected from 12 to 13 keys (added `org:sys_profile:read`).

### Removed

- All `prepare-*` / `attempt-*` factor endpoints on SignIn / SignUp, the `reset-password` action, `prepare-verification` / `attempt-verification` on `/me/email-addresses/{id}`, and the `/verify` action on `/organizations/{org}/domains/{did}` — replaced by the unified `Challenge` sub-resource.
- `Verification` schema (no longer surfaced — Challenge wraps it).

## [0.1.0] — 2026-05-03

Initial public spec. BAPI + FAPI for users, sessions, email addresses, sign-in / sign-up flows, JWT tokens, JWKS, webhook endpoints + deliveries, allowlist / blocklist identifiers, redirect URLs, instance settings.
