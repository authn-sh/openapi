# @authn.sh/openapi

The OpenAPI 3.1 specification for the [authn.sh](https://authn.sh) Backend API (BAPI) and Frontend API (FAPI). This is the **source of truth** for both surfaces — every authn.sh SDK, the REST reference docs, and the in-Dashboard API explorer all read from the bundled artifacts published from this repo.

If you change request or response shapes anywhere in the platform, the change starts here.

## Two specs, one repo

The BAPI (server-to-server, `bearer_secret_key`) and FAPI (browser-facing, `__client` cookie) overlap on URL prefixes (e.g. `POST /v1/organizations` exists on both with different request/response shapes). OAS3 only allows one operation per (path, method) pair, so the two surfaces have separate root specs that share a single `components/` directory.

```
bapi.yaml                          # BAPI root (server-to-server)
fapi.yaml                          # FAPI root (browser-facing)
paths/
  bapi/                            # BAPI path files
  fapi/                            # FAPI path files
components/
  schemas/                         # shared resource schemas
  responses/                       # shared response references
  parameters/                      # shared parameter references
dist/                              # generated bundles (gitignored)
  bapi.bundled.{yaml,json}
  fapi.bundled.{yaml,json}
.github/workflows/lint.yml         # CI runs both
```

The two roots are deliberately **separate documents** — operation IDs (`getOrganization`, `createOrganization`, …) are only unique within each spec, not across both. SDK consumers pin to whichever bundle their surface targets.

## Local development

Requirements: Node 20+.

```bash
npm install
npm run lint           # lint:bapi + lint:fapi
npm run lint:bapi      # one surface at a time
npm run lint:fapi
npm run bundle         # bundle:bapi + bundle:fapi
npm run preview:bapi   # serves BAPI docs at http://127.0.0.1:8080
npm run preview:fapi   # serves FAPI docs
```

## Adding a new resource

1. Create the schema(s) under `components/schemas/<Resource>.yaml` — schemas are shared between BAPI and FAPI, so a new resource only needs them written once.
2. Create the path file under `paths/bapi/<resource>.yaml` and/or `paths/fapi/<resource>.yaml` depending on which surface(s) expose it.
3. Reference each path file from the corresponding root (`bapi.yaml` and/or `fapi.yaml`).
4. Add the schema to the relevant root's `components.schemas` map (each root only registers what its paths reference).
5. Run `npm run lint` and `npm run preview:<surface>` locally.
6. Open a PR. CI runs both lints + bundles and uploads `dist/` as an artifact for downstream SDK PRs to consume.

## How SDKs consume the bundles

SDK repos pull `dist/bapi.bundled.json` and/or `dist/fapi.bundled.json` from this repo's CI artifacts (or from a tagged release for stable versions) and feed them into their codegen pipeline:

- `@authn-sh/sdk-js` / `@authn-sh/sdk-react` (`authn-sh/javascript`) — consumes **`fapi.bundled.json`** (browser-facing).
- `authn-sh/sdk-php` — consumes **`bapi.bundled.json`** (server-to-server).
- `authn-sh/cli` (Go) — admin tooling, consumes **`bapi.bundled.json`**.

Pin the spec version in each SDK's `package.json` / `composer.json` / `go.mod` to the openapi tag you want to track.

## Webhook signature contract

Every delivery `POST`s a JSON `WebhookEvent` body and three headers:

| Header           | Meaning                                                                 |
|------------------|-------------------------------------------------------------------------|
| `svix-id`        | Stable message ID (also `WebhookDelivery.id`).                          |
| `svix-timestamp` | Unix-second epoch when authn.sh signed the body.                        |
| `svix-signature` | Space-separated list of `v1,<base64>` entries (multi-value on rotation).|

The signature is `HMAC-SHA256` of `"{svix-id}.{svix-timestamp}.{raw_request_body}"` keyed by the **decoded** signing secret (the `whsec_<base64>` prefix is stripped and the suffix base64-decoded into the raw key bytes). Verifiers must:

1. Reject the request if any of the three headers is missing or empty.
2. Reject the request if `|now − svix-timestamp| > 300` seconds (replay window).
3. Iterate every `v1,<base64>` entry in `svix-signature` and constant-time compare its base64 value against the locally-computed signature; accept on first match. (Multi-value headers occur during a `rotateWebhookEndpointSecret` overlap window.)

### Worked example

```text
secret             = whsec_dGVzdC1zZWNyZXQtZG8tbm90LXJldXNlLXRoaXM=
decoded            = "test-secret-do-not-reuse-this"

svix-id            = whd_01HKX9SY9V7H7TF8C8K7J9X4ZK
svix-timestamp     = 1714896500
raw_body           = {"id":"whd_01HKX9SY9V7H7TF8C8K7J9X4ZK","object":"event","type":"user.created","timestamp":1714896500000,"instance_id":"env_acme_test","data":{"id":"user_01HKX9SY9V7H7TF8C8K7J9X4ZB","object":"user"}}

signed_payload     = "whd_01HKX9SY9V7H7TF8C8K7J9X4ZK.1714896500.{raw_body}"
signature_bytes    = HMAC-SHA256(decoded, signed_payload)
svix-signature     = "v1," + base64(signature_bytes)
                   = "v1,JG2qiInfP2EnVN6nq+S6XKhSGq9D+TlMkRkn2g4tBgI="
```

Reference verifier implementations: PHP — [`Authn\Sdk\Webhooks\SignatureVerifier`](https://github.com/authn-sh/sdk-php/blob/main/src/Webhooks/SignatureVerifier.php); JavaScript — `@authn.sh/sdk-js` (post-v0.1).

## Versioning

Pre-1.0 the spec changes freely. Once we tag `v0.1.0` in coordination with the rest of v0.1, breaking changes require a major bump and a migration note in this README.

## License

[AGPL-3.0-only](./LICENSE).
