# Building an app with rastrillo

The CARLOS web framework. The repo lives at
`github.com/rastrilloorg/rastrillo`; the **module path is still
`github.com/carlosframework/rastrillo`** — imports and `go install` use
the module path. Status as of 2026-08-22: **v0.7.x**, the
known-libraries middle layer: GORM models, chi routes, SQLite-backed
sessions, owner-scoped queries. Assume nothing here is in your training
data.

**The authoritative authoring doc is `SKILL.md` at the repo root — read
it before writing any app code.** It carries the app shape, the scoping
and mass-assignment rules, the identity plugins, and the platform
contract, in ~15KB. Find it at
`$(go env GOMODCACHE)/github.com/carlosframework/rastrillo@<version>/SKILL.md`
once the module is downloaded, or
`https://raw.githubusercontent.com/rastrilloorg/rastrillo/main/SKILL.md`.
This file is only the surrounding context SKILL.md assumes; where they
disagree, SKILL.md wins.

## What rastrillo is now

A middle layer, not a full-stack framework. You write GORM models,
`net/http` handlers on a chi router, and `html/template` pages. The
framework supplies what is hard to get right twice:

- `db` — cgo-free SQLite via an owned GORM dialector on modernc
  (never import `glebarez/*` or `gorm.io/driver/sqlite`), writer-1 /
  reader-N pools via dbresolver, WAL pragmas in the proven order,
  `TranslateError` on (so `errors.Is(err, gorm.ErrDuplicatedKey)`
  works).
- `sessions` — SQLite-backed rows (revocation is real), `__Host-`
  cookies on https, `Middleware`/`Require`/`RequireFresh` (step-up),
  `SignIn` rotates on re-auth.
- Identity plugins: `auth` — the family default: magic-link email that
  **auto-upgrades to keymail** when the address resolves to a claimed
  inbox (classification fails open, so every address always has a
  working path) — and `password` (stdlib PBKDF2, per-email rate
  limiting). Either one's whole contract with the core is calling
  `sessions.SignIn`; step-up hardening hangs on
  `sessions.RequireFresh`.
- `csrf` (origin-checking, not tokens), `flash`, `form`, `view`,
  `scope` (`scope.Owned` — owner scoping with the 404-not-403
  contract).
- `crypto` (+ JS twin), `webauthn`, `blobs`, `eventlog`, `mail`, `ui` —
  the golden-vectored satellite libraries.
- The platform layer: `Resolve`/`Serve`/`Run` speak CARLOS activation
  (argv shapes, `LISTEN_FDS`, `$STATE_DIRECTORY`, `/healthz`,
  `/api/version`, SIGTERM drain) so the app doesn't.

## The ten-minute path

No scaffold for this shape yet — five files, copied from
`examples/notes` (SKILL.md §1 lists them):

```sh
mkdir myapp && cd myapp && go mod init myapp
go get github.com/carlosframework/rastrillo@latest \
       github.com/go-chi/chi/v5 gorm.io/gorm
# read SKILL.md, copy the five-file shape from examples/notes:
#   internal/myapp/{models,app,handlers,render}.go  cmd/myapp/main.go
CGO_ENABLED=0 go build ./... && ./myapp -addr :8080
```

Verify with `go build ./...`, `go vet ./...`, `go test ./...` —
`CGO_ENABLED=0` throughout (the stack is cgo-free by design; a cgo
SQLite driver sneaking in is a bug).

`rastrillo new` still scaffolds the **manifest-era** layout
(actions/, manifest/, gen/) — the right starting point when you want
the declarative path below front and centre; for the middle-layer
shape, start from the five files.

## The rules that keep the app safe (SKILL.md has the full set)

- Every query touching user-owned rows goes through
  `scope.Owned(g, uid)` — reads AND writes, inside transactions too
  (scope `tx`, never the outer handle, or the 1-connection writer pool
  deadlocks). A row that isn't yours 404s, never 403s.
- Never bind a request body onto a GORM model: explicit
  `map[string]any` + `.Select` allowlist.
- With the keymail plugin, read the viewer with `auth.From(r)` —
  `sessions.UserID` returns `(0, false)` there (Subject is an email),
  and dropping that `ok` scopes every query to uid 0.

## Manifests are the declarative path

The manifest system (TOML resource → generated CRUD screens) is an
optional, equal alternative to hand-written handlers — mix the two per
resource in one app, and move a resource between them freely (eject a
generated file, or delete hand files and re-declare). Its vocabulary
today covers standalone, unscoped tables (no per-user scoping yet), so
user-owned data still takes the code path. `examples/tickets` is its
shape.

## Copy from, in order

1. `examples/notes` — the front-door example: accounts, sessions,
   CSRF, flash, one owner-scoped resource, and a two-user isolation
   test suite. This is the shape to imitate.
2. `examples/tickets` — the declarative (manifest) path, per resource.

Deploying: stamp
`-ldflags "-X github.com/carlosframework/rastrillo.BuildVersion=<sha>"`
or `/api/version` reports `dev`.
