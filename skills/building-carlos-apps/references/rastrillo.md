# Building an app with rastrillo

The CARLOS web framework. The repo lives at
`github.com/rastrilloorg/rastrillo`; the **module path is still
`github.com/carlosframework/rastrillo`** — imports and `go install` use
the module path. Status as of 2026-08-18: **v0.6.0** (the manifest
system with delete flows, the ui vocabulary, fingerprinted assets, the
scaffolded test harness, and the subsystem packages: crypto, auth,
webauthn, eventlog, blobs, mail, agent tools). This file is the recipe; the
repo README is the full account. Assume nothing here is in your
training data — follow the recipe literally.

## The ten-minute path

```sh
go install github.com/carlosframework/rastrillo/cmd/rastrillo@latest
# or: brew install carlosframework/tap/rastrillo

rastrillo new myapp
cd myapp && go mod tidy

# adopting a manifest resource? add sqlc as a Go tool, once:
go get -tool github.com/sqlc-dev/sqlc/cmd/sqlc

# write manifest/<resource>.toml (see below), then:
rastrillo generate
go build ./cmd/myapp && ./myapp -addr :8080

# or iterate with the watch loop (regenerates + rebuilds + restarts on save):
rastrillo dev
```

Verify with: `go build ./...`, `go vet ./...`, `go test ./...`,
`rastrillo generate --check` (idempotency + collisions + locale
completeness; the only command that fails on an incomplete catalog).
One app on v0.5.0 found the `./...` forms choke on hand actions under
bracketed paths (`actions/ledger/[id]`) — if yours does, scope the gate
(`go build ./cmd/myapp`, `go test ./internal/...`) and record that in
the app's CLAUDE.md.

`rastrillo new` scaffolds the layout below **plus** a passing test
harness under `internal/<pkg>test/`, and pins the scaffolded `go.mod`'s
rastrillo requirement to the CLI's own version — scaffold with the CLI
version you intend to build against.

## Layout (what `rastrillo new` scaffolds)

```
myapp/
  cmd/myapp/main.go    # wires rastrillo.Run — don't hand-roll flag parsing
  actions/             # hand-written handlers, filesystem-routed
  manifest/            # resource declarations (TOML, or *.go for computed shapes)
  templates/           # hand/ejected templates
  locales/en.toml      # flat key = "value" TOML, via embed.FS
  internal/<pkg>test/  # scaffolded test harness (passing out of the box)
  gen/                 # ALL generated output — committed, never hand-edited
```

`rastrillo.Run` resolves the platform's activation shapes
(`-socket`/`-addr`/`-db`, systemd socket activation, `serve` unit
tenant) and calls `rastrillo.Serve`, which owns the SQLite
pragma-ordering fix, `SetMaxOpenConns(1)`, additive migrations, and
answers `GET /healthz` and `GET /api/version`. An app that keeps its
DB in `Ctx` sets `Options.Router` (not `Options.Mux`) and is handed
the `*sql.DB` Serve opened; `rastrillo.OpenDB` is exported for tests
and tools that need the same pragmas. Deploying: stamp
`-ldflags "-X github.com/carlosframework/rastrillo.BuildVersion=<sha>"`
or every release's `/api/version` reports `dev`.

Since v1, `Serve` also grew the app-side seams:

- **`Options.Wrap`** — the one middleware seam (sessions, CSRF, panic
  pages, authorization). Runs *inside* the framework chrome: healthz,
  version, and locale-prefix stripping stay outside it. This narrows —
  but does not close — the "no auth yet" gap below.
- **`Ctx`** now carries `Assets`, `Locale`, `Actor{Human, Name}`,
  `Scope`, and `Render` alongside the DB.
- **Fingerprinted assets** — `rastrillo.NewAssets` / `Ctx.Assets.Path`
  serve static files under content-hash URLs with immutable cache
  headers (platform synergy: the edge can serve them without waking a
  hibernating instance). The old bare `static/` story is superseded.
