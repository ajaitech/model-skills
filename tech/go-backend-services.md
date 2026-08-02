# Go Backend Services

## Applies when
`go.mod` exists at repo root.

## Authoritative sources
| Source | URL |
|---|---|
| Go documentation | https://go.dev/doc |
| Go package index | https://pkg.go.dev |
| Go source repo | https://github.com/golang/go |

## Non-obvious rules
- `context.Context` must be the first parameter of any function on a call path that can block (DB queries, HTTP calls, gRPC, channel receives) and propagated explicitly through every layer — it must never be stored on a struct field, per Go's own documented convention, because that hides cancellation and defeats per-call deadlines.
- Error wrapping with `fmt.Errorf("doing X: %w", err)` preserves the chain for `errors.Is`/`errors.As`. Using `%v` instead of `%w` compiles fine and looks identical in logs but silently breaks unwrapping for callers checking sentinel errors or error types.
- `encoding/json` silently skips unexported struct fields — no error, no warning, the field just never appears in the output. A typo'd lowercase field name is a silent data-loss bug, not a compile error.
- A nil pointer of a concrete type assigned to an interface produces a non-nil interface. A function returning `error` via a named concrete `*MyError` type that is nil still makes `err != nil` true at the caller — the classic "nil interface, non-nil box" trap. Prefer returning the interface type directly and only ever returning a literal `nil`, not a typed nil, on success paths.
- Graceful shutdown requires wiring `signal.NotifyContext` (or `signal.Notify`) to `http.Server.Shutdown(ctx)` with a bounded timeout. Container orchestrators send SIGTERM on redeploy/scale-down, not SIGKILL — a service without a SIGTERM handler is killed mid-request instead of draining.
- `go.mod`'s `go` directive sets a minimum language/stdlib version; it does not pin the exact toolchain used to build. CI should run with `GOFLAGS=-mod=readonly` (or `-mod=vendor`) and a committed `go.sum` to prevent an unreviewed dependency bump from entering a build silently.
- A goroutine blocked reading a channel that is never written to or closed leaks for the life of the process. Every goroutine that can block indefinitely needs a `context.Context` cancellation path or an explicit close signal, not just a `select` with no default and no ctx branch.

## Production checklist
- [ ] `go vet` and `go build` clean for every target `GOOS`/`GOARCH` actually shipped
- [ ] SIGTERM wired to `http.Server.Shutdown` (or equivalent) with a bounded drain timeout
- [ ] Structured logging carries a request-scoped trace/request ID through `context.Context`
- [ ] Liveness and readiness endpoints are separate, not the same handler
- [ ] DB and outbound HTTP connection pools have explicit bounded limits, not defaults
- [ ] `go.sum` committed; CI runs with `-mod=readonly`
- [ ] Every error return checked or explicitly discarded with a documented reason (`_ = err // why`)

## Never
- Never store a `context.Context` on a struct field.
- Never silently discard an error return without an explicit, documented reason.
- Never use `%v` where `%w` is needed to keep an error chain unwrappable.
- Never start a goroutine that can block indefinitely without a cancellation or shutdown path.
