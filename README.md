# codeflow

A Claude Code plugin that packages a three-phase development workflow with
per-phase model pinning, built entirely on top of the
[superpowers](https://github.com/obra/superpowers) plugin.

| Phase | Role | Model | What happens |
|-------|------|-------|--------------|
| 1 | Planner | Fable (main thread) | `superpowers:brainstorming` → spec, `superpowers:writing-plans` → plan |
| 2 | Implementer | Opus (subagents) | `superpowers:subagent-driven-development`, one Opus subagent per task, strict TDD |
| 3 | Reviewer | Fable (subagent) | `superpowers:requesting-code-review` + automatic fix loop (Opus fixes, 3-round cap) |

## Requirements

- Claude Code with plugin support.
- The **superpowers** plugin installed and enabled. codeflow delegates all
  process logic to superpowers skills and stops with an error if they are
  missing.
- Access to the Fable and Opus models.

## Install

From a marketplace that carries this plugin:

```
/plugin install codeflow@<marketplace-name>
```

For local development, load the repo as a local plugin directory (see
`claude --help` for the plugin dev flag in your version, e.g.
`claude --plugin-dir /path/to/codeflow`), or add the repo to a local
marketplace of your own.

## Usage

### `/codeflow:run [feature description]`

The full workflow. Blocking gates along the way:

1. **Model guard** — Phase 1 runs on the main thread, so the session model
   must be Fable. The command checks and stops if it is not; run
   `/model fable` and re-invoke. (Plugins cannot switch the main-thread
   model programmatically; this instruction guard is the enforced-by-text
   best effort. Subagent models in Phases 2–3 ARE structurally pinned via
   agent frontmatter.)
2. **Spec approval** — brainstorming writes the spec; you review it.
3. **Plan approval** — writing-plans writes the plan; you review it.
4. **Branch gate** — before any code is written, you confirm the development
   branch (new `feature/<topic>` branch, current branch, custom name, or a
   worktree).
5. **Final review** — a Fable reviewer subagent audits the whole diff
   against the spec; valid CRITICAL/HIGH findings are fixed by Opus
   subagents and re-reviewed, up to 3 rounds; MEDIUM/LOW findings are
   reported to you.

### `/codeflow:implement [plan-path]`

Resume at Phase 2 from an existing plan (after a dead session or context
compaction). Finds the first unchecked task, confirms the branch, executes,
then runs the Phase 3 review. No Fable guard: the orchestrator writes no
code, and subagent models are pinned regardless of the session model.

### `/codeflow:review [base-ref]`

Phase 3 only: final review of the current branch (default base:
`git merge-base HEAD main`), including the fix loop.

## Hard rules baked into every command and agent

- **Commit only, never push.** No `git push` of any form, no PR-creation
  that pushes. Pushing is always left to you, manually.
- **Test coverage is first-class.** Implementer subagents follow strict TDD;
  where coverage tooling exists the floor is 80% on touched code; the final
  reviewer treats missing/weak tests for new behavior as a HIGH finding.

## Model pinning notes

Agent frontmatter uses the model aliases `opus` and `fable`. If your Claude
Code version does not recognize an alias, edit `agents/*.md` and replace the
alias with a full model ID (e.g. `claude-opus-5`, `claude-fable-5`).

Agent types are referenced as `codeflow:implementer` / `codeflow:reviewer`;
some Claude Code versions register plugin agents under bare names
(`implementer` / `reviewer`) — the commands mention both.

## Layout

```
codeflow/
├── .claude-plugin/plugin.json   # manifest
├── commands/
│   ├── run.md                   # /codeflow:run
│   ├── implement.md             # /codeflow:implement
│   └── review.md                # /codeflow:review
├── agents/
│   ├── implementer.md           # model: opus
│   └── reviewer.md              # model: fable
└── README.md
```
