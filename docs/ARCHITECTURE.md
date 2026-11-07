# Architecture

This service uses **Hexagonal Architecture (Ports & Adapters)** with light DDD vocabulary. The goal is to keep business logic isolated from infrastructure so that infrastructure choices (DB, cache, message broker, …) are swap-able.

## Dependency direction

```
        ┌───────────────────────────┐
        │     internal/http/        │  Inbound adapter (Echo)
        └─────────────┬─────────────┘
                      │ uses Service interface
                      ▼
        ┌───────────────────────────┐
        │   internal/domain/<X>/    │  Center
        │   • entities              │  No imports from http/adapter/app
        │   • Service interface     │
        │   • Service impl          │
        │   • Repository interface  │  ← PORT
        │   • Cache interface       │
        └─────────────┬─────────────┘
                      ▲ implements ports
                      │
        ┌─────────────┴─────────────┐
        │   internal/adapter/<Y>/   │  Outbound adapters
        │   • postgres / mysql / …  │
        │   • redis / inmemory / …  │
        │   • otel / datadog / …    │
        └───────────────────────────┘

        ┌───────────────────────────┐
        │       internal/app/       │  Composition root: build adapters,
        │       container.go        │  inject into services, register
        │                           │  routes. The ONLY place that
        │                           │  imports many packages.
        └───────────────────────────┘
```

## Allowed / forbidden imports

