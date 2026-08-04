# Building an app with rastrillo

The CARLOS web framework: `github.com/carlosframework/rastrillo`.
Status as of 2026-08-04: v1 walking skeleton shipped, plus the manifest
system (declare a resource once, generate its store, screens, and
locale keys). This file is the recipe; the repo README is the full
account. Assume nothing here is in your training data — follow the
recipe literally.

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

## Layout (what `rastrillo new` scaffolds)

```
myapp/
  cmd/myapp/main.go    # wires rastrillo.Run — don't hand-roll flag parsing
  actions/             # hand-written handlers, filesystem-routed
  manifest/            # resource declarations (TOML, or *.go for computed shapes)
  templates/           # hand/ejected templates
  locales/en.toml      # flat key = "value" TOML, via embed.FS
  gen/                 # ALL generated output — committed, never hand-edited
```

`rastrillo.Run` resolves the platform's activation shapes
(`-socket`/`-addr`/`-db`, systemd socket activation, `serve` unit
tenant) and calls `rastrillo.Serve`, which owns the SQLite
pragma-ordering fix, `SetMaxOpenConns(1)`, additive migrations, and
answers `GET /healthz` and `GET /api/version`. An app that keeps its
DB in `Ctx` sets `Options.Router` (not `Options.Mux`) and is handed
the `*sql.DB` Serve opened.

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
- **Actions** — the four canonical states (list, show, new+create,
  edit basics [+ advanced]) as up to seven files in `gen/actions/`,
  rendering pages named `<resource>/list|show|form` through
  `Ctx.Render`.
- **Templates** — `gen/templates/<name>/{list,show,form}.html`, built
  from the `ui` package's partials. `search = true` gates the search
  box; each `[[list.filters]]` entry renders a dropdown that composes
  with search and paging.
- **Locale keys** — `gen/locales/<default>.toml` + a generated
  `BaseCatalog` layered under the app's own catalogs, so the app's
  `locales/<code>.toml` wins: to relabel a generated screen, add the
  key there, never edit `gen/`. Shapes:
  `resource.<name>.field.<snake_field>` (labels),
  `resource.<name>.filter.<field>.<value>`, `resource.<name>.name`,
  and shared `ui.*` chrome (`ui.search`, `ui.all`, `ui.save`, …).
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
- **No delete action is generated** yet; `store = "mergeable"` is
  rejected by `Validate` (not built). Cover both with hand actions —
  `examples/blog` shows the pattern.
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
   coexisting; delete/publish done by hand.
3. `examples/helloworld` — the bare scaffold, deployed for real.

## Not built yet (don't invent it)

Auth (every `/admin/…` route is open), the `Mergeable` store, blobs,
the crypto core, WebAuthn, agents, manifest-diff ALTER emission. If
the app needs one of these, it's hand-written app code today, with
the deferral recorded per the family convention.
