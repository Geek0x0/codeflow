# codeflow

A Claude Code plugin that packages a three-phase development workflow with
per-phase model pinning, built entirely on top of the
[superpowers](https://github.com/obra/superpowers) plugin.

| Phase | Role | Model | What happens |
|-------|------|-------|--------------|
| 1 | Planner | Fable (main thread, or relayed via a Fable subagent) | `superpowers:brainstorming` → spec, `superpowers:writing-plans` → plan |
| 2 | Implementer | Codex (MCP) | `superpowers:subagent-driven-development`'s structure, Codex executes each task, strict TDD |
| 3 | Reviewer | Fable (subagent) | `superpowers:requesting-code-review` + automatic fix loop (Codex fixes, 3-round cap) |

## Requirements

- Claude Code with plugin support.
- The **superpowers** plugin installed and enabled. codeflow delegates all
  process logic to superpowers skills and stops with an error if they are
  missing. Install it with:

  ```
  /plugin marketplace add obra/superpowers-marketplace
  /plugin install superpowers@superpowers-marketplace
  ```
- Access to the Fable model, and a working Codex MCP connection for
  implementation (`mcp__codex__codex` / `mcp__codex__codex-reply`).

## Install

This repo is itself a single-plugin marketplace. Once it is pushed to a
git host (e.g. `github.com/<you>/codeflow`):

```
/plugin marketplace add <you>/codeflow
/plugin install codeflow@codeflow-marketplace
```

GitHub accepts the `owner/repo` shorthand; for other hosts or private
repos use the full git URL (HTTPS or SSH — resolved with your local git
credentials). To pick up a new version later, run
`/plugin marketplace update codeflow-marketplace`.

For local development, load the repo as a local plugin directory (e.g.
`claude --plugin-dir /path/to/codeflow`), or add the checkout as a local
marketplace: `/plugin marketplace add /path/to/codeflow`.

## Usage

### `/codeflow:run [feature description]`

The full workflow. Blocking gates along the way:

1. **Adaptive Phase 1** — if the session model is Fable, planning runs
   inline on the main thread, same as before. If it isn't, codeflow relays:
   it repeatedly dispatches a pinned-Fable `planner` subagent, shows you
   each question the subagent asks, and sends your answer back — so
   planning still happens on Fable without you having to `/model fable`
   first. (Plugins cannot switch the *session's* model programmatically;
   the relay works around that by pinning a subagent's model instead —
   the same structural mechanism Phases 2–3 already use via agent
   frontmatter.) Tip: in projects where you use codeflow regularly, set
   `{ "model": "claude-fable-5" }` in the project's `.claude/settings.json`
   so sessions start on Fable and Phase 1 always runs inline.
2. **Spec approval** — brainstorming writes the spec; you review it.
3. **Plan approval** — writing-plans writes the plan; you review it.
4. **Branch gate** — before any code is written, you confirm the development
   branch (new `feature/<topic>` branch, current branch, custom name, or a
   worktree).
5. **Final review** — a Fable reviewer subagent audits the whole diff
   against the spec; valid CRITICAL/HIGH findings are fixed via Codex and
   re-reviewed, up to 3 rounds; MEDIUM/LOW findings are reported to you.

### `/codeflow:implement [plan-path]`

Resume at Phase 2 from an existing plan (after a dead session or context
compaction). Finds the first unchecked task, confirms the branch, executes,
then runs the Phase 3 review. No Fable guard: the orchestrator writes no
code — Codex executes tasks, and the Fable reviewer subagent is pinned
regardless of the session model.

### `/codeflow:review [base-ref]`

Phase 3 only: final review of the current branch (default base:
`git merge-base HEAD main`), including the fix loop.

## Hard rules baked into every command and agent

- **Commit only, never push.** No `git push` of any form, no PR-creation
  that pushes. Pushing is always left to you, manually.
- **Test coverage is first-class.** Every Codex implementation dispatch
  follows strict TDD; where coverage tooling exists the floor is 80% on
  touched code; the final reviewer treats missing/weak tests for new
  behavior as a HIGH finding.

## Model pinning notes

Agent frontmatter uses the model alias `fable` (in `agents/reviewer.md` and
`agents/planner.md`). If your Claude Code version does not recognize the
alias, edit those two files and replace it with a full model ID (e.g.
`claude-fable-5`).

Agent types are referenced as `codeflow:reviewer` / `codeflow:planner`;
some Claude Code versions register plugin agents under bare names
(`reviewer` / `planner`) — the commands mention both.

Codex's model (`gpt-5.6-sol`) is not a plugin agent pin — it's specified
directly on every `mcp__codex__codex` dispatch call in `commands/*.md`, per
this project's own Codex-usage conventions.

## Layout

```
codeflow/
├── .claude-plugin/
│   ├── plugin.json              # manifest
│   └── marketplace.json         # self-hosted single-plugin marketplace
├── commands/
│   ├── run.md                   # /codeflow:run
│   ├── implement.md             # /codeflow:implement
│   └── review.md                # /codeflow:review
├── agents/
│   ├── reviewer.md              # model: fable
│   └── planner.md               # model: fable — relay for non-Fable sessions
└── README.md
```
