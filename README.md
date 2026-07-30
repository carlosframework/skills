# 🤖 CARLOS skills

Claude Code skills for building apps on **CARLOS** — *Cost-efficient,
Available, Replicated, Lightweight, Open, Secure* — the application
architecture described at [carlosframework.com](https://carlosframework.com).

One skill so far: **[building-carlos-apps](skills/building-carlos-apps/SKILL.md)**
— the guiding principles, technical preferences, infrastructure blueprint,
and working conventions of the CARLOS family, distilled from the apps the
architecture was extracted from, so a new app can be built on the model
without reading their source.

## Install

As a Claude Code plugin (recommended):

```
/plugin marketplace add carlosframework/skills
/plugin install carlos@carlos
```

The skill is then available as `carlos:building-carlos-apps`.

Or plainly: copy `skills/building-carlos-apps/` into your app repo's
`.claude/skills/` (or `~/.claude/skills/` for all projects). It's three
Markdown files; nothing else is required.

## Provenance

Distilled in July 2026 from the prompt histories, CLAUDE.md files and code
of the five source applications plus one deliberate adopter. Contains no
secrets, hostnames, or private links; verified with baseline and retrieval
tests on fresh agents before publishing. The values layer underneath is
[the eleven factors](https://11factor.org).