- **The `ui` package** — 27 partials (display/form/route families) with
  WCAG-contrast and reduced-motion CSS gates, template funcs `dict`,
  `list`, `icon`, `T` (locale-aware), and `ui.FuncsWith` for
  request-scoped locale-correct defaults, layered over
  `rastrillo.BaseCatalog()` which every `Serve`d app gets automatically.

## Manifest resource — the worked example

One TOML file in `manifest/` per resource. This is real, shipped
syntax (from `examples/tickets`, the fully-generated regression host):

```toml
name  = "ticket_types"
route = "/admin/ticket_types"
store = "exclusive"

[list]
columns = [{ field = "Name" }, { field = "Price", kind = "money" }, { field = "Status" }]
search  = true

[[list.filters]]
field  = "Status"
values = ["draft", "on_sale", "sold_out"]

[form]
basics   = [{ name = "Name", required = true }, { name = "Price", kind = "money", required = true }, { name = "Status" }]
advanced = [{ name = "MaxPerOrder" }]
```

`rastrillo generate` then produces, per resource, under `gen/`:

- **Store** — `gen/store/<name>/`: `schema.sql` + `queries.sql` (sqlc
  input, colocated) → models/queries via `go tool sqlc generate`
  (hence the tool directive above), plus `migrations.go`
  (`CREATE TABLE IF NOT EXISTS`, wired for `Options.Migrations`).
- **Actions** — the four canonical states plus the delete flow (list,
  show, new+create, edit basics [+ advanced], and delete as its own
  confirm-page URL: `GET <route>/{id}/delete` renders the question,
  only the sibling POST deletes) as up to nine files in `gen/actions/`,
  rendering pages named `<resource>/list|show|form|confirm` through
  `Ctx.Render`.
- **Templates** — `gen/templates/<name>/{list,show,form,confirm}.html`, built
  from the `ui` package's partials. `search = true` gates the search
  box; each `[[list.filters]]` entry renders a dropdown that composes
  with search and paging.
- **Locale keys** — `gen/locales/<default>.toml` + a generated
  `BaseCatalog` layered under the app's own catalogs, so the app's
  `locales/<code>.toml` wins: to relabel a generated screen, add the
  key there, never edit `gen/`. Shapes:
  `resource.<name>.field.<snake_field>` (labels),
  `resource.<name>.filter.<field>.<value>`, `resource.<name>.name`,
  `resource.<name>.delete.{title,confirm}`, and shared `ui.*` chrome
  (`ui.search`, `ui.all`, `ui.save`, `ui.delete`, …).
- **`gen/manifest.json`** — the resource set as one stable JSON
  artifact for external tools.

