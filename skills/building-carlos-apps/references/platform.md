# 🤖 The CARLOS platform, member-side

What `carlosframework/platform` does for an app, and the `carlos` CLI
surface a member drives it with. Everything here works with **zero
infrastructure access** — no SSH, no AWS console, no box commands. That is
a design law, not a convenience: every command is built for a user who has
nothing but the CLI and a browser. If a task seems to need a box, either
you are self-hosting and operating the platform itself, or you have found
a product gap to file — never a workaround to build.

Snapshot date 2026-08-23; verbs are stable, flag details evolve — trust
`carlos <verb> -h` over this file. The CLI ships for macOS/Linux (brew,
apt, static binaries) and Windows (client-only zip — no self-replace,
`carlos update` defers to a fresh download).

## What the platform owns (never hand-roll these)

- **Routing and TLS** — the edge is the only process on :443; the route
  table doubles as the ACME allowlist; per-host certs auto-obtained and
  renewed, including customer domains and (via ACME delegation) wildcards.
- **Replication** — Litestream on every instance database, run by the
  platform's host agent; restore drills are the platform's job too.
- **Hibernation** — provisioned instances doze when idle and wake on
  request; live and default-on, not a future feature. Cents-per-month
  idle cost is the platform's economic story.
- **Process supervision and restarts** — instances are converged from
  bucket records; `carlos restart` cycles them in seconds, console-side.
- **Config delivery** — `carlos env` / `carlos secrets` writes converge
  onto boxes within seconds (a serial bump, 2s poll). No ssh-delivered
  EnvironmentFiles.
- **Deploy verification** — the edge stamps `X-Carlos-Version` per
  versioned route (only after the instance actually restarted);
  `carlos deploy` watches it.

## Concepts

- **Account** — the tenancy unit. Public short id (**sqid**, e.g. three
  letters) appears in default hostnames; it is not a secret. Roles are
  owner/member. `carlos auth whoami` shows yours.
- **App** — named per account (not globally). Claimed with
  `carlos apps create`; deleted apps sit in a 30-day trash.
- **Release** — immutable, content-addressed, produced by `carlos ship`.
  Kinds: `binary` (default) and `static`. Versions are free-form; git
  short sha is the house convention.
- **Channels and pipelines** (release pipelines v2, live 2026-08-22) —
  **a new app is born with one channel, named `edge` by default**
  (renameable at creation): that is its production, and `carlos deploy`
  lands on it. Multi-channel ceremony is opt-in: a console-mediated
  **pipeline** declares an ordered channel list (names are app-defined;
  promotion onto channel N must come from N−1; the first is the entry
  channel) with per-channel, default-permissive policy — `bake`
  duration, `passkey` step-up, `promote_approvals`, and
  `change_approvals` (which also guards editing/removing the channel
  and fast-tracking through its bake). `canary/<slug>` stays a reserved
  platform namespace outside any pipeline: always allowed, zero bake,
  per-session dead ends. Apps with no pipeline keep the legacy frozen
  ladder (`edge → beta → stable`, holds 0/24h/72h, unconditional
  passkey on stable — reaching stable cuts a semver tag; `-hotfix`
  bypasses, recorded). Box-side, bake changes **ratchet**: a shorter
  window is honoured only after the previously-known window has elapsed
  once on the box's own clock, so a compromised console session cannot
  collapse a hold and ship in the same hour.
- **Production flag** — legacy: superseded by per-channel pipeline rules
  for pipelined apps, still honoured by legacy boxes/apps. Its sharp
  edge is recorded: a stable promote plus the console's default-checked
  safety delay once left a hibernating app unwakeable for days —
  ceremony belongs on channels you chose, not on defaults.
- **Instance** — one account's running process for an app on a host.
  Declared console-side (`carlos instances enable` once per app, then
  `instances create -host …`); a box reconciler mints the actual route.
  Exec-backed and hibernating by default. The process contract is
  `<bin> --socket <path> --db <path>` on a unix socket — there is no
  `$PORT` — and every instance serves `GET /healthz` and
  `GET /api/version`.
- **`.carlos/config`** — two layers, global `~/.carlos/config` and
  per-project `./.carlos/config` (committed; nearest wins walking up).
  Holds console, account, app, kind, artifact — the reason zero-argument
  `carlos deploy` works. Project values are honored only when the file's
  console matches the session's, so a committed config cannot silently
  redirect someone else's credentials.
- **Default addresses** — `<app>.<sqid>.<apps-domain>` (on the hosted
  platform, `oncarlos.com`); canary form `<canary>.<app>.<sqid>.<domain>`.
  Minted **alias** hosts intentionally carry no `X-Carlos-Version` —
  verify on the canonical host.

