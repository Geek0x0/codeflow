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
   only mechanism the platform actually enforces.
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
  `docs/plans/` and committed → user approves.
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

## Error Handling & Known Risks

| Risk | Mitigation |
|------|------------|
| Main-thread model cannot be programmatically enforced | Instruction guard at Phase 0 + README documentation |
| `model: fable` / `model: opus` alias not recognized by an older Claude Code | Use aliases; README documents falling back to full model IDs (`claude-fable-5`, `claude-opus-5`) |
| Exact subagent namespace (`codeflow:implementer` vs bare name) may vary | Pin down during smoke test; commands reference the plugin-qualified name |
| superpowers not installed | Explicit Phase 0 check, hard stop with install instructions |
| Review-fix infinite loop | 3-round cap, then report |
| Session dies mid-implementation | Plan file on disk + SDD per-task checkboxes = natural checkpoint; resume with `/codeflow:implement` |

## Verification Plan

The plugin is pure Markdown/JSON — no unit-testable code. Verification:

1. **Static:** plugin.json validates against the official plugin schema;
   command/agent frontmatter fields are well-formed.
2. **Smoke test:** install the plugin locally, run `/codeflow:run` on a toy
   requirement end-to-end. Confirm: Fable guard fires when the session model
   is wrong; each gate stops and waits (spec approval, plan approval, branch
   confirmation); implementer subagents run as Opus; the final reviewer runs
   as Fable; the fix loop triggers on a seeded defect; no `git push` occurs
   anywhere; recovery entries `/codeflow:implement` and `/codeflow:review`
   work from a fresh session.

## Out of Scope

- No hooks, no MCP servers, no marketplace publishing setup.
- No per-task reviewer model pinning (SDD defaults suffice).
- No programmatic model switching (platform does not support it).
- No CI integration.
