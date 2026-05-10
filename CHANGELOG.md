# Changelog

## [0.3.0] — 2026-05-10

### Added

- **Multi-factor authentication** — `TotpSecret`, `BackupCode`, `BackupCodeBatch` schemas. `Challenge.strategy` enum extended with `totp` and `backup_code`. `Challenge.step` enum includes `"second"`. `ChallengeCreateRequest` second-factor variants.
- **FAPI MFA endpoints** — `POST /v1/me/totp` (start enrollment, returns one-time secret + otpauth URI + QR data URL), `POST /v1/me/totp/verify`, `GET /v1/me/totp`, `DELETE /v1/me/totp`. `POST /v1/me/backup-codes` (regenerate, returns plaintext once), `GET /v1/me/backup-codes` (status), `DELETE /v1/me/backup-codes`.
- **BAPI MFA admin overrides** — `POST /v1/users/{user_id}/verify-totp` (operator-driven check; does not stamp `verified_at`), `DELETE /v1/users/{user_id}/mfa` (operator nuke; clears TotpSecret + BackupCode + flips flags).
- **SignIn second-factor flow** — `needs_second_factor` status path documented; `SignIn.supported_strategies[]` narrows to per-user enrollment when applicable.
- **`InstanceSettings.multi_factor`** — required block: `totp.enabled` (default `true`), `backup_codes.enabled` (default `true`), `backup_codes.default_count` (default `10`, bounds 4..24). `Environment.auth_config.second_factors[]` mirrors per-env state for the public surface.

### Changed

- TOTP enrollment-verify is documented as a dedicated endpoint (`POST /v1/me/totp/verify`), distinct from the Challenge sub-resource (which is reserved for parent-bound flows: SignIn / SignUp / EmailAddress / OrganizationDomain).

## [Unreleased]

### Added

