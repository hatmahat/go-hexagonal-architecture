# Adapters Catalog

Outbound capabilities are grouped into **axes** (persistence, cache, auth, …). Each axis has a port; each **option** is one concrete adapter at `internal/adapter/<axis>/<option>/` that implements it. This document explains when to choose each option.

Only ship options you will actually run and test. A listed option is a candidate, not an obligation — an adapter that only compiles gives false assurance (see `CLAUDE.md` Rule 4).

## Conventions

- Adapters implement a port declared by the domain, or by the axis root package for standalone capabilities (storage, messaging, notification, …).
- An adapter never imports another adapter. Shared types live in the domain or the axis root package.
- A `noop` option is the explicit "off" choice: it satisfies the interface so service code does not change.
- Selection happens in the composition root (`internal/app/container.go`), usually by an env var such as `STORAGE_BACKEND`. An empty or unrecognised value must fail at startup with an error listing the valid options.

## Wiring mode

| Mode | Meaning | Typical axes |
|---|---|---|
| **wired** | Instantiated in `container.go`; works out of the box. | persistence, cache, auth, observability, storage |
| **standalone** | Ships as a reference implementation; wire it manually into the service that needs it. | messaging, notification, jobs, cloud secret sources |
| **meta** | Affects build/tooling only; no Go runtime wiring. | api-doc, container |

## persistence

Required by any domain that declares a `Repository` port. Migrations live at `db/migrations/<dialect>/` (golang-migrate format, forward-only).

| Option | When to choose |
|---|---|
| `postgres` | Default. Strong consistency, mature ecosystem, JSON support, full SQL feature set. |
| `mysql` | Compatibility with existing MySQL infra (e.g., Aurora MySQL, PlanetScale). |
| `mongodb` | Flexible schema, document-oriented data, log-like workloads. Combine with a relational option if you need both. |

### Seeding

Each dialect keeps seed files under `db/seeds/<dialect>/`:

- **Postgres / MySQL** — `NNN_<name>.sql`, executed in lexicographic order. Idempotency is per file (`ON CONFLICT DO NOTHING` / `INSERT IGNORE`).
- **MongoDB** — `NNN_<collection>.json` arrays of documents, upserted with `$setOnInsert` so re-runs do not overwrite existing rows.

Run: `make seed-<dialect>` (wraps `go run . db:seed`; routes to the dialect in config).

## cache

Optional capability used by services for read-through caching.

| Option | When to choose |
|---|---|
| `redis` | Default. Shared across pods, persistent if needed. |
| `inmemory` | Single-instance services or tests; no cross-pod coherence. |
| `noop` | No caching. |

## auth

Authentication middleware applied to `/v1/*` routes.

| Option | When to choose |
|---|---|
| `jwt` | Default. HS256 self-issued tokens or symmetric keys. |
| `apikey` | Service-to-service or simple API-key authentication. |
| `oauth2-oidc` | OIDC-compliant IdP (Keycloak, Auth0, Cognito, Okta, Google, …). Validates via JWKS. |
| `none` | Auth happens at the edge (API gateway), or the service is internal-only. |

Every option must provide an echo middleware factory and satisfy the domain `auth` port, even when stubbed.

## observability

Tracing + structured-logging glue.

| Option | When to choose |
|---|---|
| `otel` | Default. Vendor-neutral; export to any OTLP backend (Tempo, Honeycomb, …). |
| `datadog` | Datadog APM. |
| `noop` | Local development without an APM target. |

## messaging

Publishers for outbound events. Wire into the service constructor when you need to emit events.

| Option | When to choose |
|---|---|
| `gcp-pubsub` | GCP-hosted services. |
| `rabbitmq` | Self-hosted broker; topic exchanges, routing keys. |
| `redis-streams` | Lightweight broker; reuses Redis infra. |
| `aws-sqs` | Queue (point-to-point). |
| `aws-sns` | Pub/sub (fan-out). Common pair with SQS. |
| `noop` | No outbound events. |

## storage

Object storage for uploads, exports, attachments. Selected at startup via `STORAGE_BACKEND`, which allows per-environment switching.

| Option | When to choose |
|---|---|
| `gcs` | GCP. |
| `s3` | AWS or any S3-compatible service. |
| `minio` | Self-hosted S3-compatible. |
| `local` | Local filesystem (dev or single-instance). |
| `noop` | No file storage. |

Example per-environment setup:

| Env file | `STORAGE_BACKEND` | Why |
|---|---|---|
| `env.local` | `minio` | Free, self-hosted, no cloud credentials |
| `env.staging` | `gcs` | Match production infra |
| `env.production` | `gcs` | Production object store |

## secrets

Secret-source adapters. `env` is always available (implicit via `config.Load`); cloud sources are additive and wired manually.

| Option | When to choose |
|---|---|
| `env` | Default; reads the process environment. |
| `gcp` | Google Secret Manager. |
| `aws` | AWS Secrets Manager. |
| `vault` | HashiCorp Vault. |

## notification

Outbound email.

| Option | When to choose |
|---|---|
| `smtp` | Generic SMTP (any provider). |
| `sendgrid` | Twilio SendGrid. |
| `noop` | No email. |

## jobs

In-process background work.

| Option | When to choose |
|---|---|
| `cron` | Scheduled tasks via `robfig/cron/v3`. |
| `worker` | Channel-driven worker pool. Pair with a `messaging` option for queue consumers. |
| `noop` | No background jobs. |

## rpc

gRPC transport alongside the always-present Echo HTTP server.

| Option | When to choose |
|---|---|
| `none` | Default. HTTP-only service. |
| `grpc` | Adds a gRPC server on `GRPC_HOST:GRPC_PORT`. |

Key paths: `proto/` (sources), `gen/` (committed stubs, so `go build` needs no `buf`), `internal/adapter/rpc/grpc/` (server bootstrap, client factory, health service). Regenerate with `make buf-generate`.

## api-doc (meta)

| Option | When to choose |
|---|---|
| `swag` | Code-first via inline comments on handlers; generates `swagger.json`. |
| `oapi-codegen` | Spec-first; types and Echo server stubs generated from `openapi.yaml`. |
| `none` | No API docs, or maintained externally. |

## container (meta)

Local dev and packaging files.

| Option | When to choose |
|---|---|
| `dockerfile` | Always. Production image build. |
| `compose` | Local dev orchestration. |
| `tilt` | Multi-service hot-reload during dev. |
| `nginx` | Reverse-proxy front for local (mimic prod hostnames). |

## Adding a new option

See "Adding a new adapter" in `docs/CONTRIBUTING.md`, then add a "When to choose" row here.
