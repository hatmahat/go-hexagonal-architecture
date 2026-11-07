# Go Service — Working Rules

Loaded automatically by Claude Code when working in this service.

## Stack at a glance

Go 1.25 · Echo v4 · sqlx + Squirrel · Viper · zerolog · testify +
testcontainers. Architecture: Hexagonal (Ports & Adapters).

→ Full architecture: `docs/ARCHITECTURE.md`
→ Error codes: `docs/ERROR_CODES.md`
→ Contribution conventions: `docs/CONTRIBUTING.md`

## Rule 1 — Hexagonal layering

Strict dependency direction enforced by `golangci-lint depguard`:

- Domain (`internal/domain/<feature>/`) declares ports as interfaces.
  Pure business logic. May import only `pkg/*`. May NOT import
  `internal/adapter/*`, `internal/http/*`, `internal/app/*`.
- Adapters (`internal/adapter/<axis>/<option>/`) implement domain ports.
  Each adapter may NOT import another adapter.
- Inbound HTTP (`internal/http/`) translates HTTP ↔ domain via service
  interfaces. It may not import adapter types (an auth-middleware adapter
  may be explicitly excluded from that rule).
- Composition root (`internal/app/container.go`) is the only file that
  imports many packages — it wires concrete adapters into domain
  services.
- Cross-domain access goes through the other domain's `Service`
  interface, never its `Repository`. The service carries business
  rules (authz, cache, audit, events) you must not bypass.

The depguard `pkg:` entries must use the **full module path** from
`go.mod`. A path without the module prefix matches nothing and the rule
silently enforces nothing. When you rename the module, update
`.golangci.yaml` too.

→ Detail + diagrams + depguard snippet: `docs/ARCHITECTURE.md`

## Rule 2 — Response envelope

Every HTTP response (except `/healthz`) uses helpers from
`internal/http/response/`:

- `response.OK(c, http.StatusOK, "message", data)` — single-resource success
- `response.Paged(c, "message", data, pagination)` — list with pagination
- `response.Err(c, err)` — error (accepts any `error`; non-custom errors render as 500)

Shape:

```json
// Success
{"success": true, "code": 200, "message": "...", "data": {...}, "timestamp": "..."}
// Paginated (adds "pagination")
{"success": true, "code": 200, "message": "...", "data": [...], "pagination": {...}, "timestamp": "..."}
// Error
{"success": false, "code": 404, "message": "...", "reason": "TODO_NOT_FOUND", "timestamp": "...", "metadata": {...}}
```

- `code` is the plain HTTP status code (same as the status line).
  `reason` (optional, `UPPER_SNAKE_CASE`) and `metadata` (optional) are
  omitted when empty. Add a `reason` only when a client must tell two
  errors with the same status apart.
- DELETE endpoints return `200 OK` + `data: {}` — NOT 204.
- Handlers MUST go through these helpers. Manual `c.JSON(...)` of
  envelope-shaped maps defeats the single-source-of-truth guarantee.

→ Error model + how to add errors: `docs/ERROR_CODES.md`
→ Envelope contract: `docs/ARCHITECTURE.md#response-envelope`

## Rule 3 — Requirement annotations (optional but recommended)

If you track functional requirements, every handler, service method,
or repository method that implements one carries a
`// FR-<ID>: <one-line description>` comment above its declaration, so
coverage can be computed by grep.

```go
// FR-001: List todos for the authenticated user, paginated.
func (h *TodoHandler) List(c echo.Context) error { ... }
```

Test names encode the same ID: `TestList_FR001_ReturnsTodosForUser`.

## Rule 4 — Persistence adapter parity

Each `internal/adapter/persistence/<dialect>/` folder MUST compile and
satisfy the same domain `Repository` port. When you rename a domain
entity, change a port signature, or add a port method:

- Update every dialect adapter you ship.
- Update migrations: `db/migrations/<dialect>/NNN_<name>.up.sql`.
- Update seeds: `db/seeds/<dialect>/NNN_<name>.{sql,json}`.

