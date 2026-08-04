---
name: building-carlos-apps
description: Use when building a new app on the CARLOS architecture (Cost-efficient, Available, Replicated, Lightweight, Open, Secure) or with rastrillo (the CARLOS web framework), or bringing an existing app onto it — the model extracted from Eleven Messenger, Keymail, Woodstar, Slopbox and Kass, adopted by Tito — and you need the family's principles, stack, infrastructure shape, and working conventions without reading the source apps.
---

# 🤖 Building CARLOS apps

## Overview

CARLOS is an application architecture: **one static Go binary, one
fully-isolated instance per account (own process, own SQLite file, own unix
socket, own hostname), many instances on one small box behind a small
router**, with every database continuously replicated to object storage.
Everything below is distilled from the prompt histories, CLAUDE.md files and
code of the apps it was extracted from. The house style is to state rules as
laws, date and attribute decisions, and make rules mechanical rather than
remembered.

## When to use

- Starting a new product that should join the family.
- Porting an existing app onto the model (Tito's path — see "Adopting" below).
- Any question of the form "what would a CARLOS app do here?" — stack,
  hosting, testing, deploys, process.

Not for: contributing to one of the existing apps (read that repo's CLAUDE.md
instead — it always wins over this skill).

## Where this sits now (2026-08-04)

Two more pieces of the family exist as real infrastructure now, not just
conventions to remember, and this skill should be read alongside them:

- **The platform** (`carlosframework/platform`, live) runs the router,
  registry, hibernation, and Litestream replication described under "The
  carlos core" and "Replication" in blueprint.md. An app deployed on it
  does not hand-roll `internal/carlos` or a litestream config — those
  sections are the reference for self-hosting outside the platform, or
  for understanding what it's doing on an app's behalf.
- **Rastrillo** (`carlosframework/rastrillo`, **shipped** — v1 walking
  skeleton plus the manifest system; the
  [design doc](https://github.com/carlosframework/platform/blob/main/docs/superpowers/specs/2026-08-01-carlos-framework-design.md)
  remains the map of what's built vs deferred) is the Go web framework
  that mechanizes a further slice of blueprint.md: the SQLite
  pragma/migration rules, the JS module-line cap, filesystem routing,
  and — via manifests — a declared resource's whole store, screens, and
  locale keys become things `rastrillo generate` and `rastrillo.Serve`
  enforce and produce, not things kept correct by hand. Marked inline
  in blueprint.md as **Automatic on rastrillo**.
  **Building with it, read
  [references/rastrillo.md](references/rastrillo.md)** — the install
  commands, CLI verbs, layout, and a worked manifest; written to be
  sufficient on its own, since the framework postdates most models'
  training data.

Building a **new** app: use rastrillo (references/rastrillo.md for the
how), and read the rest of this skill mainly for what a framework can't
enforce — the one rule below, dated settled decisions, the
git/PR/canary workflow, and "Common mistakes". Porting an existing app,
or building without rastrillo for a specific reason: blueprint.md in
full still applies, hand-rolled.

## The one rule comes first

Every app opens its CLAUDE.md with a single load-bearing rule and derives
everything from it. Write yours before writing code (the first commit of Kass
is literally "Write down the architecture before writing any code").

The family default is server-blindness: **"If the server is compromised, the
attacker gets nothing."** Plaintext or a usable private key in SQLite is a bug
— and because Litestream ships the whole file to S3, *a plaintext column is a
plaintext leak*. Enforce it with a test that greps the raw SQLite bytes (and
S3 objects) for plaintext.

The exception proves it is a decision, not a dogma: Tito deliberately chose
server-side trust because a six-person support team must be able to
`SELECT email FROM auth_users` ("Vicky's rule": every decision is weighed by
the support load it creates — prefer the boring thing that can't page
anyone). Pick your trust model explicitly, on day one, and record why.

## Guiding principles

1. **Intent before code.** CLAUDE.md is written first and is binding. Design
   docs precede big builds; the doc's decisions are taken — if one proves
   wrong, stop and flag it in the PR rather than redesigning.
2. **Decisions are dated, attributed, and settled.** "Paul's call,
   2026-07-26" next to every ruling, rejected alternatives recorded beside
   it, under a heading like "Settled decisions (don't relitigate casually)".
3. **Built to outlive AI.** Human legibility beats AI convenience. Never add
   machinery only an LLM can drive; operational knowledge lives in scripts
   and docs in the repo, never solely in prompts or agent memory. "No LLM
   cookbooks": ops run through a command (`ops deploy`, `ops doctor`), never
   reconstructed from prose — a script drifts loudly, a document drifts
   silently.
4. **Rules are mechanical, not remembered.** The module-size cap is a test.
   The "only main deploys to shared hosts" rule is the deploy script building
   `origin/main` in a throwaway worktree. Generated configs are marked
   GENERATED and regenerated on a timer, never hand-edited.
5. **Blast radius is the unit of design.** One process + one DB per account:
   a panic takes down one person, and systemd restarts it.
6. **Defer until earned, with the trigger written down.** Hibernation waits
   for idle-cost numbers; mesh waits for a second box; wildcard DNS-01 certs
   wait for ~50 new certs/week. Keep a "Deliberately not built yet" section.
7. **Don't diverge gratuitously from the family.** Shared shape is what
   makes components extractable; a local improvement to shared code is a debt
   to upstream, noted as such.
8. **Honesty over polish.** Publish trade-offs plainly ("Honest trade-offs —
   read before trusting it"), never invent an escrow or overclaim maturity,
   and don't let the UI lie (estimates that "neither flap nor lie").
9. **Every deviation from the one rule is enumerated**, justified, and
   published — never hidden.

## The eleven factors

CARLOS is the architecture; the values under it are **the eleven factors**
([11factor.org](https://11factor.org) is the canonical text — link to it,
don't copy it). Sibling repos cite factors **by number**, so know the names:

I. Trust no one · II. Let kids play · III. Encrypt everything ·
IV. Great design is for everyone · V. Intent is the system ·
VI. Built by humanity, owned by humanity · VII. Self-hosting is a right ·
VIII. Many small things · IX. Inefficient builds efficient ·
X. Humans come first · XI. Centralised infrastructure is glue.

The ones the family invokes operationally, and what they mean in practice
here:

- **III — Encrypt everything** → the one rule: server-blind by default.
  A deliberate deviation (Tito's server-side trust) is legitimate under
  the family's deviation rule — enumerated, justified, published — not a
  violation.
- **V — Intent is the system** → CLAUDE.md and design docs before code;
  "the code should have to argue with something." (The argument goes both
  ways: a doc decision that proves wrong in practice is stopped and
  flagged, not silently obeyed.)
- **VI — Built by humanity, owned by humanity** → open source; private
  until it works is allowed, but "factor VI is not optional, only
  deferred."
- **VII — Self-hosting is a right** → self-hosting is first-class, never a
  degraded tier; hosted convenience is the product, never the software.
- **VIII — Many small things** → instance-per-account, scale by adding
  instances, not replicas of one.
- **X — Humans come first** → AI is "a guest with a name tag": opt-in,
  never required, always disclosed (🤖 marking), and every path works with
  the AI switched off.

So when a sibling CLAUDE.md says "factor X applied" or "factor VI is not
optional", that's what it means.

## Quick reference — the shape

| Concern | The CARLOS answer |
|---|---|
| Language | Go, one static binary, `CGO_ENABLED=0`, thin `main.go` dispatch |
| Dependencies | Stdlib first; "adding a dependency is a decision, not a default"; hand-roll small clients (SigV4 is ~a page of HMACs) over SDKs |
| Storage | `modernc.org/sqlite`, WAL, one DB per instance, additive-only migrations |
| Quantities | Integer cents, integer grams — "a float never touches an amount" |
| Frontend | Server-rendered HTML first; vanilla ES modules, no framework, no bundler, no build step; `go:embed`; vendored VanJS if a view needs reactivity |
| JS discipline | 300-line module cap enforced by test, ratchet-down only — "we are not doing shell.js again" |
| Routing | `internal/carlos`: SQLite registry (host → unix socket) + TLS router; the route table IS the ACME allowlist |
| Replication | Litestream WAL → S3 for every DB; restore drills on a timer |
| Hosting | One or two tiny ARM boxes (t4g.nano/micro or one Hetzner box); no containers, no k8s, no LB, no managed DB |
| IaC | OpenTofu (`tofu`, never Terraform) |
| Deploys | Build exact `origin/main` in a throwaway worktree, scp, `systemctl restart`, verify `/api/version` == sha on every socket |
| Identity | Passkeys (WebAuthn, PRF extension) or magic link + TOTP; tokens stored hashed, never logged |
| Sign-in span | One content-blind "home" vault spans all of a person's instances — port Eleven's home server, don't reinvent it (server-trust apps: identity beside the router instead) |
| UI stance | Hide the machinery: no hostnames, keys, or crypto vocabulary in the default flow — "no nerdspeak" |
| Process | Worktree per session; branch → PR → squash-merge; canary per session, review never on localhost |
| Authorship | 🤖/👨 markers, `Co-Authored-By: Claude …` trailers, published prompt + carbon ledgers |

**Automatic on rastrillo (as shipped, 2026-08-04):** Storage, Quantities
(its `Money` kind, integer cents), and — via manifests — a declared
resource's whole CRUD surface: store, screens, locale keys. Designed for
rastrillo but not built yet: the JS-discipline line cap and the
ECDH/AES-GCM envelope half of Identity (those stay hand-kept for now —
see "Not built yet" in references/rastrillo.md). **Automatic on the platform, regardless of
framework:** Routing and Replication (see blueprint.md). Everything else
in the table — Hosting, IaC, Deploys, Sign-in span, UI stance, Process,
Authorship — is unchanged either way.

## Details

- **[references/rastrillo.md](references/rastrillo.md)** — building an
  app with rastrillo: install, CLI verbs (`new`/`generate`/`dev`), the
  scaffold layout, a worked manifest resource, ejection, migrations.
- **[references/blueprint.md](references/blueprint.md)** — the technical and
  infrastructure blueprint: the carlos core, storage rules and known SQLite
  fixes, crypto conventions, deploy and box setup, security defaults.
- **[references/process.md](references/process.md)** — working conventions:
  repo docs, git/PR workflow, canaries, the test layers, `/end-session`,
  ledgers and AI-authorship marking.

## Adopting CARLOS in an existing app (Tito's path)

1. Restate the model in your first commit and at the top of README and
   CLAUDE.md.
2. **Deploying onto `carlosframework/platform`: skip this step.** It
   provides the router, registry, and cert handling already — see "Where
   this sits now" above. **Self-hosting outside the platform instead:**
   take the core now into a package named `internal/carlos`, kept
   deliberately close to the source shape in blueprint.md's "The carlos
   core" section so it can be swapped for the platform later if you adopt
   it.
3. Adopt the conventions alongside (worktrees, PR playbook, canaries,
   additive migrations, zero-JS-first).
4. List every deferred component with its trigger; list every deliberate
   departure with its reason.
5. Fixes you make to shared-shape code are candidates to upstream — record
   them.

## Common mistakes

| Mistake | Reality |
|---|---|
| Adding a framework/bundler "just for this screen" | The no-build-step rule is load-bearing (auditability, longevity). One more ES module, one concern. |
| Growing the biggest file/package | A new concern gets a new small module or package — never more growth of the biggest one. |
| Destructive migration "to clean up" | Migrations are additive-only. New code over an old DB must always be safe. Never delete data to update. On rastrillo this is a `carlos vet` check, not just a rule to remember. |
| Trusting a backup that exists | "A backup you've never restored is a hope, not a backup." Restore-verify on a timer. |
| Reviewing on localhost | Review happens on a deployed canary, always. Shared hosts only ever run merged main. |
| Skipping the post-deploy browser check | A JS syntax error takes the whole client down and only a real engine sees it (the keymail rule). |
| Deploying a branch to a shared host | The deploy path must make this impossible, not discouraged. |
| Relitigating a settled decision in a drive-by | Settled decisions change with new facts, dated and attributed — not casually. |
