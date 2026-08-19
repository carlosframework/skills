# 🤖 CARLOS skills

Claude Code skills for building apps on **CARLOS** — *Cost-efficient,
Available, Replicated, Lightweight, Open, Secure* — the application
architecture described at [carlosframework.com](https://carlosframework.com).

Two paths, one family:

- **[getting-started](skills/getting-started/SKILL.md)** — the recipe:
  empty directory to a live URL on the hosted platform
  ([carloku.com](https://carloku.com)), every decision pre-made — rastrillo,
  server-rendered HTML, `carlos deploy`. For a first app with minimum
  further guidance.
- **[building-carlos-apps](skills/building-carlos-apps/SKILL.md)** — the
  menu: the family's principles, the platform member surface, the rastrillo
  recipe, and the four decision axes — trust model (full, partial, or
  server-side, with each source app's recorded reasoning), app shape
  (server-rendered vs client-owned, "HTML-first or more like Woodstar"),
  hosting (hosted, your own boxes, or self-hosting the platform), and
  identity.

Plus **[delegate](skills/delegate/SKILL.md)** — the operator's shorthand
for decomposing a workload across capability-tiered subagents.

## Install

As a Claude Code plugin (recommended):

```
/plugin marketplace add carlosframework/skills
/plugin install carlos@carlos
```

The skills are then available as `carlos:getting-started`,
`carlos:building-carlos-apps`, and `carlos:delegate`.

Or plainly: copy a skill's directory into your app repo's
`.claude/skills/` (or `~/.claude/skills/` for all projects). Each is a
handful of Markdown files; nothing else is required.

## Provenance

Distilled July 2026 from the prompt histories, CLAUDE.md files and code of
the five source applications plus the deliberate adopters; refreshed
August 2026 for the platform era — the `carlos` CLI, Carloku hosting,
rastrillo v0.5.0, and the decisions recorded by Keymail, Kass, Woodstar,
Tito, and Seapointish along the way. Contains no secrets, hostnames of
boxes, account ids, or private links; verified with baseline and
retrieval tests on fresh agents before publishing. The values layer
underneath is [the eleven factors](https://11factor.org).