Enforced by `golangci-lint depguard` rules in `.golangci.yaml` (see [Enforcement](#enforcement)).

| Allowed | Forbidden |
|---|---|
| `domain` → `pkg/*` | `domain` → `adapter/*`, `http/*`, `app/*` |
| `adapter/<X>` → `domain` (to implement port) | `adapter/<X>` → `adapter/<Y>` (no cross-adapter) |
| `http` → `domain` (to call service) | `http` → `adapter/persistence/*`, `cache/*`, `messaging/*`, … |
| `app` → all | `domain.A` → `domain.B.Repository` (must use `domain.B.Service`) |

Note: if your auth middleware lives in an adapter (`internal/adapter/auth/*`), `internal/http` may import it — the middleware bridges the HTTP boundary. Exclude that path from the `http-no-adapter` rule.

## Cross-domain access

A domain accesses another domain's data **only via that domain's `Service` interface**, never via its `Repository`. This preserves business rules (soft delete, authz, cache, audit, events) that the service enforces.

```go
// internal/domain/order/service.go (hypothetical)
type service struct {
    repo        Repository
    customerSvc customer.Service   // ← cross-domain via interface
}

func (s *service) CreateOrder(ctx context.Context, ...) error {
    customer, err := s.customerSvc.FindByID(ctx, customerID)  // not customerRepo
    // ...
}
```

Wiring in `app/container.go` injects the concrete `customerService` into `orderService`.

### Why not access the repository directly?

Going through the repository directly bypasses business rules carried by the service:

- Soft-delete visibility filters
- Authorization checks against `ctxutil.UserID` / `Roles`
- Cache invalidation on mutation
- Audit logging
- Domain events / outbox

The performance "win" is illusory — Go interface dispatch is essentially free, and the business consequences of bypassing the service are real (security, consistency, observability gaps).

## Composition root

`internal/app/container.go` is the **only** file that imports many adapter packages. It:

1. Reads typed config (`config.Load`)
2. Connects to selected infrastructure (Postgres, Redis, …)
3. Wires adapters into domain services
4. Constructs HTTP handlers and middleware

As the service grows, split the wiring per domain (e.g. `wire_<domain>.go` in `internal/app/`) instead of letting `container.go` become a single large file. Keep it explicit — no DI framework, no reflection.

## Configuration

`internal/config/config.go` declares a single typed `Config` struct with one field group per adapter. Loaded via Viper from `env/env.<APP_ENV>` (defaults to `local`). Every variable read in `config.go` must have a matching entry in `env/env.example`, added in the same commit.

## Testing layers

| Layer | Path | Tooling | Coverage target |
|---|---|---|---|
| Unit (domain) | `internal/domain/<x>/*_test.go` | testify + handwritten mocks | ≥ 80% |
| Handler | `internal/http/handler/*_test.go` | httptest + mock service | ≥ 70% |
| Integration (adapter) | `internal/adapter/persistence/<x>/*_test.go` (build tag `integration`) | testcontainers-go | ≥ 60% per adapter |

Run: `make test-unit`, `make test-integration`, `make test-cover`.

## Response envelope

All HTTP responses (except the liveness probe) follow one envelope contract. Three shapes:

| Envelope             | Used for                                  | Required fields |
|----------------------|-------------------------------------------|-----------------|
| `SuccessEnvelope`    | Single-resource success + no-content      | `success`, `code`, `message`, `data`, `timestamp` |
| `PaginatedEnvelope`  | List endpoints                            | `SuccessEnvelope` + `pagination` |
| `ErrorEnvelope`      | All error responses                       | `success`, `code` (HTTP status), `message`, `timestamp`; optional `reason`, `metadata` |

The renderer lives at `internal/http/response/envelope.go` and exposes exactly three helpers — `OK`, `Paged`, `Err`. Handlers MUST use them; manual `c.JSON` of envelope-shaped maps defeats the single-source-of-truth guarantee.

### Package boundaries

```
internal/http/handler        → calls response.OK / Paged / Err
internal/http/response       → owns envelope shape; depends on pkg/customerror + pkg/pagination
pkg/customerror              → error primitive (HTTP-agnostic); raisable from any layer
pkg/pagination               → Page struct + paging math (HTTP-agnostic)
```

`pkg/customerror.CustomError` is the cross-layer error currency. Domain, service, and repository layers all raise it. The HTTP layer (`response.Err`) is the only place that translates `CustomError` to wire JSON.

### Error codes

The `ErrorEnvelope.Code` field is the plain HTTP status code (`404`, `409`, …). The base ships 8 standard errors in `pkg/customerror`; features add an optional `reason` string (e.g. `TODO_NOT_FOUND`) only when a client must tell two errors with the same status apart.

Full convention: see `docs/ERROR_CODES.md`.

### No-content responses

Endpoints that conceptually return "no payload" (e.g., `DELETE`) still return `200 OK` + envelope with `data: {}`. This trades the HTTP-idiomatic `204 No Content` for envelope universality — every client parses the same shape; observability tooling sees the same fields.

`204 No Content` is reserved for non-JSON endpoints (webhook acks, file streams).

## Enforcement

The layering rules are only as strong as the lint config that backs them.

```yaml
# .golangci.yaml (excerpt)
linters:
  settings:
    depguard:
      rules:
        domain-no-infra:
          list-mode: lax
          files: ["**/internal/domain/**/*.go"]
          deny:
            - pkg: "<module>/internal/adapter"
              desc: "domain must not import infrastructure adapters"
            - pkg: "<module>/internal/http"
              desc: "domain must not import the HTTP layer"
            - pkg: "<module>/internal/app"
              desc: "domain must not import app wiring"
        http-no-adapter:
          list-mode: lax
          files: ["**/internal/http/**/*.go"]
          deny:
            - pkg: "<module>/internal/adapter"
              desc: "HTTP handlers must use domain interfaces, not adapters directly"
        # one rule per adapter port (cache, persistence, …) denying the others
```

**Gotcha:** `pkg` is matched as a prefix of the *full import path*, so `<module>` must be the exact module path from `go.mod` (e.g. `github.com/you/project`). A path without the module prefix is silently accepted and matches nothing — the rule looks active but enforces nothing. After renaming the module, update this file, then prove the rule works by temporarily adding a forbidden import and confirming `make lint` fails.

**Limit:** `domain.A → domain.B.Repository` cannot be linted, because `Repository` and `Service` live in the same package. That rule is held by code review.
