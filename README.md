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

## Versioning

Pre-1.0 the spec changes freely. Once we tag `v0.1.0` in coordination with the rest of v0.1, breaking changes require a major bump and a migration note in this README.

## License

[AGPL-3.0-only](./LICENSE).
