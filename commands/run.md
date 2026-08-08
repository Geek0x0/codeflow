---
description: Run the full codeflow workflow — Fable planner → Opus implementers → Fable reviewer
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

1. **Model guard:** check which model you are running as (your system prompt
   states it). If you are not Fable (`claude-fable-5` family), STOP and tell
   the user: "codeflow Phase 1 (planner) requires the Fable model. Run
   `/model fable`, then re-run `/codeflow:run`." Do not continue.
2. **Dependency guard:** the superpowers plugin must be available — check
   that the skill `superpowers:brainstorming` appears in your available
   skills. If missing, STOP and tell the user to install the superpowers
   plugin first: `/plugin marketplace add obra/superpowers-marketplace`,
   then `/plugin install superpowers@superpowers-marketplace`.

## Phase 1 — Planner (this session, Fable)

1. Invoke the `superpowers:brainstorming` skill and follow it exactly:
   interactive clarifying questions, design presented for approval, spec
   written to `docs/superpowers/specs/` and committed, user review gate.
2. Continue into the `superpowers:writing-plans` skill (brainstorming's
   terminal state): plan written to `docs/superpowers/plans/` and committed.
3. Gate: the user approves the plan. When writing-plans offers execution
   options, this workflow always proceeds subagent-driven — but the branch
   gate in Phase 2 comes first.

## Phase 2 — Implementer (Opus subagents)

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
2. Invoke the `superpowers:subagent-driven-development` skill with ONE
   substitution: dispatch every per-task implementation subagent as agent
   type `codeflow:implementer` instead of a general-purpose agent. (If that
   type is not found, the same agent may be registered under the bare name
   `implementer`.) Pass each task's text verbatim as that skill prescribes.
3. Leave the skill's per-task spec-review and code-review subagents at their
   defaults — only the coding agents have a pinned model.
4. All commits stay local. No push, no exceptions.

## Phase 3 — Reviewer (Fable subagent) + fix loop

1. Follow the `superpowers:requesting-code-review` skill, but dispatch agent
   type `codeflow:reviewer` (bare-name fallback `reviewer`) instead of the
   default code-reviewer agent. Provide it: the spec path, the plan path,
   the base ref recorded at the Phase 2 branch gate, and the head ref
   (current HEAD).
2. If the reviewer reports CRITICAL or HIGH findings:
   a. Triage them with the `superpowers:receiving-code-review` skill —
      verify each finding against the code before acting; push back on
      invalid findings.
   b. For each valid CRITICAL/HIGH finding, dispatch one
      `codeflow:implementer` subagent to fix it (TDD applies: regression
      test first, then fix, then commit).
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
