---
description: Resume codeflow at Phase 2 — execute an existing plan via Codex or DeepSeek, then run the Fable final review
argument-hint: [plan-path]
---

# codeflow: implement from an existing plan

Resume the codeflow workflow at Phase 2 using a plan that already exists on
disk. Use this after a session died or context was compacted mid-run.

## Global constraints (every phase, every subagent)

- **Commit only, NEVER push.** `git push` in any form is prohibited for you
  and for every subagent you dispatch. Do not run PR-creation commands that
  push. When an outcome would require a push, stop and tell the user to push
  manually.
- Every gate below is blocking: stop and wait for the user's explicit answer
  before continuing.

## Setup

1. **Dependency guard:** the superpowers plugin must be available — check
   that the skill `superpowers:subagent-driven-development` appears in your
   available skills. If missing, STOP and tell the user to install the
   superpowers plugin first: `/plugin marketplace add obra/superpowers-marketplace`,
   then `/plugin install superpowers@superpowers-marketplace`.
   (No Fable model guard here: the orchestrator
   writes no code; models are pinned in the subagents.)
2. **Resolve the plan:** if $ARGUMENTS names a file, use it. Otherwise list
   `docs/superpowers/plans/*.md` newest-first and ask the user which plan to
   execute. Read the plan fully.
3. **Resume point:** find the first unchecked `- [ ]` step in the plan and
   confirm with the user that this is where execution should resume. If
   every step is already checked, skip Phase 2 and go directly to Phase 3.
4. **Locate the spec:** use the spec path referenced in the plan header; if
   absent, ask the user.

## Phase 2 — Implementer (worker-executed, Claude-reviewed)

1. **Branch gate (mandatory):** before ANY implementation work, confirm the
   development branch with the user. When resuming, the work usually
   continues on the branch where earlier task commits live — confirm that
   branch is checked out and correct. Otherwise offer:
   - create a new branch from current HEAD (suggest `feature/<topic>`),
   - use the current branch as-is,
   - a branch name the user specifies,
   - isolate in a worktree via the `superpowers:using-git-worktrees` skill.
   Do NOT start implementation until the user has confirmed. Record the
   review base for Phase 3: the branch point of the feature branch
   (`git merge-base HEAD <parent-branch>`, or a rev the user provides),
   confirmed with the user.
2. **Worker gate:** before the first task dispatch, ask the user which
   worker executes Phase 2's implementation units — Codex MCP or DeepSeek
   MCP — per this project's global Worker Delegation Policy (`WORKER.md`).
   Treat the answer as the standing preference for every task in this run,
   including Phase 3's fix-loop dispatches; do not re-ask per task unless
   the policy's own exception applies. **Exception:** any single task that
   genuinely needs an MCP tool DeepSeek cannot reach (codebase-memory-mcp,
   CodeGraph, browser tools, Slack, Jira/Confluence) goes to Codex
   regardless of the standing preference — tell the user when this
   happens instead of silently routing around it.
