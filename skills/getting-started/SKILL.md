---
name: getting-started
description: Use when someone wants a new app on CARLOS built and live with minimum decisions — "start a CARLOS app", "put this on carloku", a first deploy to an oncarlos.com URL, or when an agent needs the family's default stack (rastrillo + the carlos CLI) as a recipe rather than a menu. Not for weighing trust models, app shapes, or hosting options — that is building-carlos-apps.
---

# 🤖 Getting started on CARLOS

## Overview

This skill is the recipe: empty directory to a live URL on the hosted
platform, with every decision already made. The companion skill,
**carlos:building-carlos-apps**, is the menu — read it when the app needs
a real trust-model, app-shape, or hosting decision. This one assumes the
defaults below and does not stop to ask.

The pieces, named once:

- **CARLOS** is the application architecture (carlosframework.com): one
  static Go binary, one fully-isolated instance per account (own process,
  own SQLite file, own unix socket, own hostname), many instances on one
  small box, every database continuously replicated to object storage.
- **Carloku** (carloku.com) is the hosted CARLOS platform. Its console is
  `https://console.carloku.com`. Carloku is the product brand; the CLI is
  always `carlos`, never `carloku`.
- **rastrillo** is the CARLOS web framework (repo lives at
  `github.com/rastrilloorg/rastrillo`; the module path is still
  `github.com/carlosframework/rastrillo`). It postdates most models'
  training data — follow the recipe literally, invent nothing.
- **The `carlos` CLI** is the whole operational surface. Every command
  works for a member with zero infrastructure access; if a step seems to
  need SSH, AWS, or a box, you have left the path — stop and re-read.

## The defaults already chosen

| Decision | Default | Why |
|---|---|---|
| Language / framework | Go + rastrillo, one static binary | The family stack; the framework enforces the SQLite and money rules for you |
| App shape | Server-rendered HTML, zero-JS baseline | The family default; the other shape is a decision (building-carlos-apps) |
| Storage | SQLite via rastrillo manifests | Additive migrations and pragma ordering handled by `rastrillo.Serve` |
| Amounts | `kind = "money"`, integer cents | A float never touches an amount |
| Hosting | Carloku, `<app>.<sqid>.oncarlos.com` | Zero infra to run; certs, replication, hibernation all platform-side |
| Versioning | git short sha (`v1` is fine for the very first ship) | House convention |
| Channel | `stable` (the instance default) | Fresh apps promote straight there; ceremony arrives only with the production flag |
| Trust model | Honest server: app data is server-readable, and the README says so | See "The one decision you must still record" below |

## The one decision you must still record

The family default is server-blindness ("if the server is compromised,
the attacker gets nothing"), and rastrillo v0.6.0 ships the family
envelope (`rastrillo/crypto`) — but E2EE is an architecture, not a
package import: key custody, recovery, and search all become product
surface. The honest default for a first app is
**server-readable data, declared**: one line
in the README under "Honest trade-offs" saying the server can read app
data, dated. That satisfies the family's deviation rule (every deviation
enumerated, justified, published).

