---
description: Run the full codeflow workflow — Fable planner → Codex implementer → Fable reviewer
argument-hint: [feature description]
---

# codeflow: full three-phase workflow

Drive the codeflow workflow for the feature described in $ARGUMENTS. If
$ARGUMENTS is empty, first ask the user what to build.

## Global constraints (every phase, every subagent)

- **Commit only, NEVER push.** `git push` in any form is prohibited for you
  and for every subagent you dispatch. Do not run PR-creation commands that
  push. When an outcome would require a push, stop and tell the user to push
  manually.
- Every gate below is blocking: stop and wait for the user's explicit answer
  before continuing.

## Phase 0 — Guards

1. **Dependency guard:** the superpowers plugin must be available — check
   that the skill `superpowers:brainstorming` appears in your available
   skills. If missing, STOP and tell the user to install the superpowers
   plugin first: `/plugin marketplace add obra/superpowers-marketplace`,
   then `/plugin install superpowers@superpowers-marketplace`.
2. **Model check (routing, not a stop):** check which model you are running
   as (your system prompt states it). This decides HOW Phase 1 runs below —
   it is no longer a hard stop:
   - Fable (`claude-fable-5` family): run Phase 1 **inline** (§1a).
   - Any other model: run Phase 1 **relayed** through a pinned-Fable
     subagent (§1b), so the planning work still happens on Fable even
     though this session isn't.

## Phase 1 — Planner (Fable, inline or relayed per Phase 0)

### §1a — Inline (session model is Fable)

1. Invoke the `superpowers:brainstorming` skill and follow it exactly:
   interactive clarifying questions, design presented for approval, spec
   written to `docs/superpowers/specs/` and committed, user review gate.
2. Continue into the `superpowers:writing-plans` skill (brainstorming's
   terminal state): plan written to `docs/superpowers/plans/` and committed.
3. Gate: the user approves the plan. When writing-plans offers execution
   options, this workflow always proceeds subagent-driven — but the branch
   gate in Phase 2 comes first.
4. **Model drift check:** this phase spans many conversational turns.
   Re-verify you are still running as Fable immediately before committing
   the spec and again immediately before committing the plan; if the
   session model has drifted, STOP and switch to §1b instead of committing.

### §1b — Relayed (session model is not Fable)

