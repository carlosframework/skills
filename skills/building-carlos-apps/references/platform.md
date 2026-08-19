# 🤖 The CARLOS platform, member-side

What `carlosframework/platform` does for an app, and the `carlos` CLI
surface a member drives it with. Everything here works with **zero
infrastructure access** — no SSH, no AWS console, no box commands. That is
a design law, not a convenience: every command is built for a user who has
nothing but the CLI and a browser. If a task seems to need a box, either
you are self-hosting and operating the platform itself, or you have found
a product gap to file — never a workaround to build.

Snapshot date 2026-08-17; verbs are stable, flag details evolve — trust
`carlos <verb> -h` over this file.

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
- **Channel ladder** — `canary → edge → beta → stable`, climbed by
  `carlos promote`. Reaching `stable` cuts a semver tag; `-hotfix`
  bypasses the ladder (recorded, not forbidden). `canary/<slug>`
  side-channels are per-session dead ends. **The serving rung is
  `edge`** (Paul's ruling, 2026-08-19): for an app with no production
  flag, edge *is* production, and climbing to `beta`/`stable` records
  human sign-off rather than gating what anyone can see. Instances can
  be wired to follow other channels — `carlos channels` shows what
  actually serves — but edge-serves is the family story.
- **Production flag** — a flagged app gets bake windows on promotion and
  console passkey step-up for the sensitive moves; unflagged apps
  promote freely. Ceremony is opt-in per app, not per channel.
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
| `carlos ship` | Publish an immutable release (`-kind binary\|static`, `-version`, `-notes`) |
| `carlos promote` | Move a version up the ladder (`-hotfix` to bypass, recorded) |
| `carlos deploy` | ship + promote + watch `X-Carlos-Version` until live — the one-command release; zero-arg with a saved project config |
| `carlos rollback` | Point a channel back at an earlier version |
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

- **`carlos deploy` is the release motion**: ship, promote to the channel
  the app's instances follow, then watch the URL until the header reports
  the shipped build. "Held for bake" on a production app is the system
  working, not failing.
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
