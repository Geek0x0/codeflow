# codeflow Plugin Design

**Date:** 2026-08-07
**Status:** Approved pending user review

## Goal

A Claude Code plugin (`codeflow`) that packages a three-phase development
workflow with per-phase model pinning, built entirely on top of the
superpowers plugin:

| Phase | Role | Model | Superpowers skills used |
|-------|------|-------|-------------------------|
| 1 | Planner | Fable (main thread) | `brainstorming` → spec, `writing-plans` → plan |
| 2 | Implementer | Opus (subagents) | `subagent-driven-development` |
| 3 | Reviewer | Fable (subagent) | `requesting-code-review`, `receiving-code-review` |

## Key Architectural Decisions

1. **Hybrid topology.** Subagents cannot converse with the user, and
   `brainstorming` is inherently interactive. Therefore Phase 1 runs on the
   main thread (session model must be Fable, enforced by an instruction
   guard). Phases 2 and 3 pin models via subagent frontmatter, which is the
   only mechanism the platform actually enforces. (Extended in v0.2 below:
   when the session isn't Fable, Phase 1 relays through a pinned-Fable
   subagent instead of hard-stopping.)
2. **Main command plus recovery entries.** `/codeflow:run` drives all three
   phases in one session. `/codeflow:implement` re-enters at Phase 2 from an
   existing plan file (session died, context compacted). `/codeflow:review`
   triggers Phase 3 alone.
3. **Automatic review-fix loop.** Phase 3 findings are triaged with
   `receiving-code-review`, valid CRITICAL/HIGH findings are fixed by Opus
   implementer subagents, then re-reviewed. Hard cap: 3 rounds, then stop and
   report to the user.
4. **Thin wrappers, no forks.** Commands orchestrate and substitute agent
   types; they do not restate or modify superpowers skill content. Agent
   files exist solely to pin models; their prompts delegate to superpowers
   conventions.

## Plugin Structure

```
codeflow/
├── .claude-plugin/
│   └── plugin.json          # name: codeflow, version, description
├── commands/
│   ├── run.md               # /codeflow:run — full three-phase workflow
│   ├── implement.md         # /codeflow:implement [plan-path] — resume at Phase 2
│   └── review.md            # /codeflow:review — Phase 3 only
├── agents/
│   ├── implementer.md       # model: opus — executes one plan task
│   └── reviewer.md          # model: fable — whole-feature final review
└── README.md                # install steps, superpowers dependency, model guard notes
```

There is deliberately **no planner agent file**: the planner is the main
thread itself.

## Workflow Detail

### Phase 0 — Guards (start of `/codeflow:run`)

- Main thread self-checks its model is Fable. If not: stop, tell the user to
  run `/model fable` and re-invoke. (The platform offers no programmatic way
  for a plugin to switch or enforce the main-thread model; an instruction
  guard is the honest best effort. Documented in README.)
- Check superpowers skills are available (e.g. `superpowers:brainstorming`
  appears in the skill list). If missing: stop with a clear dependency error.

### Phase 1 — Planner (main thread, Fable)

- Invoke `superpowers:brainstorming`: interactive Q&A → design doc written to
  `docs/superpowers/specs/` and committed → user approves.
- Invoke `superpowers:writing-plans`: implementation plan written to
  `docs/superpowers/plans/` and committed → user approves.
- Superpowers flows are used as-is, no modification.

### Phase 2 — Implementer (Opus subagents)

- **Branch gate (mandatory):** before any implementation work, confirm the
  development branch with the user via a question with options:
  - create new branch from current HEAD (default suggestion:
    `feature/<topic>` derived from the spec topic),
  - use the current branch as-is,
  - user-specified branch name,
  - optionally isolate in a worktree via `superpowers:using-git-worktrees`.
  Implementation MUST NOT start until the user has confirmed.
- Run `superpowers:subagent-driven-development` with one substitution: every
  per-task implementation subagent is dispatched as `codeflow:implementer`
  (model: opus) instead of a general-purpose agent.
- SDD's own per-task spec-review and code-review subagents stay at their
  defaults (they inherit the session model, Fable). Only the coding agents
  and the final reviewer have pinned models.
- **Test coverage requirement:** the implementer agent enforces TDD per
  `superpowers:test-driven-development` (test first, watch it fail,
  implement, watch it pass) for every task. Where the project has coverage
  tooling configured, the 80% minimum applies; where it does not, behavioral
  coverage is judged in review.
- Per-task commits happen as SDD prescribes. **Push is prohibited** (see
  global constraints).

### Phase 3 — Reviewer (Fable subagent) + fix loop

