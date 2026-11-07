# Go Hexagonal Service Docs

Architecture and contribution docs for a Go backend built on **Hexagonal Architecture (Ports & Adapters)**: layering rules, a uniform response envelope, a simple error model built on standard HTTP statuses, an adapter catalog, and the commit/PR conventions that keep it consistent.

This repository contains documentation only. Drop it into a Go service (or use it as the starting point for one) and adapt the placeholders listed under [How to use](#how-to-use).

---

## Contents

- [What's inside](#whats-inside)
- [Architecture at a glance](#architecture-at-a-glance)
- [Core rules](#core-rules)
- [Response envelope & error codes](#response-envelope--error-codes)
- [How to use](#how-to-use)
- [What these docs assume](#what-these-docs-assume)
- [Tooling](#tooling)
- [Commit & PR conventions](#commit--pr-conventions)
- [License](#license)

---

## What's inside

| File | What's in it |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Working rules for the service, loaded automatically by Claude Code. Doubles as a concise human-readable rulebook. |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Layering rules, dependency direction, cross-domain access, composition root, testing layers, response envelope, and how to enforce the rules with `depguard`. |
| [`docs/ADAPTERS.md`](docs/ADAPTERS.md) | Adapter catalog: when to choose each persistence, cache, auth, observability, messaging, storage, secrets, notification, jobs, rpc, api-doc, and container option. |
| [`docs/ERROR_CODES.md`](docs/ERROR_CODES.md) | Error model on standard HTTP statuses, the error type contract, how to add feature-specific errors, and design principles learned from running it. |
| [`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md) | Branch, commit, and PR conventions; local and server enforcement; how to add an adapter or a domain. |

Start with `docs/ARCHITECTURE.md`.

---

## Architecture at a glance

```
internal/
├── domain/<feature>/         ← pure business logic; ports declared as interfaces
├── adapter/<axis>/<option>/  ← infrastructure adapters that implement ports
├── http/                     ← inbound HTTP layer (Echo) + envelope renderer
├── app/                      ← composition root: container.go wires everything
└── config/                   ← Viper-loaded typed config

pkg/                          ← shared utilities (logger, ctxutil, customerror, pagination, …)
cmd/                          ← CLI subcommands (serve, migrate, seed)
db/migrations/<dialect>/      ← forward-only SQL migrations
db/seeds/<dialect>/           ← idempotent seed data
docs/                         ← the documents in this repository
```

Dependencies point inward: `http` → `domain` ← `adapter`, and only `app` knows about all of them.

---

## Core rules

1. **Layering.** Domain imports only `pkg/*`. Adapters implement domain ports and never import each other. HTTP talks to the domain through `Service` interfaces. Enforced with `golangci-lint depguard`.
2. **Cross-domain access goes through the other domain's `Service`, never its `Repository`**, so authz, caching, audit, and events cannot be bypassed.
3. **One response envelope.** Handlers use `response.OK`, `response.Paged`, and `response.Err` only — never hand-built JSON.
4. **One error currency.** `customerror.CustomError` is raised from any layer; the HTTP layer is the only place it becomes wire JSON.
5. **Composition root only.** `internal/app/container.go` is the one place that imports many packages. No DI framework.
6. **Forward-only migrations**, idempotent seeds, and `env/env.example` kept in sync with `config.go`.
7. **Layered tests** with coverage targets: domain ≥ 80%, handler ≥ 70%, adapter integration ≥ 60%.

Details and rationale: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) and [`CLAUDE.md`](CLAUDE.md).

---

## Response envelope & error codes

All responses (except the liveness probe) share one shape:

- **Success:** `{ "success": true, "code": 200, "message": "...", "data": {...}, "timestamp": "..." }` (lists add `pagination`)
- **Error:** `{ "success": false, "code": 404, "message": "...", "reason": "TODO_NOT_FOUND", "timestamp": "...", "metadata": {...} }` (`reason` and `metadata` are optional)

`code` is the plain HTTP status code, identical to the HTTP status line. `reason` is an optional `UPPER_SNAKE_CASE` string for the rare case where a client must tell two errors with the same status apart. `DELETE` returns `200` with `data: {}`, not `204`.

The eight standard statuses: `400`, `401`, `403`, `404`, `409`, `422`, `429`, `500`.

Full convention and design principles: [`docs/ERROR_CODES.md`](docs/ERROR_CODES.md).

---

## How to use

1. Copy `CLAUDE.md` to the root of your Go service and `docs/` next to it.
2. Replace the placeholders:

   | Placeholder | Replace with | Where |
   |---|---|---|
   | `<module>` | Your Go module path from `go.mod`, e.g. `github.com/you/project` | `docs/ARCHITECTURE.md` (depguard snippet), your `.golangci.yaml` |
   | `PROJ-123` | Your ticket-ID format | `docs/CONTRIBUTING.md`, hook scripts |

3. Put the depguard rules in your `.golangci.yaml` using the **full module path**. A rule without the module prefix silently matches nothing. Prove the rules work by adding a forbidden import on purpose and confirming `make lint` fails.
4. Delete the options and sections in `docs/ADAPTERS.md` that your service will never ship. Document only what you run and test.
5. Add one small reference domain (a `todo` works well) at `internal/domain/todo/` and treat it as the pattern for every new domain.

---

## What these docs assume

The documents describe code and tooling that this repository does not contain. Either bring them over or write your own equivalents:

| Referenced | Purpose |
|---|---|
| `pkg/customerror`, `pkg/pagination`, `pkg/ctxutil`, `pkg/logger` | Error primitive, paging, request context, logging |
| `internal/http/response/envelope.go` | `OK` / `Paged` / `Err` renderer |
| `internal/app/container.go`, `internal/config/config.go` | Composition root and typed config |
| `env/env.example` | Documented environment variables |
| `.golangci.yaml` | Lint config including the depguard rules |
| `.pre-commit-config.yaml`, `scripts/check-branch-name.sh`, `scripts/check-refs-footer.sh`, `.gitmessage` | Local commit and branch enforcement |
| `.github/workflows/pr-title.yml`, `.github/PULL_REQUEST_TEMPLATE.md` | Server-side PR checks |
| `db/migrations/README.md`, `db/seeds/README.md` | Migration and seed policies referenced from `CLAUDE.md` |

---

## Tooling

**Runtime**

- **Go 1.25**, pinned in `go.mod` and `.tool-versions`
- **Echo v4** — HTTP router and middleware
- **sqlx + Squirrel** — SQL access and query builder
- **Viper** — typed config loaded from `env/env.<APP_ENV>`
- **zerolog** — structured logging

**Dev & test**

- **testify + testcontainers-go** — unit and integration tests against real containers
- **golangci-lint** — lint, including the architecture rules
- **golang-migrate** — forward-only migrations

**Expected Make targets**

```bash
make help               # list every target
make setup              # install tools and pre-commit hooks
make run                # start the service locally
make build              # build the binary
make test-unit          # unit tests
make test-integration   # integration tests (build tag, needs Docker)
make test-cover         # coverage report
make lint               # golangci-lint
```

---

## Commit & PR conventions

**Conventional Commits** with a mandatory `Refs: <TICKET-ID>` footer for `feat` and `fix`. Branch names follow `<type>/<TICKET-ID>-<slug>` (types: `feature | fix | chore | docs | experiment`, slug in kebab-case, 50 characters at most).

Enforced in three layers:

1. **commit-msg hook** — Conventional Commits format and `Refs:` footer
2. **pre-push hook** — branch-name validation, `go test ./...`, `golangci-lint run`
3. **GitHub Action** — PR title validation

Never bypass hooks with `--no-verify`. See [`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md).

---

## License

Choose a license for this repository and add a `LICENSE` file at the root.
