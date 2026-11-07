# Contributing

Branch, commit, and PR conventions, plus how to extend the service with a new adapter or domain.

---

## Branch naming

Format: `<type>/<TICKET-ID>-<slug>`

| Type | Use when |
|---|---|
| `feature` | New feature |
| `fix` | Bug fix |
| `chore` | Refactor, dependency update, non-functional change |
| `docs` | Documentation-only change |
| `experiment` | Spike / POC, not intended to merge |

Examples:

- `feature/PROJ-123-payment-flow`
- `fix/PROJ-456-login-redirect`
- `chore/PROJ-789-upgrade-go-1.25`

Rules:

- Slug uses kebab-case.
- Total branch name ≤ 50 chars.
- Direct push to `main` is not allowed.

## Commit messages — Conventional Commits

Format: `<type>(<scope>): <subject>`

```
feat(payment): add stripe checkout integration

Implements card-only flow per spec section 5.2.
Integration tested against stripe sandbox.

Refs: PROJ-123
```

Valid types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`.

Rules:

- Subject ≤ 72 chars, imperative ("add", not "added").
- Body explains **why**, not **what**.
- Footer `Refs: <TICKET-ID>` is required for `feat`/`fix`.
- Breaking change: append `!` after scope (e.g. `feat(api)!: rename endpoint`).
- Never bypass hooks with `--no-verify`. Investigate the failure instead.

## Local enforcement

Hooks installed by `make setup` (via the `pre-commit` framework):

1. `commit-msg` — validates Conventional Commits format + `Refs:` footer for feat/fix.
2. `pre-commit` — fast checks: `go mod tidy`, gitleaks.
3. `pre-push` — validates branch name, then runs `go test ./...` and `golangci-lint run`.

Slow checks (tests, lint) run at pre-push to keep commits fast.

## Server enforcement

A GitHub Action (`.github/workflows/pr-title.yml`) validates the PR title — this matters for squash-merge, where the PR title becomes the commit message.

## PR template

`.github/PULL_REQUEST_TEMPLATE.md` — keep summary, link spec/ticket, declare type, list test plan.

## Supporting files this document assumes

If you copy this file into a new repo, bring (or recreate) these, otherwise the references above dangle:

| File | Purpose |
|---|---|
| `.pre-commit-config.yaml` | Hook wiring |
| `scripts/check-branch-name.sh` | Branch-name check (pre-push) |
| `scripts/check-refs-footer.sh` | `Refs:` footer check (commit-msg) |
| `.gitmessage` | Commit template (`git config commit.template .gitmessage`) |
| `.github/workflows/pr-title.yml` | PR-title check |
| `.github/PULL_REQUEST_TEMPLATE.md` | PR template |
| `Makefile` (`setup`, `test-unit`, `test-integration`, `lint`) | Entry points |

---

## Adding a new adapter

Adapters live under `internal/adapter/<axis>/<option>/` (for example `cache/redis`, `persistence/postgres`).

1. Create the folder under `internal/adapter/<axis>/<new-option>/`.
2. Implement the port interface declared by the domain (or by the axis root package for standalone capabilities such as storage or messaging).
3. Do not import another adapter. Shared types go in the domain or in the axis root package.
4. Add the config fields to `internal/config/config.go` and the matching entries to `env/env.example` (descriptive comment, empty or safe default — never real credentials).
5. Wire it in `internal/app/container.go` (or the per-domain wiring file).
6. Add a depguard rule or extend the existing one so the new adapter tree cannot import its siblings.
7. Persistence adapters: add migrations under `db/migrations/<dialect>/NNN_<name>.up.sql` and idempotent seeds under `db/seeds/<dialect>/`.
8. Tests: integration test with the `integration` build tag and testcontainers; target ≥ 60% coverage for the adapter.
9. Add a "When to choose" row to `docs/ADAPTERS.md`.
10. `make build && make test && make lint` must pass.

## Adding a new domain

1. Create `internal/domain/<name>/` with the entity, `Service` interface + implementation, and the `Repository` port (plus `Cache` or other ports if needed).
2. Domain code imports only `pkg/*` and, for cross-domain access, the other domain's `Service` interface.
3. Raise errors as `customerror.CustomError`; add feature-specific reasons per `docs/ERROR_CODES.md`.
4. Add handwritten mocks under `internal/domain/<name>/mock/`.
5. Add the persistence implementation for each supported dialect.
6. Add handlers + DTOs under `internal/http/`, using `response.OK` / `Paged` / `Err` only.
7. Wire everything in the composition root.
8. Tests: unit (≥ 80%), handler (≥ 70%), adapter integration (≥ 60%). Name tests so they encode the requirement, e.g. `TestList_FR001_ReturnsTodosForUser`.

## Testing

`make test-unit`, `make test-integration`, `make test-cover`, `make lint`. Integration tests are gated by the `integration` build tag and need Docker.