- Follow `superpowers:requesting-code-review`, but dispatch
  `codeflow:reviewer` (model: fable) instead of the default code-reviewer
  agent. Review scope: full diff from the confirmed branch point to HEAD,
  checked against the spec and plan.
- **Test coverage is a first-class review dimension:** missing or weak tests
  for new behavior are at least a HIGH finding.
- On findings: triage with `superpowers:receiving-code-review` (verify, don't
  blindly comply). Valid CRITICAL/HIGH findings → dispatch
  `codeflow:implementer` (Opus) to fix → re-run the final review.
- Loop until no CRITICAL/HIGH findings remain. MEDIUM/LOW findings are
  reported to the user for decision, not auto-fixed.
- Safety valve: after 3 rounds with CRITICAL/HIGH still present, stop and
  report to the user.
- When clean: run `superpowers:finishing-a-development-branch` with its
  push/PR options removed — allowed outcomes are local merge, keep branch, or
  discard; anything requiring push is reported to the user to do manually.

## Global Constraints

- **Commit only, never push.** Every command and both agent prompts carry a
  hard prohibition on `git push` (any form, including `-u`, tags, and
  PR-creation commands that push). Commits follow the conventional format.
- Plugin depends on the superpowers plugin being installed; this is declared
  in README and checked at Phase 0.

## Component Specifications

### plugin.json

Minimal manifest: `name: codeflow`, `version: 0.1.0`, one-line description.

### commands/run.md

Frontmatter: `description`. Body: Phase 0 guards, then Phases 1–3 as above,
with explicit gates (user approval after spec, after plan, branch
confirmation, final report). States the no-push constraint. Instructs the
main thread to invoke superpowers skills via the Skill tool and to dispatch
`codeflow:implementer` / `codeflow:reviewer` agent types where specified.
Phase 0/1 additionally route to a relay loop dispatching `codeflow:planner`
when the session model isn't Fable (see v0.2 section below).

### commands/implement.md

Frontmatter: `description`, `argument-hint: [plan-path]`. Body: superpowers
dependency guard; resolve plan file (argument, or list candidates from
`docs/plans/` and confirm with user); branch gate; then Phase 2 → Phase 3.
No Fable main-thread guard: the orchestrator writes no code, so the main
model does not matter here.

### commands/review.md

Frontmatter: `description`. Body: superpowers dependency guard; determine
review base (ask user, defaulting to merge-base with main); Phase 3 only,
including the fix loop and its 3-round cap.

### agents/implementer.md

Frontmatter: `name: implementer`, `description` (executes exactly one task
from a codeflow plan), `model: opus`, tools: Read, Write, Edit, Bash, Grep,
Glob. Body: accept the per-task prompt from SDD verbatim; follow TDD strictly
(RED before GREEN, refactor after); run the task's verification commands;
commit the task's changes; never push; report changed files, test results,
and blockers back.

### agents/reviewer.md

Frontmatter: `name: reviewer`, `description` (whole-feature final review
against spec and plan), `model: fable`, tools: Read, Grep, Glob, Bash
(read-only usage by instruction). Body: follow the review template
conventions from `superpowers:requesting-code-review` (what was built, spec
compliance, severity buckets CRITICAL/HIGH/MEDIUM/LOW); treat absent or weak
tests for new behavior as HIGH; verify claimed test results by running the
test suite; never modify files; never push.

### agents/planner.md (v0.2)

Frontmatter: `name: planner`, `description` (relay-dispatched Fable planner
for codeflow Phase 1), `model: fable`, tools: Read, Write, Edit, Bash, Grep,
Glob (no Skill tool — see v0.2 section below for why). Body: per-dispatch
protocol (read the transcript, never re-ask an answered question, emit
exactly one of `QUESTION:`/`SPEC_READY:`/`PLAN_READY:`); condensed,
independently-authored spec and plan disciplines (not vendored skill text);
never push.

## Error Handling & Known Risks

| Risk | Mitigation |
|------|------------|
| Main-thread model cannot be programmatically enforced | Instruction guard at Phase 0 + README documentation |
| `model: fable` / `model: opus` alias not recognized by an older Claude Code | Use aliases; README documents falling back to full model IDs (`claude-fable-5`, `claude-opus-5`) |
| Exact subagent namespace (`codeflow:implementer` vs bare name) may vary | Pin down during smoke test; commands reference the plugin-qualified name |
| superpowers not installed | Explicit Phase 0 check, hard stop with install instructions |
| Review-fix infinite loop | 3-round cap, then report |
| Session dies mid-implementation | Plan file on disk + SDD per-task checkboxes = natural checkpoint; resume with `/codeflow:implement` |
| Relay loop never terminates (v0.2) | 30-round cap per stage, then stop and report |
| `planner` subagent returns malformed output (v0.2) | Orchestrating session shows the raw text to the user and asks how to proceed instead of guessing |