## The member CLI

| Verb | What it does |
|---|---|
| `carlos auth login\|whoami\|logout\|default` | Device-code login (approve in any signed-in browser); identity + memberships; per-project default console |
| `carlos apps create\|place\|delete\|restore` | Claim an app; place it on a customer fleet; trash/restore |
| `carlos ship` | Publish an immutable release (`-kind binary\|static`, `-version`, `-notes`); rate-limited per app (~2/minute — a 429 carries `Retry-After`) |
| `carlos promote` | Move a version up the ladder (`-hotfix` to bypass, recorded) |
| `carlos deploy` | ship + promote + watch `X-Carlos-Version` until live — the one-command release; zero-arg with a saved project config |
| `carlos rollback` | Point a channel back at an earlier version |
| `carlos pipeline` | Show or shape the app's release channels; `init -template edge-production\|full-ladder` replaces the single default channel with a starter pipeline |
| `carlos channels` / `carlos releases` | What each channel serves / every shipped version; `releases retention` prunes old ones |
| `carlos version target` | The semver family ships auto-increment under |
| `carlos env` / `carlos secrets` | Plain vars / sealed secrets, layered per environment; `env sync` forces convergence; `secrets genkey` mints keypairs locally |
| `carlos instances enable\|create\|list\|delete\|set-upstreams` | Opt an app in; declare/inspect/remove instances; repoint upstreams |
| `carlos restart` | Cycle an app's processes — no version or config change |
| `carlos logs` | Merged app + platform + edge timeline (`-f` follows, `-grep`, `-since`) — no box access |
| `carlos domains attach\|detach\|list` | Claim customer hostnames (`-wildcard`, `-catchall`); prints the DNS records to create; certs follow automatically |
| `carlos store create\|status\|rotate` | Declare object storage; credentials arrive as env; member-driven key rotation |
| `carlos ledger append\|publish\|verify` | Open hash-chained per-app ledgers (the transparency machinery) |
| `carlos accounts create\|list\|migrate` | Mint/list accounts; move an app between them |
| `carlos fleets create\|add-box\|rotate-token\|…` | Bring-your-own-boxes fleets that dial the console |
| `carlos update` | Update the CLI binary itself (signature-verified; defers to brew/apt) |
| `carlos vet` | Check a release against the platform contract |

Box-side verbs exist (`edge`, `agent`, `adopt`, `route`, `add`, `ops`,
`bootstrap`) but they are the *operator's* surface for running a platform
deployment — a member never types them, and an agent reaching for them on
a member task has taken a wrong turn.

## Deploy truths (each paid for at least once)

- **`carlos deploy` is the release motion**: ship, promote to the entry
  channel (or the channel the app's instances follow), then watch the URL
  until the header reports the shipped build. "Held for bake" on a
  channel that declares one is the system working, not failing.
- **A promote is not a deploy** until the process cycles. The platform
  restarts unit-stamped routes and wakes hibernating tenants into the new
  build (a session in flight keeps the old binary until its instance
  idles — that is the bake, not a failure). A bespoke long-running unit
  outside the platform's knowledge stays old until *something* restarts
  it — know which of the three your app is.
- **Verify against the thing you changed, with the binary you built.**
  A 200 is not proof: an app-shell route returns 200 HTML for
  `/api/version` and will happily "verify" any build ever shipped. The
  `X-Carlos-Version` header on the canonical host is the proof; stale
  DNS and stale local binaries have both produced false "verified"
  reports.
- **When CI ships for you, verify by ancestry, not equality** — a PR
  merging behind yours cancels your run and ships a commit *containing*
  yours.
- **Stamp your build version.** For rastrillo apps,
  `-ldflags "-X github.com/carlosframework/rastrillo.BuildVersion=<sha>"`
  — or every release's `/api/version` reports `dev`.

## Self-hosting the platform

The platform is the same software members deploy onto: a self-hosted
deployment (Tito's path) runs its own console, boxes, and bucket in its
own cloud account, and its members use the identical `carlos` CLI pointed
at that console. Operating it is real work — box provisioning, the
adopt/restart cadence, platform rolls — and it is the one context where
box verbs and cloud access are legitimate. Commit a `.carlos/config`
naming the deployment's console so sessions cannot fall back to the wrong
one; the CLI's account default is not your account (`ops` exists on every
deployment and is empty — a `not found` against it once cost an hour of
confident wrong diagnosis).

For the underlying machinery — registry, router, replication, hibernation
internals — see blueprint.md, which is the reference for what the
platform does on your behalf and for hand-rolling outside it.