The actual design and planning work happens in a `planner` subagent
(`model: fable`, pinned regardless of this session's model — bare-name
fallback `planner` if `codeflow:planner` isn't found). Because a subagent
cannot hold a live conversation, drive it as a relay loop: dispatch it,
relay its question to the human, dispatch it again with the answer, repeat.

**Two hard rules for this loop:**
- **Always dispatch a fresh subagent, never resume one.** Even if your
  environment supports continuing or resuming a previously-dispatched
  agent with its memory intact, do not use that here — dispatch a brand
  new `codeflow:planner` instance every round, passing it only the
  transcript file. This keeps the relay working in any environment and
  keeps provenance clean (see the next rule).
- **You are the only writer of the transcript files.** You append every
  `QUESTION:`/answer pair to the transcript after relaying it to the real
  human. The planner subagent only ever reads the transcript — it must
  never write to it itself. A subagent's own file claiming "the human said
  X" is indistinguishable from a fabricated approval to anyone auditing it
  later; keeping the write on your side, where the real conversation with
  the human actually happened, is what makes that claim trustworthy.

1. Set up: derive a short topic slug from the feature description; create
   the workspace directory `.superpowers/codeflow-relay/<slug>/` with an
   empty `spec-transcript.md`.
2. **Spec relay loop** (cap: 30 rounds — if exceeded, STOP and report to
   the user): dispatch `codeflow:planner` with stage `spec`, the feature
   description, today's date, and the transcript file path. Its final
   message is exactly one of:
   - `QUESTION: <text>` — show `<text>` to the human verbatim as your own
     message and wait for their reply. Append both to
     `spec-transcript.md`, then dispatch again.
   - `SPEC_READY: <path>` — verify the file exists and `git log -1 --
     <path>` shows a commit touching it. If either check fails, treat this
     as a malformed response (see below). Otherwise the spec stage is
     done; record `<path>`.
   - Anything else: malformed response — show the human the raw text and
     ask how to proceed. Do not guess.
3. **Plan relay loop** (cap: 30 rounds, same malformed-response handling):
   same shape as step 2, stage `plan`, passing the approved spec path, into
   a fresh `plan-transcript.md` in the same workspace directory. Terminal
   signal is `PLAN_READY: <path>`.
4. Once `PLAN_READY` is confirmed, delete the workspace directory — the
   record now lives in git. Continue to Phase 2 exactly as the inline path
   would, using the plan at `<path>`.

## Phase 2 — Implementer (Codex-executed, Claude-reviewed)

1. **Branch gate (mandatory):** before ANY implementation work, confirm the
   development branch with the user. Offer these options:
   - create a new branch from current HEAD (suggest `feature/<topic>` derived
     from the spec topic) — recommended default,
   - use the current branch as-is,
   - a branch name the user specifies,
   - isolate in a worktree via the `superpowers:using-git-worktrees` skill.
   Do NOT start implementation until the user has confirmed. Record the
   confirmed branch point (`git rev-parse HEAD` before the first task
   commit) — Phase 3 uses it as the review base.
2. Follow the `superpowers:subagent-driven-development` skill's structure —
   workspace/ledger, per-task brief extraction, review-package generation,
   per-task review, fix loop — with ONE substitution: **the per-task
   implementer is Codex MCP, not a Claude subagent.** For each task:
   - Extract the task's brief exactly as the skill's `task-brief` step
     prescribes.
   - Dispatch it with `mcp__codex__codex`: `cwd` = the repository root,
     `model: gpt-5.6-sol`, `sandbox: workspace-write`,
     `approval-policy: never`, and `config.model_reasoning_effort` chosen
     per task by complexity:
     - If the plan annotates the task with an explicit effort level
       (e.g. `Effort: max`), use that annotation.
     - Otherwise classify the task: single-file mechanical change →
       `high`; standard TDD task → `xhigh`; cross-module, new-subsystem,
       or algorithm-heavy task → `max`.
     - When unsure, default to `xhigh`. Never auto-select `ultra` — it is
       reserved for an explicit user request.
     - Effort is fixed at dispatch time: `codex-reply` correction rounds
       inherit it and cannot raise it, so prefer the higher tier when a
       task sits between two.
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
     question instead of guessing; report back what was implemented, files
     changed, the exact test commands and their output, the commit hash,
     and one status on its own line — DONE, DONE_WITH_CONCERNS, BLOCKED, or
     NEEDS_CONTEXT — with a one-line reason if not DONE.
   - Record the returned `structuredContent.threadId`. Per-task correction
     rounds (findings from the task reviewer, below) use
     `mcp__codex__codex-reply` on that same thread — not a fresh dispatch —
     matching this project's standing Codex-usage policy. If two
     correction rounds on the same thread don't resolve the finding, this
     project's own standing fallback applies: edit the fix yourself and
     disclose the degraded path and reason in the task's completion note,
     rather than escalating to a different model.
3. Dispatch the per-task reviewer as agent type `codeflow:task-reviewer`
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
4. Do not run the skill's own "Final Review" section — codeflow's own
   Phase 3, below, replaces that step entirely.
5. All commits stay local. No push, no exceptions — state this in every
   Codex `developer-instructions` block, not just here.

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
   b. For each valid CRITICAL/HIGH finding, dispatch a fresh
      `mcp__codex__codex` work unit to fix it — same dispatch conventions,
      per-task reasoning-effort selection, and `developer-instructions`
      as Phase 2 (TDD: regression test first, then fix, then commit; never
      push). A fresh dispatch, not a thread reply: a finding can span code
      from more than one task's original dispatch.
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