## Verification Plan

The plugin is pure Markdown/JSON — no unit-testable code. Verification:

1. **Static:** plugin.json validates against the official plugin schema;
   command/agent frontmatter fields are well-formed.
2. **Smoke test:** install the plugin locally, run `/codeflow:run` on a toy
   requirement end-to-end on a Fable session (inline Phase 1) and again on a
   non-Fable session (relayed Phase 1 — confirm each `QUESTION:` reaches the
   human, `SPEC_READY:`/`PLAN_READY:` are verified before proceeding, and the
   30-round cap is sane). Confirm: each gate stops and waits (spec approval,
   plan approval, branch confirmation); implementer subagents run as Opus;
   the final reviewer runs as Fable; the fix loop triggers on a seeded
   defect; no `git push` occurs anywhere; recovery entries
   `/codeflow:implement` and `/codeflow:review` work from a fresh session.

## v0.2 — Relay Planner (adaptive Phase 1)

**New file:** `agents/planner.md` (`model: fable`; tools: Read, Write, Edit,
Bash, Grep, Glob — no Skill tool, see below).

**Problem:** the v0.1 Phase 0 model guard is a hard stop — if the session
model isn't Fable when `/codeflow:run` is invoked, the user must manually
`/model fable` and re-run. Phase 1 can't simply become a subagent outright:
a dispatched subagent runs to completion and returns one final report, so
it cannot hold `brainstorming`'s turn-by-turn conversation with the human
directly (see Key Architectural Decision 1).

**Solution:** Phase 0 routes instead of stopping. Session model is Fable →
Phase 1 runs inline exactly as in v0.1 (with a mid-phase drift check: if
the model changes before the spec or plan commit, fall through to the
relay path instead of committing). Session model is anything else → Phase 1
runs as a **relay loop**: the orchestrating session (on whatever model)
repeatedly dispatches a fresh `codeflow:planner` subagent (bare-name
fallback `planner`), pinned to Fable regardless of the session model —
mirroring how Phases 2–3 already pin Opus/Fable via agent frontmatter.
Each dispatch performs exactly one relay round and returns one of three
signals:

- `QUESTION: <text>` — the session relays this to the human verbatim, waits
  for the reply, appends both to a transcript file, and dispatches again.
- `SPEC_READY: <path>` / `PLAN_READY: <path>` — the stage is done; the
  session verifies the file exists and is committed before proceeding.

**Why stateless re-dispatch, not subagent continuation:** some harnesses
support resuming a previously-dispatched subagent with its memory intact
(seen in this project's own build session). codeflow does not rely on that
— it isn't guaranteed across every Claude Code environment the plugin might
run in. Each dispatch is instead a fresh `planner` instance that reads a
transcript file (every question asked and answered so far) and continues
from there — this only requires basic subagent dispatch, which every
target environment has.

**Why `agents/planner.md` inlines the discipline instead of invoking
`superpowers:brainstorming`/`writing-plans` via the Skill tool:** matches
the existing precedent in `agents/implementer.md` (inlines TDD) and
`agents/reviewer.md` (inlines the review template) instead of adding an
unverified new capability — a subagent invoking the Skill tool — to the
plugin's tool surface. This is an independently-authored, condensed
discipline for the same job, not a copy of the skill files, so it does not
violate the "no vendoring" global constraint below.

**Scope-downs versus the inline path:** no visual companion offer (the
subagent has no browser tool); writing-plans' "which execution approach"
question is skipped (codeflow always runs subagent-driven).

**Safety valve:** each relay stage (spec, plan) caps at 30 rounds; exceeding
it stops and reports to the user, mirroring the Phase 3 fix-loop's 3-round
cap.

**Cost tradeoff:** each human answer costs one additional Fable dispatch.
Projects that use codeflow often can skip the relay entirely by pinning the
project to Fable in `.claude/settings.json` (`{ "model": "claude-fable-5" }`,
documented in README).

## Out of Scope

- No hooks, no MCP servers, no marketplace publishing setup.
- No per-task reviewer model pinning (SDD defaults suffice).
- No programmatic model switching of the **main thread** (platform does not
  support it) — v0.2's relay works around this for Phase 1 specifically by
  pinning a subagent's model instead, not by switching the session itself.
- No CI integration.
