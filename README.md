# 🤖 CARLOS skills

Claude Code skills for building apps on **CARLOS** — *Cost-efficient,
Available, Replicated, Lightweight, Open, Secure* — the application
architecture described at [carlosframework.com](https://carlosframework.com).

Two paths, one family:

- **[getting-started](skills/getting-started/SKILL.md)** — the recipe:
  it opens by interviewing the operator (what they hope to achieve, how
  much of an existing app they will replace, how polished the first
  slice should be and with what personality, where to host), then
  builds a first slice that feels finished with rastrillo, and deploys
  it where they chose: Carloku ([carloku.com](https://carloku.com)), a
  self-hosted CARLOS Platform, their own boxes, a host they already use
  (Fly, Render, Railway) or a plain server.
- **[building-carlos-apps](skills/building-carlos-apps/SKILL.md)** — the
  menu. It starts with tenancy: what one instance of the app is (a person,
  a team, a customer, a repository, the whole app), how people reach it
  (one instance, a simple home, a hub, a blind home), the trust class of
  each workload (S to C), and the benefits each choice earns. Then the
  family's principles, the platform member surface, the rastrillo
  recipe, and the four decision axes — trust model (full, partial, or
  server-side, with each source app's recorded reasoning), app shape
  (server-rendered vs client-owned, "HTML-first or more like Woodstar"),
  hosting (Carloku, self-hosting the open-source platform, your own
  boxes, or a plain server), and identity.

Plus **[devbox](skills/devbox/SKILL.md)**, for agents on a shared development
machine running devbox mode: publish an owned, tailnet-only HTTPS link to
your dev server or a mockup folder with `carlos devbox share`, and remove
it when the review is done.

And **[delegate](skills/delegate/SKILL.md)** — the operator's shorthand
for decomposing a workload across capability-tiered subagents.

## Install

As a Claude Code plugin (recommended):

```
/plugin marketplace add carlosframework/skills
/plugin install carlos@carlos
```

The skills are then available as `carlos:getting-started`,
`carlos:building-carlos-apps`, `carlos:devbox`, and `carlos:delegate`.

Or plainly: copy a skill's directory into your app repo's
`.claude/skills/` (or `~/.claude/skills/` for all projects). Each is a
handful of Markdown files; nothing else is required.

Or with no install at all: the platform-facing skills are published for
agent discovery at
[carlosframework.com/.well-known/agent-skills/index.json](https://carlosframework.com/.well-known/agent-skills/index.json),
per the [Agent Skills Discovery
draft](https://github.com/cloudflare/agent-skills-discovery-rfc) — an
index of name, description, url and sha256 digest per skill, with each
`SKILL.md` (and its `references/`) served beside it. `delegate` is
plugin-only on purpose: it is operator tooling, not knowledge about
CARLOS.

The website repo vendors those files from here (`hack/sync-skills.mjs`
in `carlosframework/website`, byte-for-byte — the digests pin the served
bytes to this repo's). **When a published skill changes here, re-sync
and ship the website in the same batch**, or the served copy drifts
behind the plugin.

## Provenance

Distilled July 2026 from the prompt histories, repo instruction files and
code of the five source applications plus the deliberate adopters; refreshed
August 2026 for the platform era — the `carlos` CLI, Carloku hosting,
rastrillo v0.5.0, and the decisions recorded by Keymail, Kass, Woodstar,
and Tito along the way; refreshed October 2026 to lead with tenancy and
trust classes across the full gallery of app shapes, for platform v0.28
and rastrillo v0.28; refreshed November 2026 to interview the operator
first, ask where to host, treat JavaScript as progressive enhancement,
and catch up with platform v0.34 and rastrillo v0.31. Contains no secrets, hostnames of
boxes, account ids, or private links; verified with baseline and
retrieval tests on fresh agents before publishing. The values layer
underneath is [the eleven factors](https://11factor.org).
