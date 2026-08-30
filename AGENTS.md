# 🤖 AGENTS.md

Working notes for anyone (human or agent) changing this repo.

- This repo is a Claude Code plugin marketplace (`.claude-plugin/`) whose
  root is the single plugin, `carlos`. Skills live under `skills/<name>/`.
- **AI authorship is always marked** (🤖 visible before the heading it
  covers, cascading; a person emoji certifies human text an LLM must not
  rewrite). Same rule as the rest of the family — see
  [carlosframework.com](https://carlosframework.com).
- The skill's claims trace to the CARLOS source applications. If you change
  a claim, check it against the apps, not against the previous copy.
  Never add secrets, hostnames, account ids, or links to the private
  source repos.
- Skill edits follow the skill-TDD rule: verify with a fresh-agent
  retrieval test before publishing, and bump the plugin `version` in
  `.claude-plugin/plugin.json` when a skill changes.
