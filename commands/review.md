---
description: Run codeflow Phase 3 only — Fable final review of the current branch with automatic Codex fix loop
argument-hint: [base-ref]
---

# codeflow: standalone final review

Run the codeflow final review (Phase 3) against the current branch, without
re-running planning or implementation.

## Global constraints (every step, every subagent)

- **Commit only, NEVER push.** `git push` in any form is prohibited for you
  and for every subagent you dispatch. Do not run PR-creation commands that
  push. When an outcome would require a push, stop and tell the user to push
  manually.
- Every gate below is blocking: stop and wait for the user's explicit answer
  before continuing.

## Setup

1. **Dependency guard:** the superpowers plugin must be available — check
   that the skill `superpowers:requesting-code-review` appears in your
   available skills. If missing, STOP and tell the user to install the
   superpowers plugin first: `/plugin marketplace add obra/superpowers-marketplace`,
   then `/plugin install superpowers@superpowers-marketplace`.
2. **Review range:** base ref = $ARGUMENTS if given. Otherwise compute
   `git merge-base HEAD main` (fall back to `master` if `main` does not
   exist) and confirm it with the user; if neither exists or the user
   disagrees, ask for the base ref. Head ref = current HEAD.
3. **Locate spec and plan:** suggest the newest files in
   `docs/superpowers/specs/` and `docs/superpowers/plans/` and confirm with
   the user; if they do not exist, ask the user for paths, or proceed
   spec-less only if the user explicitly says there is no spec (the reviewer
   then reviews against the diff alone).

## Phase 3 — Reviewer (Fable subagent) + fix loop

1. Follow the `superpowers:requesting-code-review` skill, but dispatch agent
   type `codeflow:reviewer` (bare-name fallback `reviewer`) instead of the
   default code-reviewer agent, and do not pass an explicit `model`
   parameter for this dispatch — an explicit override takes precedence over
   the agent type's own frontmatter and would silently replace the
   required Fable pin. Provide it: the spec path, the plan path, the base
   ref from Setup, and the head ref (current HEAD).
2. If the reviewer reports CRITICAL or HIGH findings:
   a. Triage them with the `superpowers:receiving-code-review` skill —
      verify each finding against the code before acting; push back on
      invalid findings.
   b. For each valid CRITICAL/HIGH finding, dispatch a fresh
      `mcp__codex__codex` work unit to fix it — same dispatch conventions
      as this project's Phase 2 (TDD: regression test first, then fix,
      then commit; never push). A fresh dispatch, not a thread reply: a
      finding can span code from more than one task's original dispatch.
   c. Re-run step 1 with a fresh `codeflow:reviewer`.
3. **Loop cap:** after 3 review rounds with CRITICAL/HIGH findings still
   present, STOP and report the remaining findings to the user for a
   decision.
4. MEDIUM/LOW findings: report them to the user; do not auto-fix.
5. When the verdict is APPROVE: offer the
   `superpowers:finishing-a-development-branch` skill, EXCLUDING every
   option that pushes or creates a PR. Allowed outcomes: merge locally, keep
   the branch, or discard. If the user wants a remote/PR, tell them to push
   manually.
6. **Final report:** spec path, plan path, branch name, commit list
   (`git log <base>..HEAD --oneline`), diff stat, review verdict, and test
   coverage notes.
