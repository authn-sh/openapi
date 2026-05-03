# @authn.sh/openapi

The OpenAPI 3.1 specification for the [authn.sh](https://authn.sh) Backend API (BAPI) and Frontend API (FAPI). This is the **source of truth** for both surfaces — every authn.sh SDK, the REST reference docs, and the in-Dashboard API explorer all read from the bundled artifact published from this repo.

If you change request or response shapes anywhere in the platform, the change starts here.

## Layout

```
openapi.yaml                     # root document — info, servers, tags, security, refs into:
paths/                           # one file per resource group (added in OA-2 onward)
components/
  schemas/                       # split-out schemas (added in OA-2 onward)
  responses/                     # reusable response references
  parameters/                    # reusable parameter references
  securitySchemes/               # reusable security-scheme references
dist/                            # generated bundle output (gitignored)
.github/workflows/lint.yml       # CI
```

## Local development

Requirements: Node 20+.

```bash
npm install
npm run lint        # redocly lint — blocks on errors, warns on style issues
npm run bundle      # produces dist/openapi.bundled.{yaml,json}
npm run preview     # serves the rendered docs at http://127.0.0.1:8080
```

## Adding a new resource

1. Create the schema(s) under `components/schemas/<Resource>.yaml`.
2. Create the path file under `paths/v1_<resource>.yaml` (one file per top-level resource family).
3. Reference both from `openapi.yaml` (the bundler handles the stitching — keep `openapi.yaml`'s top-level `paths:` and `components:` sections updated with the `$ref`s).
4. Run `npm run lint` and `npm run preview` locally.
5. Open a PR. CI runs `lint` + `bundle` and uploads the bundle as an artifact for downstream SDK PRs to consume.

## How SDKs consume the bundle

SDK repos pull `dist/openapi.bundled.json` from this repo's CI artifacts (or from a tagged release for stable versions) and feed it into their codegen pipeline:

- `@authn.sh/sdk-js` / `@authn.sh/sdk-react` (`authn-sh/javascript`) — generates `@authn.sh/types` from the bundle.
- `authn/sdk` PHP (`authn-sh/sdk-php`) — contract tests validate fixtures against the bundled schemas.
- `authn-sh/cli` (Go) — `oapi-codegen` produces typed clients.

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