Only ship a dialect you are willing to test. A compile-only adapter
gives false assurance — prefer one dialect with full testcontainers
coverage over three without.

## Rule 5 — Migration & seed policy

- **Forward-only.** No `.down.sql` files. Local-dev rollback uses
  drop + remigrate.
- File prefix `NNN_` (zero-padded, monotonic per dialect):
  `001_create_users.up.sql`, `002_create_todos.up.sql`.
- Seeds are idempotent (`ON CONFLICT DO NOTHING` for postgres,
  `INSERT IGNORE` for mysql, `$setOnInsert` upsert for mongodb).

## Rule 6 — Auth adapter contract

`internal/adapter/auth/<option>/` (jwt, apikey, oauth2-oidc, none)
implement the same middleware port. A new auth option MUST:

- Satisfy the domain `auth` port interface.
- Provide an echo middleware factory.
- Compile and satisfy the interface even when stubbed. Stub options
  may return "not implemented" at runtime, but the type contract must
  hold.

## Rule 7 — Testing pattern

| Layer | Path | Tooling | Coverage target |
|---|---|---|---|
| Unit (domain) | `internal/domain/<x>/*_test.go` | testify + handwritten mocks | ≥ 80% |
| Handler | `internal/http/handler/*_test.go` | httptest + mock service | ≥ 70% |
| Integration (adapter) | `internal/adapter/persistence/<x>/*_test.go` (build tag `integration`) | testcontainers-go | ≥ 60% per adapter |

Run via `make test-unit`, `make test-integration`, `make test-cover`.

## Rule 8 — Config & secrets

- Typed config in `internal/config/config.go`, loaded via Viper from
  `env/env.<APP_ENV>` (defaults to `local`).
- Secret resolution via `internal/adapter/secrets/<option>/` (env /
  gcp / aws / vault). The `env` option is implicit via `config.Load`;
  cloud sources are wired manually in `container.go`.
- NEVER log secret values. Use structured logger redaction.
- **env/env.example parity:** Every env var read in `config.go` MUST
  have a matching entry in `env/env.example` with a descriptive comment
  and an empty or safe default value (never real credentials). Add the
  entry in the same commit as the config field. A missing entry is a
  review blocker.

## Rule 9 — Logging

- zerolog structured. Get the logger via `pkg/logger`.
- Levels:
  - `debug` — verbose dev tracing (not in prod by default)
  - `info` — request flow, lifecycle events
  - `warn` — recoverable issue (deprecated path, retry)
  - `error` — 5xx-class failure
- Always include `requestID` from `pkg/ctxutil`.

## Rule 10 — Commit & PR

- Conventional Commits: `<type>(<scope>): <subject>` with body
  explaining **why**, not **what**.
- `feat` and `fix` REQUIRE a `Refs: <TICKET-ID>` footer.
- Branch: `<type>/<TICKET-ID>-<slug>` (≤ 50 chars, kebab-case slug).
- NEVER `--no-verify`. Pre-push runs `go test ./...` + `golangci-lint run`.

→ Hook detail: `docs/CONTRIBUTING.md`

## Reference domain

Keep one small end-to-end domain (a `todo` is the usual choice) at
`internal/domain/todo/` as the canonical example of: entity, `Service`,
`Repository` port, handler + DTOs, mocks, and all three test layers.
When adding a real domain, copy its shape rather than inventing a new
one.

## Quick reference

| Topic | Source |
|---|---|
| Layering rules + diagrams + lint config | `docs/ARCHITECTURE.md` |
| Adapter catalog (when to choose each option) | `docs/ADAPTERS.md` |
| Error model + design principles | `docs/ERROR_CODES.md` |
| Commit/PR/hook detail, adding adapters/domains | `docs/CONTRIBUTING.md` |
| Migration policy | `db/migrations/README.md` |
| Seed policy | `db/seeds/README.md` |