- **`phone_code` strategy + phone-MFA settings (v0.4)**.
  - `Challenge.strategy` documents `phone_code` for sign-up first-factor (env's `attributes.phone_number != "off"`), sign-in second-factor (`step: "second"` SMS MFA), and `PhoneNumber` verification on a signed-in user. `ChallengeCreateRequest.phone_number_id` is a documented field (was reserved); `ChallengeCreateRequest.strategy` lists `phone_code` and `oauth_<provider_key>`.
  - `InstanceSettings.multi_factor` gains a required `phone_code: { enabled }` block (default `false` — opt-in due to SMS cost + SIM-swap risk). `InstanceUpdateRequest.multi_factor` accepts the matching patch.
  - `Environment.auth_config.identifier_requirements.{email_address, phone_number, username}` enums shift from `[required, used_for_first_factor, off]` to `[required, optional, off]` for the v0.4 SDK vocabulary. `first_factors[]` / `second_factors[]` descriptions document the per-env narrowing rules.

- **Phone-number identifier surface (v0.4) — schemas**.
  - `PhoneNumber` (`phn_` prefix) — flat shape with `verified` + `current_challenge_id` (mirrors `EmailAddress` post-v0.2). Carries `is_primary`, `reserved_for_second_factor`, `default_second_factor`, and `linked_to_external_account_id` for SMS-MFA + external-account integrations.
  - `PhoneNumberCreateRequest` — `{ phone_number }` body. Server normalises to E.164.
  - `PhoneNumberUpdateRequest` — `{ is_primary?, reserved_for_second_factor?, default_second_factor? }` patch body. The phone-number value itself is immutable (re-add a new row to change).
  - `User.phone_numbers` resolves to a real `PhoneNumber` array (was a `[]`-only placeholder); `User.primary_phone_number_id` description tightened (no longer "always `null` in v0.1").

- **Social sign-in surface (v0.4) — schemas**.
  - `OauthProvider` (`oauthp_` prefix) — per-environment social-IdP configuration with three kinds in a single shape: `preset` (bundled `google` / `github` / `apple` / `microsoft`), `custom_oidc` (workspace-registered OIDC IdP, populated by `/.well-known/openid-configuration` discovery), `custom_oauth2` (plain OAuth 2.0 IdP, operator supplies authorize / token / userinfo URLs and `userinfo_method` + `userinfo_auth`). Computed read-only `redirect_uri = https://<env_slug>.authn.sh/v1/oauth-callback/<provider_key>`. Per-provider `attribute_mapping` (`User` / `ExternalAccount` field → IdP claim or userinfo path). Per-provider `block_email_subaddresses` and `allow_sign_in` / `allow_sign_up` toggles.
  - `OauthProviderRequest` — POST/PATCH body. `client_secret` is write-only (encrypted at rest, never returned). `provider_kind` and `provider_key` are immutable on PATCH.
  - `OauthProviderTestResult` — return shape of the BAPI dry-run probe.
  - `Challenge.strategy` documents `oauth_<provider_key>` as a first-factor value; the IdP authorize URL is surfaced via `external_verification_redirect_url`.

- **BAPI `/v1/oauth-providers` (v0.4)**.
  - `GET /v1/oauth-providers` (`listOauthProviders`), `POST /v1/oauth-providers` (`createOauthProvider`), `GET /v1/oauth-providers/{id}` (`getOauthProvider`), `PATCH /v1/oauth-providers/{id}` (`updateOauthProvider`), `DELETE /v1/oauth-providers/{id}` (`deleteOauthProvider`), `POST /v1/oauth-providers/{id}/test` (`testOauthProvider`). Soft-delete refuses (`409 oauth_provider_in_use`) when `ExternalAccount` rows still link to the provider. `test` returns `200` with `OauthProviderTestResult { authorize_url, userinfo_status, errors }` even when probes fail.

- **FAPI OAuth sign-in / sign-up flow (v0.4)**.
  - `ChallengeCreateRequest` documents the `oauth_<provider_key>` variant — required `redirect_url` (failure landing) and `redirect_url_complete` (success landing), both validated against the redirect-URL allowlist. Server stamps `step: "first"`; OAuth never serves as second factor.
  - New FAPI path `GET /v1/oauth-callback/{provider_key}` (`oauthCallback`). Hosted on the FAPI server but lives outside `/v1/client/...` — the IdP redirect is a top-level browser navigation. Always responds `302`; the SDK reads the result off the returning URL.
  - `SignIn` / `SignUp` describe the OAuth flow end-to-end: identifier-less Challenge issuance, `external_verification_redirect_url` carrying the IdP authorize URL, `transferable` cross-flow (sign-in for unknown identity → bounce to sign-up; sign-up for existing identity → bounce to sign-in). `supported_strategies[]` documents the per-provider narrowing: one `oauth_<provider_key>` entry per enabled `OauthProvider`, gated by `allow_sign_in` / `allow_sign_up`.

- **Multi-factor authentication surface (v0.3)**.
  - `TotpSecret` (one per `User`, `totp_` prefix) and `BackupCode` (`bcc_` prefix) resource schemas, plus `BackupCodeBatch` envelope for the one-time plaintext reveal.
  - `Challenge.strategy` enum gains `totp` and `backup_code`. `ChallengeCreateRequest` documents the second-factor variants (no extra fields needed beyond `strategy`); `ChallengeAnswerRequest` documents the `{ code }` answer shape for both.
  - FAPI: `POST /v1/me/totp` (start enrollment), `POST /v1/me/totp/verify` (confirm with first code), `DELETE /v1/me/totp` (remove). `POST /v1/me/backup-codes` ((re)generate batch), `DELETE /v1/me/backup-codes` (remove all unused).
  - BAPI: `POST /v1/users/{user_id}/verify-totp` (operator-driven TOTP check) and `DELETE /v1/users/{user_id}/mfa` (operator MFA reset).
  - `SignIn` documents the `needs_second_factor` step end-to-end — second-factor Challenge issuance with `step: "second"`, `supported_strategies` narrowing per the resolved user's enrolled methods (`totp` / `backup_code`), and the answer + transition rules.
  - `InstanceSettings.multi_factor` block — per-env `totp.enabled`, `backup_codes.enabled`, `backup_codes.default_count` (4..24, default 10). Matching `InstanceUpdateRequest.multi_factor` patch shape.

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
