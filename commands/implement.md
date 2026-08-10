---
description: Resume codeflow at Phase 2 — execute an existing plan with Opus implementer subagents, then run the Fable final review
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

## Phase 2 — Implementer (Opus subagents)

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
2. Invoke the `superpowers:subagent-driven-development` skill with two
   substitutions: (a) dispatch every per-task implementation subagent as
   agent type `codeflow:implementer` instead of a general-purpose agent —
   if that type is not found, the same agent may be registered under the
   bare name `implementer`; (b) stop before the skill's own "Final Review"
   section and do not dispatch its whole-branch reviewer — codeflow's own
   Phase 3, below, replaces that step entirely. Pass each task's text
   verbatim as the skill prescribes for the per-task loop.
3. **Never pass an explicit `model` parameter when dispatching
   `codeflow:implementer`**, even when the skill's own Model Selection
   guidance recommends a tier for the task's complexity (e.g. "cheap tier"
   for a mechanical, fully-specified task). An explicit model override on a
   dispatch call takes precedence over the agent type's own frontmatter and
   would silently replace the required Opus pin with whatever tier was
   picked. Omit the model parameter entirely for this agent type — its
   frontmatter is the only model directive.
4. Leave the skill's per-task spec-review and code-review subagents at
   their defaults — only the coding agents have a pinned model.
5. All commits stay local. No push, no exceptions.

## Phase 3 — Reviewer (Fable subagent) + fix loop

1. Follow the `superpowers:requesting-code-review` skill, but dispatch agent
   type `codeflow:reviewer` (bare-name fallback `reviewer`) instead of the
   default code-reviewer agent, and do not pass an explicit `model`
   parameter for this dispatch — `codeflow:reviewer`'s frontmatter is the
   only model directive, for the same reason as Phase 2's implementer
   dispatch. Provide it: the spec path, the plan path, the base ref
   recorded at the Phase 2 branch gate, and the head ref (current HEAD).
2. If the reviewer reports CRITICAL or HIGH findings:
   a. Triage them with the `superpowers:receiving-code-review` skill —
      verify each finding against the code before acting; push back on
      invalid findings.
   b. For each valid CRITICAL/HIGH finding, dispatch one
      `codeflow:implementer` subagent (no explicit `model` parameter — same
      reason as Phase 2) to fix it (TDD applies: regression test first,
      then fix, then commit).
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
