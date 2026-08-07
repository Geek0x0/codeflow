# codeflow Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `codeflow` Claude Code plugin: a three-phase workflow (Fable planner → Opus implementers → Fable reviewer) orchestrated through commands and model-pinned agents, delegating all process logic to the superpowers plugin.

**Architecture:** Pure Markdown/JSON plugin — a manifest, three command files that orchestrate phases on the main thread, and two agent files whose only structural job is pinning models (`opus` for the implementer, `fable` for the reviewer). Commands invoke superpowers skills by name and substitute codeflow agent types where the spec requires pinned models.

**Tech Stack:** Claude Code plugin system (plugin.json manifest, `commands/*.md`, `agents/*.md`), superpowers plugin (runtime dependency).

**Spec:** `docs/superpowers/specs/2026-08-07-codeflow-plugin-design.md`

## Global Constraints

- **Commit only, never push.** No `git push` in any form anywhere in this repo, and every file written by this plan must carry that same prohibition where it defines behavior (commands, agents).
- Conventional commit messages: `feat:`, `fix:`, `docs:`, `chore:`.
- Plugin depends on the superpowers plugin at **runtime** only; nothing in this repo imports or vendors superpowers content.
- Model pinning uses aliases `opus` and `fable`; the README documents full-ID fallback (`claude-opus-5`, `claude-fable-5`).
- Agent dispatch uses plugin-qualified type `codeflow:implementer` / `codeflow:reviewer`, with an inline fallback note for bare names (`implementer` / `reviewer`) since the exact namespace is pinned during smoke testing.
- All files are UTF-8, LF line endings.
- This plan creates files only; there is no build step and no test framework — each task's verification is a structural check with an exact command and expected output.

---

### Task 1: Plugin manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

**Interfaces:**
- Produces: plugin name `codeflow` — the namespace every later task's command (`/codeflow:run`, `/codeflow:implement`, `/codeflow:review`) and agent type (`codeflow:implementer`, `codeflow:reviewer`) hangs off.

- [ ] **Step 1: Write the manifest**

Create `.claude-plugin/plugin.json` with exactly this content:

```json
{
  "name": "codeflow",
  "version": "0.1.0",
  "description": "Three-phase development workflow: Fable planner (brainstorming + writing-plans), Opus implementers (subagent-driven development), Fable reviewer (final review with fix loop). Requires the superpowers plugin."
}
```

- [ ] **Step 2: Verify the manifest parses and has required fields**