3. Follow the `superpowers:subagent-driven-development` skill's structure —
   workspace/ledger, per-task brief extraction, review-package generation,
   per-task review, fix loop — with ONE substitution: **the per-task
   implementer is the worker chosen at the gate above (Codex MCP or
   DeepSeek MCP), not a Claude subagent.** For each task:
   - Extract the task's brief exactly as the skill's `task-brief` step
     prescribes.
   - Dispatch it to the chosen worker, `cwd` = the repository root,
     `sandbox: workspace-write`, `approval-policy: never` for either
     worker, matching the global policy's Initial Calls conventions:
     - **Codex** (`mcp__codex__codex`): `model: gpt-5.6-sol`, and
       `config.model_reasoning_effort` chosen per task by complexity —
       single-file mechanical change → `high`; standard TDD task →
       `xhigh`; cross-module, new-subsystem, or algorithm-heavy task →
       `max`; default `xhigh` when unsure; never auto-select `ultra` —
       reserved for an explicit user request.
     - **DeepSeek** (`deepseek`): omit `model` (ds-mcp's own default)
       unless the user requests otherwise, and `reasoning-effort` on a
       two-tier scale — `high` (ds-mcp's own default) for mechanical and
       standard TDD tasks alike; `max` only for cross-module,
       new-subsystem, or algorithm-heavy tasks, used sparingly.
     - If the plan annotates the task with an explicit effort level
       (e.g. `Effort: max`), use that annotation; for a DeepSeek dispatch,
       translate the plan's Codex-shaped label to DeepSeek's ladder
       position (`high`/`xhigh`→`high`, `max`→`max`).
     - Effort is fixed at dispatch time for either worker: the reply tool
       (`codex-reply` / `deepseek-reply`) inherits it and cannot raise it,
       so prefer the higher tier when a task sits between two.
     The `prompt` is the task's brief verbatim plus the scene-setting
     context the skill's dispatch template calls for (where this task
     fits, interfaces from earlier tasks).
   - `developer-instructions` for every dispatch: follow strict TDD (RED —
     failing test first, confirm it fails for the expected reason — GREEN
     — minimal implementation, confirm it passes — REFACTOR with tests
     green); where coverage tooling is configured keep touched-code
     coverage at or above 80%; run every verification command the task
     specifies and report actual output; commit with a conventional
     message; **NEVER run `git push` in any form**; if the task is
     ambiguous or needs a scope/architecture decision, stop and report the
     question instead of guessing; for a DeepSeek dispatch, since it has
     no codebase-memory-mcp/CodeGraph/other-MCP access, either let it use
     its own shell `rg`/`grep` for exploration beyond the assigned files
     or complete that discovery yourself first and hand it concrete
     paths; report back what was implemented, files changed, the exact
     test commands and their output, the commit hash, and one status on
     its own line — DONE, DONE_WITH_CONCERNS, BLOCKED, or NEEDS_CONTEXT —
     with a one-line reason if not DONE.
   - Record the returned `structuredContent.threadId`. Per-task correction
     rounds (findings from the task reviewer, below) use `codex-reply` or
     `deepseek-reply` on that same thread — matching whichever worker
     started the unit — not a fresh dispatch, per the global Worker
     Delegation Policy's Thread Continuity rule. If two correction rounds
     on the same thread don't resolve the finding, the policy's own
     standing fallback applies regardless of worker: edit the fix
     yourself and disclose the degraded path and reason in the task's
     completion note, rather than escalating to a different model or
     worker.
4. Dispatch the per-task reviewer as agent type `codeflow:task-reviewer`
   (bare-name fallback `task-reviewer`) instead of an ad-hoc dispatch of
   the skill's own template — do not pass an explicit `model` parameter
   for this dispatch: an explicit override takes precedence over the
   agent type's frontmatter and would silently replace the required
   `model: opus` / `effort: max` pin, the same failure the skill's own
   Model Selection section warns about ("an omitted model inherits your
   session's model"). Provide it the same inputs the skill's task-review
   step already collects: the task brief file, the implementer's report
   file, the review-package diff file with base/head SHAs, and the
   global-constraints block copied verbatim from the plan or spec.
5. Do not run the skill's own "Final Review" section — codeflow's own
   Phase 3, below, replaces that step entirely.
6. All commits stay local. No push, no exceptions — state this in every
   worker's `developer-instructions` block, not just here.

## Phase 3 — Reviewer (Fable subagent) + fix loop

1. Follow the `superpowers:requesting-code-review` skill, but dispatch agent
   type `codeflow:reviewer` (bare-name fallback `reviewer`) instead of the
   default code-reviewer agent, and do not pass an explicit `model`
   parameter for this dispatch — an explicit override takes precedence over
   the agent type's own frontmatter and would silently replace the
   required Fable pin. Provide it: the spec path, the plan path, the base
   ref recorded at the Phase 2 branch gate, and the head ref (current
   HEAD).
2. If the reviewer reports CRITICAL or HIGH findings:
   a. Triage them with the `superpowers:receiving-code-review` skill —
      verify each finding against the code before acting; push back on
      invalid findings.
   b. For each valid CRITICAL/HIGH finding, dispatch a fresh work unit to
      the chosen worker (per the Phase 2 worker gate) to fix it — same
      dispatch conventions, per-task reasoning-effort selection, and
      `developer-instructions` as Phase 2 (TDD: regression test first,
      then fix, then commit; never push). A fresh dispatch, not a thread
      reply: a finding can span code from more than one task's original
      dispatch.
   c. Re-run step 1 with a fresh `codeflow:reviewer`.
3. **Loop cap:** after 3 review rounds with CRITICAL/HIGH findings still
   present, STOP and report the remaining findings to the user for a
   decision.
4. MEDIUM/LOW findings: report them to the user; do not auto-fix.
5. When the verdict is APPROVE: run the
   `superpowers:finishing-a-development-branch` skill, EXCLUDING every
   option that pushes or creates a PR. Allowed outcomes: merge locally, keep
   the branch, or discard. If the user wants a remote/PR, tell them to push
   manually.
6. **Final report:** spec path, plan path, branch name, commit list
   (`git log <base>..HEAD --oneline`), diff stat, review verdict, and test
   coverage notes.