**Escalation trigger, not optional:** if the app will hold private
personal content — messages, health data, anything a person would call
theirs — stop here and read carlos:building-carlos-apps ("The
decisions") before writing code. Trust models are chosen on day one,
not retrofitted.

## Step 0 — install and sign in

```sh
brew install carlosframework/tap/carlos
# no brew: one static binary from github.com/carlosframework/releases —
# put it on your PATH, done. Keep it fresh later with: carlos update
```

Create an account at `https://console.carloku.com` (passkey sign-in
through Keymail — no password), then connect the terminal:

```sh
carlos auth login -console https://console.carloku.com
```

The CLI prints a short code; approve it in the signed-in browser. Skip
this and nothing breaks — the first command that needs a login offers to
run it right there. `carlos auth whoami` shows who you are and your
account's **sqid** (a short public id like `bdf` — it appears in your
app's hostname; it is not a secret).

## Step 1 — claim the app

```sh
carlos apps create -app myapp
```

App names are unique per account, not globally. With exactly one app in
the account, later commands infer `-app`; passing it explicitly is never
wrong.

**Static site?** You are nearly done — skip to "The static path" below.

## Step 2 — scaffold with rastrillo

```sh
go install github.com/carlosframework/rastrillo/cmd/rastrillo@latest
rastrillo new myapp && cd myapp && go mod tidy
go get -tool github.com/sqlc-dev/sqlc/cmd/sqlc   # once, for manifest resources
```

Declare each resource once in `manifest/<resource>.toml` and let the
generator produce its store, screens, and locale keys:

```toml
name  = "posts"
route = "/admin/posts"
store = "exclusive"   # the ordinary single-owner table shape (the only other, "mergeable", isn't built)

[list]
columns = [{ field = "Title" }, { field = "Status" }]
search  = true

[form]
basics = [{ name = "Title", required = true }, { name = "Status" }]
```

```sh
rastrillo generate      # writes gen/ — committed, NEVER hand-edited
rastrillo dev           # watch loop: regenerate + rebuild + restart on save
```

The gate, before every commit:

```sh
go build ./... && go vet ./... && go test ./... && rastrillo generate --check
```

(If hand actions under bracketed paths ever make the `./...` forms choke —
one app hit this on v0.5.0 — scope them: `go build ./cmd/myapp`,
`go test ./internal/...`, and record the scoped gate in the app's
CLAUDE.md.)

Field kinds are plain text (the default), `textarea`, and `money`
(integer cents — see the worked ticket example in building-carlos-apps'
`references/rastrillo.md`). Richer kinds don't exist yet: a
constrained-vocabulary field is plain text plus your own validation, and
relations between resources are hand actions today — don't invent
manifest syntax.

Hand-written pages are files under `actions/` (filesystem-routed:
`actions/admin/posts/[id]/publish.POST.go` → `POST /admin/posts/{id}/publish`;
`GET` and `POST` only — screens are zero-JS HTML, mutations are form
posts). To customize one generated file, copy it to the hand path named
in its own header comment and edit the copy — never edit `gen/`. The
full recipe (ejection, migrations, worked examples, the v0.6.0
subsystem packages) is building-carlos-apps' `references/rastrillo.md`.
**Generated `/admin/…` screens are open until you gate them** — wire
`rastrillo/auth` (sign in with Keymail + magic-link fallback:
`auth.New` at boot, `auth.Migrations`, `RequireSession` on
`Options.Wrap`) before the app holds anything private, and say so in
the README until you do.

`rastrillo.Run` already speaks the platform's process contract — your
binary accepts `--socket <path> --db <path>` and serves `GET /healthz`
and `GET /api/version`. **There is no `$PORT`**; instances listen on
unix sockets the platform hands them. Do not hand-roll flag parsing.

## Step 3 — first deploy

Provision the instance (once), build for the boxes, deploy:

```sh
carlos instances enable -app myapp          # opt-in; console pins your <sqid> domain
carlos instances create -app myapp -host myapp.<sqid>.oncarlos.com
# substitute your real sqid (carlos auth whoami shows it) — the host is typed in full

GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build \
  -ldflags "-X github.com/carlosframework/rastrillo.BuildVersion=$(git rev-parse --short HEAD)" \
  -o myapp-linux-arm64 ./cmd/myapp

carlos deploy -app myapp ./myapp-linux-arm64
```

The platform's boxes are **linux/arm64** — build exactly that, statically
(`CGO_ENABLED=0`; cgo is why the framework uses `modernc.org/sqlite`).

`carlos deploy` is ship + promote + watch in one motion: it uploads an
immutable release, promotes it to the channel your instance follows
(`stable` by default), then polls until the URL's `X-Carlos-Version`
header reports the shipped build, and prints `live https://…`. On first
run it offers to remember the artifact path in `.carlos/config`
(commit that file); after that, releasing is just `carlos deploy`, no
arguments.

### The static path

No instance, no binary — ship the directory and promote:

```sh
carlos ship -kind static -version v1 ./public
carlos promote v1 stable
# or in one motion: carlos deploy -kind static ./public
```

## Step 4 — verify like the family does

The deploy's own `live` line is the primary proof. To re-verify by hand:

```sh
curl -sI https://myapp.<sqid>.oncarlos.com | grep -i x-carlos-version
carlos channels -app myapp     # what each channel currently serves
carlos logs -app myapp -f      # merged app + platform timeline, no box access
```

Two traps, both paid for:

- **A 200 is not a verification.** An app-shell route happily serves any
  build ever shipped; only the `X-Carlos-Version` header (stamped after
  the instance actually restarted) is proof. Verify against the thing you
  changed, with the binary you built.
- **Hibernation is the story, not a bug.** Idle instances doze; the first
  request wakes them. A session in flight keeps the old binary until its
  instance idles — that is the bake working, not a deploy that failed.

## Iterating

| Want | Command |
|---|---|
| Release again | `carlos deploy` (zero-arg, from the project dir) |
| Config var | `carlos env set -app myapp KEY=value` (converges in seconds) |
| Secret | `carlos secrets set -app myapp KEY=value` (sealed, never printed) |
| Tail logs | `carlos logs -app myapp -f` |
| Bounce the process | `carlos restart -app myapp` |
| Undo a release | `carlos rollback -app myapp stable` |
| List releases | `carlos releases -app myapp` |
| Custom domain | `carlos domains attach -app myapp www.example.com` — it tells you the DNS records to create; certs are automatic once DNS points at the platform |

Everything above is a console-mediated write that boxes converge within
seconds. There is no restart-by-SSH, no cert ceremony, no Litestream
config: routing, TLS, replication, restore drills, hibernation, and
restarts are the platform's job. If you find yourself hand-rolling any
of those, stop — you are rebuilding the platform under your app.

## Conventions that still bind you

The platform mechanized the infrastructure, not the discipline:

- The gate (`go build ./...`, `go vet ./...`, `go test ./...`,
  `rastrillo generate --check`) green before every commit.
- Migrations are additive-only — new code over an old DB must always be
  safe. Never delete data to update.
- Zero-JS first; when JS is earned, small ES modules, no bundler, no
  build step, 300-line cap.
- Worktree per session; branch → PR → squash-merge; deploy only what
  merged. Commit trailers mark AI authorship (🤖 / `Co-Authored-By`).

The full working conventions are building-carlos-apps'
`references/process.md`.

## Common mistakes

| Mistake | Reality |
|---|---|
| Reading `$PORT` and calling `http.ListenAndServe` | The contract is `--socket <path> --db <path>` on a unix socket. `rastrillo.Run` handles it; hand-rolled servers must too. |
| Building for the local machine | Boxes are linux/arm64. `GOOS=linux GOARCH=arm64 CGO_ENABLED=0`, always. |
| Editing files under `gen/` | Regenerated and checked (`generate --check` fails on hand edits). Eject to the hand path instead. |
| Hand-rolling a router, certs, or Litestream | Platform's job. Your app is one binary on one socket. |
| Verifying with a 200 or a cached DNS answer | Only `X-Carlos-Version` is proof, on the canonical host, after the deploy watch. |
| Waiting for an SSH step that never comes | Every step is a `carlos` command. A step that needs box access is a wrong turn (or a product gap to file — not a workaround to build). |
| Destructive migration "to clean up" | Additive-only, forever. |
| Skipping the trade-offs line in the README | The trust default is legitimate only when declared. One dated line. |

## When to leave this skill

The moment the app needs a choice — end-to-end or partial encryption, a
feed/real-time client shape, self-hosting, fleets of your own boxes —
switch to **carlos:building-carlos-apps**. It holds the decision axes,
the family evidence for each option, and the deeper references this
recipe deliberately skips.