Run:
```bash
python3 -c "import json; d = json.load(open('.claude-plugin/plugin.json')); assert d['name'] == 'codeflow', d; assert d['version'] == '0.1.0'; assert 'superpowers' in d['description']; print('manifest OK')"
```
Expected: `manifest OK`

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add codeflow plugin manifest"
```

---

### Task 2: Implementer agent (Opus)

**Files:**
- Create: `agents/implementer.md`

**Interfaces:**
- Produces: agent type `codeflow:implementer` (bare name `implementer`), `model: opus`. Commands in Tasks 4–6 dispatch this type for every coding unit (plan tasks and review fixes). Its contract: receives one self-contained task prompt; does TDD; commits; never pushes; reports files changed, test results, commit hash, blockers.

- [ ] **Step 1: Write the agent file**

Create `agents/implementer.md` with exactly this content:

````markdown
---
name: implementer
description: Executes exactly one task from a codeflow implementation plan or one review-fix unit. Dispatched per-task during codeflow Phase 2 (subagent-driven development) and for Phase 3 fixes. Strict TDD, commits its work, never pushes.
model: opus
tools: Read, Write, Edit, Bash, Grep, Glob
---

You are a codeflow implementer. You receive exactly one work unit — a task
from an implementation plan, or a single review-fix — and you execute it
precisely.

## Rules

- Execute only the work unit you were given. Do not start adjacent work, do
  not redesign the overall solution, do not continue into other tasks.
- Follow strict TDD for every behavior change:
  1. RED — write the failing test first. Run it and confirm it fails for the
     expected reason.
  2. GREEN — write the minimal implementation. Run the test and confirm it
     passes.
  3. REFACTOR — clean up while keeping tests green.
- If the project has coverage tooling configured, keep coverage of the code
  you touch at or above 80%.
- Run every verification command your work unit specifies. Report actual
  output, never assumptions.
- Commit your changes with a conventional commit message (`feat:`, `fix:`,
  `test:`, `docs:`, `chore:`).
- **NEVER push.** `git push` in any form — including `--set-upstream`,
  `--force`, tags, or any PR-creation tool that pushes — is prohibited.
- If your work unit is ambiguous, conflicts with repository evidence, or
  needs a scope/architecture decision, STOP and report the question instead
  of guessing.

## Report

When done, report: what you implemented, files changed, test commands run
with their actual results, the commit hash, and any blockers or concerns.
````

- [ ] **Step 2: Verify frontmatter fields**

Run:
```bash
awk '/^---$/{n++} n==1' agents/implementer.md | grep -E '^(name|model|tools):' && grep -c 'NEVER push' agents/implementer.md
```
Expected output contains: `name: implementer`, `model: opus`, `tools: Read, Write, Edit, Bash, Grep, Glob`, and count `1`.

- [ ] **Step 3: Commit**

```bash
git add agents/implementer.md
git commit -m "feat: add opus implementer agent"
```

---

### Task 3: Reviewer agent (Fable)

**Files:**
- Create: `agents/reviewer.md`

**Interfaces:**
- Produces: agent type `codeflow:reviewer` (bare name `reviewer`), `model: fable`. Commands in Tasks 4–6 dispatch it with four inputs: spec path, plan path, base ref, head ref. Its contract: read-only review, runs the test suite, reports findings bucketed CRITICAL/HIGH/MEDIUM/LOW with a final verdict `APPROVE` or `NEEDS-FIXES`.

- [ ] **Step 1: Write the agent file**

Create `agents/reviewer.md` with exactly this content:

````markdown
---
name: reviewer
description: Whole-feature final review for codeflow Phase 3. Reviews the full diff against the spec and plan with test coverage as a first-class dimension. Read-only — never modifies files, never pushes.
model: fable
tools: Read, Grep, Glob, Bash
---

You are the codeflow final reviewer. You review a completed feature branch
against its spec and plan.

You will be given: a spec path, a plan path, a base ref, and a head ref.

## Process

1. Read the spec and the plan.
2. Run `git log <base>..<head> --oneline` and `git diff <base>...<head>` to
   see everything that changed.
3. Check the work against the spec requirement by requirement: implemented,
   partially implemented, or missing? Flag anything built that the spec does
   not ask for.
4. Verify test claims independently: run the project's test suite yourself
   and report the actual output. Do not trust commit messages or comments.
5. Assess test coverage: does every new behavior have a test? If coverage
   tooling is configured, check the 80% minimum for changed code. Missing or
   weak tests for new behavior are at least a HIGH finding.

## Rules

- Read-only review: never create, modify, or delete files. Run only
  read-only commands plus the project's test/coverage commands.
- **NEVER push.** Never run `git push` in any form. Never commit.

## Report format

- **What was built:** short summary of the feature as implemented.
- **Spec compliance:** requirement-by-requirement verdict (met / partial /
  missing / extra).
- **Test coverage:** assessment of tests for new behavior; coverage numbers
  if tooling exists; the exact test command you ran and its result.
- **Findings by severity:**
  - CRITICAL — security vulnerability, data loss risk, broken build/tests.
  - HIGH — bug, spec violation, missing or weak tests for new behavior.
  - MEDIUM — maintainability concern.
  - LOW — style or minor suggestion.
  Each finding: `file:line`, what is wrong, why it matters, suggested fix.
- **Verdict:** `APPROVE` (zero CRITICAL and zero HIGH) or `NEEDS-FIXES`.
````

- [ ] **Step 2: Verify frontmatter fields**

Run:
```bash
awk '/^---$/{n++} n==1' agents/reviewer.md | grep -E '^(name|model|tools):' && grep -c 'NEVER push' agents/reviewer.md
```
Expected output contains: `name: reviewer`, `model: fable`, `tools: Read, Grep, Glob, Bash`, and count `1`.

- [ ] **Step 3: Commit**

```bash
git add agents/reviewer.md
git commit -m "feat: add fable reviewer agent"
```

---

### Task 4: Main command `/codeflow:run`

**Files:**
- Create: `commands/run.md`

**Interfaces:**
- Consumes: agent types `codeflow:implementer` (Task 2) and `codeflow:reviewer` (Task 3); superpowers skills `brainstorming`, `writing-plans`, `subagent-driven-development`, `requesting-code-review`, `receiving-code-review`, `using-git-worktrees`, `finishing-a-development-branch`.
- Produces: the canonical Phase 0–3 orchestration text. Tasks 5 and 6 copy their Phase 2/Phase 3 sections from this file verbatim (deliberate duplication — command files cannot include each other).

- [ ] **Step 1: Write the command file**

Create `commands/run.md` with exactly this content:

````markdown
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
   plugin first.

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
````

- [ ] **Step 2: Verify frontmatter and phase structure**

Run:
```bash
awk '/^---$/{n++} n==1' commands/run.md | grep -c '^description:' && grep -c '^## Phase' commands/run.md && grep -c 'NEVER push' commands/run.md
```
Expected: `1`, then `4` (Phases 0–3), then a count of at least `1`.

- [ ] **Step 3: Verify agent references are consistent with Tasks 2–3**

Run:
```bash
grep -o 'codeflow:implementer\|codeflow:reviewer' commands/run.md | sort | uniq -c
```
Expected: both `codeflow:implementer` and `codeflow:reviewer` appear (implementer ≥ 2 — Phase 2 and Phase 3 fixes; reviewer ≥ 2).

- [ ] **Step 4: Commit**

```bash
git add commands/run.md
git commit -m "feat: add /codeflow:run main workflow command"
```

---

### Task 5: Resume command `/codeflow:implement`

**Files:**
- Create: `commands/implement.md`

**Interfaces:**
- Consumes: plan files in `docs/superpowers/plans/` (checkbox syntax from writing-plans), agent types from Tasks 2–3, Phase 2/3 text from Task 4 (duplicated verbatim by design).
- Produces: `/codeflow:implement [plan-path]` entry point.

- [ ] **Step 1: Write the command file**

Create `commands/implement.md` with exactly this content:

````markdown
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
   superpowers plugin first. (No Fable model guard here: the orchestrator
   writes no code; models are pinned in the subagents.)
2. **Resolve the plan:** if $ARGUMENTS names a file, use it. Otherwise list
   `docs/superpowers/plans/*.md` newest-first and ask the user which plan to
   execute. Read the plan fully.
3. **Resume point:** find the first unchecked `- [ ]` step in the plan and
   confirm with the user that this is where execution should resume.
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
   (`git merge-base HEAD <parent-branch>` or the recorded branch-gate rev),
   confirmed with the user.
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
````

- [ ] **Step 2: Verify frontmatter and structure**

Run:
```bash
awk '/^---$/{n++} n==1' commands/implement.md | grep -E '^(description|argument-hint):' | wc -l && grep -c 'NEVER push' commands/implement.md && grep -c 'docs/superpowers/plans' commands/implement.md
```
Expected: `2`, then at least `1`, then at least `1`.

- [ ] **Step 3: Commit**

```bash
git add commands/implement.md
git commit -m "feat: add /codeflow:implement resume command"
```

---

### Task 6: Standalone review command `/codeflow:review`

**Files:**
- Create: `commands/review.md`

**Interfaces:**
- Consumes: agent types from Tasks 2–3; Phase 3 text from Task 4 (duplicated verbatim by design, with a base-resolution setup step instead of the branch gate).
- Produces: `/codeflow:review [base-ref]` entry point.

- [ ] **Step 1: Write the command file**

Create `commands/review.md` with exactly this content:

````markdown
---
description: Run codeflow Phase 3 only — Fable final review of the current branch with automatic Opus fix loop
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
   superpowers plugin first.
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
   default code-reviewer agent. Provide it: the spec path, the plan path,
   the base ref from Setup, and the head ref (current HEAD).
2. If the reviewer reports CRITICAL or HIGH findings:
   a. Triage them with the `superpowers:receiving-code-review` skill —
      verify each finding against the code before acting; push back on
      invalid findings.
   b. For each valid CRITICAL/HIGH finding, dispatch one
      `codeflow:implementer` subagent (bare-name fallback `implementer`) to
      fix it (TDD applies: regression test first, then fix, then commit).
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
````

- [ ] **Step 2: Verify frontmatter and structure**

Run:
```bash
awk '/^---$/{n++} n==1' commands/review.md | grep -E '^(description|argument-hint):' | wc -l && grep -c 'merge-base' commands/review.md && grep -c 'NEVER push' commands/review.md
```
Expected: `2`, then at least `1`, then at least `1`.

- [ ] **Step 3: Commit**

```bash
git add commands/review.md
git commit -m "feat: add /codeflow:review standalone review command"
```

---

### Task 7: README

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: everything above — command names, agent names, model aliases, superpowers dependency, no-push policy.

- [ ] **Step 1: Write the README**

Create `README.md` with exactly this content:

````markdown
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
└── agents/
    ├── implementer.md           # model: opus
    └── reviewer.md              # model: fable
```
````

- [ ] **Step 2: Verify README references match reality**

Run:
```bash
grep -c '/codeflow:run\|/codeflow:implement\|/codeflow:review' README.md && grep -c 'never push' README.md && ls commands/run.md commands/implement.md commands/review.md agents/implementer.md agents/reviewer.md .claude-plugin/plugin.json
```
Expected: first count ≥ 3, second ≥ 1, and `ls` lists all six files without error.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add codeflow README"
```

---

### Task 8: Whole-plugin consistency sweep

**Files:**
- Modify: (only if a check fails) any of the six plugin files

**Interfaces:**
- Consumes: all files from Tasks 1–7.

- [ ] **Step 1: Run the full static validation**

Run:
```bash
set -e
python3 -c "import json; json.load(open('.claude-plugin/plugin.json')); print('json OK')"
for f in agents/implementer.md agents/reviewer.md commands/run.md commands/implement.md commands/review.md; do
  head -1 "$f" | grep -qx -- '---' || { echo "FAIL: $f missing frontmatter"; exit 1; }
done
grep -q 'model: opus' agents/implementer.md
grep -q 'model: fable' agents/reviewer.md
for f in commands/run.md commands/implement.md commands/review.md agents/implementer.md agents/reviewer.md; do
  grep -qi 'never push' "$f" || { echo "FAIL: $f missing no-push rule"; exit 1; }
done
grep -q 'codeflow:implementer' commands/run.md
grep -q 'codeflow:reviewer' commands/run.md
grep -qi 'branch gate' commands/run.md
grep -qi 'branch gate' commands/implement.md
grep -q 'superpowers:brainstorming' commands/run.md
for f in commands/run.md commands/implement.md; do
  grep -q 'superpowers:subagent-driven-development' "$f" || { echo "FAIL: $f missing SDD reference"; exit 1; }
done
for f in commands/run.md commands/implement.md commands/review.md; do
  grep -q 'superpowers:requesting-code-review' "$f" || { echo "FAIL: $f missing review-skill reference"; exit 1; }
done
echo 'ALL CHECKS PASS'
```
Expected: `json OK` then `ALL CHECKS PASS`.

- [ ] **Step 2: Fix and commit only if something failed**

If any check failed, fix the specific file, re-run Step 1 until it passes, then:

```bash
git add -A
git commit -m "fix: plugin consistency sweep corrections"
```

If nothing failed, no commit (working tree stays clean).

---

## Manual smoke test (post-plan, human-in-the-loop)

Not a task for an implementation subagent — it needs an interactive session and the user. After all tasks are done:

1. Load the plugin locally (`--plugin-dir` or a local marketplace).
2. In a session on a non-Fable model, run `/codeflow:run` → the Fable guard must stop it.
3. Switch to Fable, run `/codeflow:run` on a toy requirement → confirm: brainstorming interacts; spec and plan gates stop; branch gate stops; implementer subagents show Opus; final reviewer shows Fable; seed a defect to watch the fix loop; verify no `git push` at any point.
4. Pin down the exact agent-type namespace (`codeflow:implementer` vs `implementer`) and correct commands if needed.
5. Kill the session mid-Phase-2, then verify `/codeflow:implement` resumes from the first unchecked task, and `/codeflow:review` works standalone.