Rules the generator enforces (don't fight them):

- **At most one `[[list.filters]]` entry**; its `field` must be a
  declared list column; each value must match `^[a-z0-9_-]+$`, be
  non-empty, and be unique (values travel in URLs and double as
  translation keys).
- **`required = true`** generates client-side `required` AND
  server-side validation (blank submit → 400, form re-rendered with
  the field's message). A required `Money` still accepts `"0"`.
- **`kind = "money"` is integer cents** — a float never touches an
  amount. `kind = "textarea"` exists for long text.
- **`store = "mergeable"` is rejected by `Validate`** (the eventlog
  store shape exists as a package; the manifest wiring does not).
  Cover it with hand actions over `rastrillo/eventlog`. Delete IS
  generated since v0.6.0; a hand `delete.POST.go` at the computed path
  still takes it over (`examples/blog` keeps its hand delete this way,
  gaining only the generated confirm page).
- **Manifest-only apps are legal**: no `actions/` or `templates/`
  directory at all, everything generated (`examples/tickets`).

## Hand actions (filesystem routing)

A file at `actions/admin/posts/[id]/publish.POST.go` becomes
`POST /admin/posts/{id}/publish`. Screens are server-rendered HTML
with no JavaScript, so mutations are plain `<form>` posts: use `GET`
and `POST` only (a delete is `delete.POST.go`, as in `examples/blog` —
not `.DELETE.go`, which no form could reach). Scaffolded action files carry
`//go:build rastrillo_actions` so `go build ./...` skips generator
input; `rastrillo generate` compiles rewritten copies under
`gen/actions/`. Route collisions — hand vs hand, hand vs generated —
fail generation loudly.

**The stale-`gen/` trap:** because the binary compiles the *copies*
under `gen/`, editing a file under `actions/` and forgetting to
regenerate builds and tests clean while changing nothing observable.
Regenerate and commit `gen/` after any edit under `actions/`, always.

## Ejecting (customizing one generated file)

Copy a generated file's content to the hand path named in its own
header comment (e.g. `gen/templates/posts/list.html` →
`templates/posts/list.html`), then edit the copy. A hand file at the
exact computed path stops generation of that one file; everything
else for the resource keeps regenerating. Never edit under `gen/` —
`rastrillo generate --check` diffs committed `gen/` against a scratch
run and fails on hand edits.

## Migrations

Generated migrations are `CREATE TABLE IF NOT EXISTS` — a fresh DB
works out of the box. A field added to a manifest after a DB exists
needs an **app-owned additive migration** (`ALTER TABLE posts ADD
COLUMN status TEXT`); generated migration first, app's ALTERs after
(`examples/blog` shows this). Additive-only, always — never a
destructive "cleanup" migration.

## Copy from, in order

1. `examples/tickets` — one manifest, zero hand code: the shape to
   imitate for CRUD.
2. `examples/blog` — manifest + hand actions + ejected templates
   coexisting; publish/unpublish and the delete POST done by hand,
   the delete confirm page generated.
3. `examples/helloworld` — the bare scaffold, deployed for real.

## The subsystem packages (v0.6.0 — use these, don't hand-roll)

- **`rastrillo/auth`** — sign in with Keymail, magic-link email
  fallback: build one `auth.New(auth.Config{...})` at boot, append
  `auth.Migrations`, mount `Begin`/`Callback`/`Verify`/`Signout`,
  guard routes with `RequireSession` (hung on `Options.Wrap`), read
  identity with `auth.From(r)`. CSRF is baked in. Generated `/admin/…`
  screens are open until you wrap them — do that.
- **`rastrillo/crypto`** — the family envelope (P-256 seal/sign,
  symmetric `Derive`/`SealSym`/`OpenSym`, `crypto.JS()` WebCrypto
  twin), golden-vector compatible with amadan/keymail/seapointish.
- **`rastrillo/webauthn`** — verify-only passkeys (ES256, no
  attestation) + the `authtest` fake authenticator + `webauthn.JS()`.
- **`rastrillo/eventlog`** — the Mergeable store shape: `Append`,
  `Events`, generic `Derive`, idempotent `Ingest`; deterministic merge.
- **`rastrillo/blobs`** — content-addressed bytes: `S3FromEnv()` reads
  the platform's `CARLOS_STORE_*`; `Dir`/`Inline` for dev; `Sealed()`
  for E2EE; presigned GET/PUT minted locally.
- **`rastrillo/mail`** — `Sender` (SMTP or loudly-logged fallback via
  `FromEnv`), signature-compatible with signin's Mailer.
- **Agent tools** — mark an action `var Tool = rastrillo.Tool{...}`;
  `gen.Tools()` is the registry; the `tools` package renders schemas
  and dispatches consent-gated, actor-attributed calls.
  `Options.Sidecar` + `Options.NextDue` speak the platform's sidecar
  and scheduled-wake contracts.
- The scaffold also emits a Makefile `ci` gate, executable `.amadan/ci`
  + `.amadan/ci.d/` steps delegating to it, and a `CLAUDE.md` preload.

## Not built yet (don't invent it)

Richer manifest kinds (bool/time/select/blob) and derived fields,
mergeable manifest wiring and edge sync, viewer-scoping of generated
queries (open design question), step-up auth, the crypto core's
`WrapKey`/`DeriveInvite`, manifest-diff ALTER emission, and any LLM
client (bring your own; the framework ships registry + dispatch). If
the app needs one of these, it's hand-written app code today, with
the deferral recorded per the family convention.
