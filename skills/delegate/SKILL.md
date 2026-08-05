---
name: delegate
description: Use when handed a workload that should be spread across subagents — "divvy this up", multi-part implementation, parallel research, or any task big enough that one context window shouldn't hold it all. Decomposes the work, picks the cheapest capable model per piece, writes precise briefs, and pairs anything risky with an adversarial reviewer.
---

# 🤖 Delegate

## Overview

The operator's shorthand: **"figure out the best way to divvy up this
workload, and hand it to appropriate capability subagents."** This skill is
that sentence made mechanical. It exists because the two failure modes of
delegation are both silent: a workload done serially in one context that
should have fanned out, and an expensive model doing work a precise brief
would have made trivial.

The governing law (Paul, 2026-08-02): **a precise brief beats a capable
model.** Capability tiers exist for the residue a brief cannot pin down.

## Step 1 — Decompose before you dispatch

Split the workload into pieces with named outputs. For each piece, answer
two questions before choosing anything else:

- **Does it share mutable state with another piece?** Two agents committing
  to one worktree or writing one file is how git states get contested and
  work gets clobbered. Shared state ⇒ sequential pipeline. Disjoint state
  (different repos, different dirs, read-only research) ⇒ parallel fan-out,
  dispatched in one batch.
- **Does anything downstream depend on its interface?** If yes, pin the
  interface in the plan/brief FIRST so dependents can be briefed without
  waiting — then the only sequencing left is real data dependence.

If the decomposition needs judgment (unclear seams, tangled state), that
decomposition is itself a task — give it to a capable model before
dispatching anything.

## Step 2 — Pick the tier per piece, and name it

An omitted model silently inherits the session's (usually the most
expensive). **Always name the model.**

- **Haiku** — transcription-grade: the brief contains the exact code or
  text; single-file mechanical fixes; doc edits from dictation.
- **Sonnet** — implementation from a precise brief: 1–3 files, pinned
  interfaces, established conventions to follow, verification commands
  given.
- **Opus** — design judgment, multi-file integration, anything security- or
  production-touching; and the **reviewer** for work in those areas.
- **Fable** — where the tier changes the outcome, not the speed:
  implementation whose correctness rests on several interacting invariants
  at once (root-side, fleet-wide blast radius); whole-branch final reviews,
  where cross-cutting bugs live that no per-task review can see; and deep
  work that wants a fresh full-capability context window. It burns usage
  budget fastest — never spend it where a precise Sonnet brief or an Opus
  review already fits.

Two hedges worth paying for: a reviewer should not share the implementer's
blind spots (mix tiers or at least instances), and when the budget is
tight, spend the top tier on review rather than implementation — a caught
bug is worth more than a slightly better first draft.

## Step 3 — Write the brief

Every dispatch names, explicitly:

1. **Where** — absolute worktree/repo path, branch, current HEAD.
2. **What to read first** — the project CLAUDE.md, the spec/plan section
   (say "the task text is authoritative"), and the specific existing code
   to match style against.
3. **Exactly what to build** — pinned interfaces byte-for-byte where they
   matter, exact values, file lists. What NOT to touch, said out loud.
4. **Accumulated rulings** — anything a previous review round already paid
   for. Do not let a fresh agent re-learn a lesson the branch already
   learned; paste the ruling into the brief.
5. **The gate** — the exact build/test/format commands that must pass
   before commit, and any sandbox caveats.
6. **Commit discipline** — message style, co-author trailer, stage named
   files only, push or don't.
7. **The report format** — commit sha, files, test names and what each
   pins, deviations WITH reasons, gate output tail. Deviations from pinned
   text must be argued, not silent.

## Step 4 — Pair risky work with an adversarial review

Anything that grants authority, touches production paths, or parses hostile
input gets an independent adversarial review before the next dependent
piece lands. The reviewer's brief: assume malice, try to BREAK it, name
concrete failure scenarios, mutation-check the tests ("which mutations
survive the suite?"), and report Critical / Important / Minor with
file:line and a fix sketch — verdict `approve` or `fix-first`, no praise
padding.

Fix rounds go back to the **original implementer** (they have the context);
the orchestrator adjudicates findings it disagrees with rather than
blindly applying them, and records rulings where the next task's brief will
find them.

## Step 5 — Orchestrator's own conduct

- Never let two agents commit to one worktree concurrently. One pipeline
  per checkout; parallelism lives in disjoint checkouts or read-only work.
- Verify the gate yourself before shipping — with the binary you built,
  against the thing you changed. An implementer's green run is a claim;
  the orchestrator's is a fact.
- Relay results in your own words. The user never sees an agent's report.
- If an agent dies (usage limit, API error), the work is yours: do it
  in-session or re-dispatch, but never report it done on the strength of a
  dead agent's partial output.
