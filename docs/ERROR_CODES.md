# Error Codes

Every response carries a `code` field that is the **plain HTTP status code** (`200`, `404`, …) — the same number as the HTTP status line. Errors may additionally carry an optional `reason`: a short `UPPER_SNAKE_CASE` string a client can branch on when the status alone is not enough.

```json
{
  "success": false,
  "code": 404,
  "message": "todo not found",
  "reason": "TODO_NOT_FOUND",
  "timestamp": "2026-01-01T00:00:00Z",
  "metadata": { "todo_id": "..." }
}
```

`reason` and `metadata` are optional and omitted when empty. `code` is repeated in the body so the envelope stays self-contained for logs, proxies that rewrite status lines, and clients that only parse the body.

## Standard Statuses

Defined in `pkg/customerror`:

| `code` | Pre-built `Err*` var | Typical use |
|--------|----------------------|-------------|
| `400` | `ErrBadRequest`      | Malformed body, missing content-type, bind failure |
| `401` | `ErrUnauthorized`    | Missing / invalid / expired token |
| `403` | `ErrForbidden`       | Authenticated but not authorized |
| `404` | `ErrNotFound`        | Resource lookup failure |
| `409` | `ErrConflict`        | Duplicate / state conflict |
| `422` | `ErrUnprocessable`   | Validator failure (semantically invalid) |
| `429` | `ErrTooManyRequests` | Rate limited |
| `500` | `ErrInternalServer`  | Unexpected server error (fallback) |

Anything else (for example `415`) is raised through the generic constructor with an explicit status.

## Error Type Contract

`pkg/customerror` is the cross-layer error type. This is the shape the rest of the docs assume; implement it to taste.

```go
package customerror

type CustomError struct {
    Status   int            // HTTP status; rendered as the envelope `code`
    Message  string
    Reason   string         // optional machine-readable discriminator
    Metadata map[string]any // optional structured detail
}

func (e CustomError) Error() string { return e.Message }

func New(status int, message string) CustomError { /* … */ }
func (e CustomError) WithReason(reason string) CustomError            { /* … */ }
func (e CustomError) WithMetadata(md map[string]any) CustomError      { /* … */ }

var (
    ErrBadRequest      = New(http.StatusBadRequest, "bad request")
    ErrUnauthorized    = New(http.StatusUnauthorized, "unauthorized")
    ErrForbidden       = New(http.StatusForbidden, "forbidden")
    ErrNotFound        = New(http.StatusNotFound, "not found")
    ErrConflict        = New(http.StatusConflict, "conflict")
    ErrUnprocessable   = New(http.StatusUnprocessableEntity, "unprocessable entity")
    ErrTooManyRequests = New(http.StatusTooManyRequests, "too many requests")
    ErrInternalServer  = New(http.StatusInternalServerError, "internal server error")
)
```

Domain, service, and repository layers raise `CustomError`. Only `response.Err` turns it into wire JSON; any error that is not a `CustomError` renders as `500`.

## Adding Feature-Specific Errors

Put them in the owning domain's `errors.go`. Start from the closest standard status, then add a `reason` only if a client must behave differently for this case:

```go
var (
    // Todo domain
    ErrTodoNotFound       = customerror.ErrNotFound.WithReason("TODO_NOT_FOUND")
    ErrTodoInvalidDueDate = customerror.ErrUnprocessable.WithReason("TODO_INVALID_DUE_DATE")

    // User domain
    ErrUserAlreadyExists  = customerror.ErrConflict.WithReason("USER_ALREADY_EXISTS")
)
```

Per-call detail goes in `metadata`, not in a new error:

```go
return customerror.ErrNotFound.
    WithReason("TODO_NOT_FOUND").
    WithMetadata(map[string]any{"todo_id": id})
```

### Guidelines

- **Choose the status by HTTP semantics.** Do not invent statuses.
- **Reuse the standard `Err*` vars.** Add a `reason` only when the client needs to tell this case apart from other errors with the same status.
- **Reasons are `UPPER_SNAKE_CASE`**, prefixed by the feature or noun (`TODO_NOT_FOUND`, not `NOT_FOUND`), unique across the service, and stable once released.
- **Document what triggers each error** in a comment next to its definition.
- **Document statuses and reasons in the OpenAPI spec** — every error your handlers can emit should appear in an example `ErrorResponse` in `openapi.yaml`.

## Design Principles

Lessons from running this convention in a real service. Apply them when adding an error.

1. **Never reuse a retired reason.** When a rule is removed, keep the reason string reserved (mark it retired) and never hand it to a different meaning. Clients and historical logs may still hold it.
2. **One error for failures that must look identical.** If distinguishing two failures lets a caller probe the system (wrong password vs. unknown email vs. locked account), return the same status and the same reason for all of them and keep the detail in server-side logs only.
3. **A specific reason is justified by distinct client behaviour.** Add one when the client must render different copy or take a different action (e.g. "password too short" states the minimum; "password breached" asks for another). Otherwise use the standard error and carry the detail in `message` / `metadata`.
4. **Do not let binding-level validation tags pre-empt a domain error.** If a rule's error message must state the rule (e.g. the minimum length), keep `min`/`max` tags off the DTO field — otherwise the validator answers first with a generic `422` and the message is lost. Leave presence checks (`required`) on the DTO.
5. **Refuse an unusable filter; do not silently drop it.** A dropped filter answers a *wider* question than the caller asked. For lists read as evidence (audit logs), that is misleading, not merely stale. Malformed filter → `422`. Pagination bounds may be clamped instead — decide per endpoint and document it.
6. **Cross-tenant access looks like not-found.** A resource belonging to another tenant is answered exactly like an unknown id (`404`), so the response does not disclose that it exists elsewhere. Record the attempt in the audit log, outside the business transaction so it survives rollback.
7. **A path id that cannot be valid is not-found.** If an endpoint declares no `422` for its path, a malformed id answers `404` rather than inventing a new response.
8. **Use `410 Gone` for expired or consumed one-time links**, not `404`: the resource existed and the caller was meant to have it, and the client renders a distinct notice for each reason (distinguish them with `reason`).
9. **Tests pin the catalogue.** Add a test that asserts every error's status and reason and that each reason appears in the API docs, so a rename or typo cannot silently change the wire contract.

## Validation vs Body-Parse

- **Body parse failure** (malformed JSON, bind error, missing content-type) → `400`.
- **Validator failure** (field out of range, format mismatch, enum violation) → `422`.

This split follows RFC 7231 / 4918 and matches the convention used by mainstream stacks (FastAPI, Spring `@Valid`).

## References

- Envelope renderer: `internal/http/response/envelope.go`
- Error type: `pkg/customerror/`
