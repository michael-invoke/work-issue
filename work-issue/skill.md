---
name: work-issue
description: Use when the user wants to work on a GitHub issue, references an issue number like "#42" or "issue 42", or says "/work-issue". Reads the issue, creates an isolated worktree, implements the fix/feature, creates a PR, and runs a proportional code review.
user_invocable: true
---

# Work Issue

Issue → classify → worktree → implement → verify → PR → review → cleanup. Scale effort to complexity.

## First-Run Setup

On the first invocation in a project, check if `.claude/settings.json` contains the `git worktree add` plan-gate hook. If it does not, add it automatically using the Edit or Write tool:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "prompt",
            "if": "Bash(git worktree add:*)",
            "prompt": "A git worktree is about to be created, which signals the start of implementation for a GitHub issue. Before allowing this, check the conversation history: has an implementation plan been posted as a comment on the GitHub issue via `gh issue comment`? If the plan has NOT been posted yet, BLOCK this action with reason 'Post the implementation plan to the GitHub issue before creating a worktree.' If the plan was already posted, ALLOW."
          }
        ]
      }
    ]
  }
}
```

Merge with any existing settings — do not overwrite. Tell the user: _"Added plan-gate hook to `.claude/settings.json` — worktree creation will be blocked until the implementation plan is posted to the issue."_

## Ground Rules

- **Local filesystem is source of truth.** Read actual files, never assume state.
- **One worktree per issue.** Created via `git worktree add` from the main repo directory.
- **Main repo stays on master.** No `git checkout -b`, no `git switch -c`, no branch switching in the main directory. Ever.
- **Scale effort to complexity.** A one-line fix does not need brainstorming.
- **Plan MUST be posted to the issue before creating a worktree.** Every path (SHORT, STANDARD, FULL) requires posting the implementation plan as a GitHub issue comment via `gh issue comment <number>` BEFORE `git worktree add`. No exceptions. If you are about to create a worktree and have not yet posted the plan, STOP and post it first.
- **Cleanup is not optional.** Worktree removal happens immediately after PR creation, not as a separate "later" step.
- **Serial by default.** Multiple issues are worked one at a time. Each issue is merged to master before the next begins. See **Multiple Issues** section.

## Multiple Issues

When given multiple issues to work on (e.g., "work on #13, #14, #15"):

```
████████████████████████████████████████████████████████████████████████
██  SERIAL BY DEFAULT. Work one issue at a time. Merge to master     ██
██  before starting the next. Do NOT parallelize unless the user     ██
██  explicitly says "in parallel" AND the dependency check passes.   ██
████████████████████████████████████████████████████████████████████████
```

### Serial flow (default)

1. Summarize all issues to the user with complexity classifications
2. Propose an order (simplest first, or dependency-aware if one issue builds on another)
3. Wait for user sign-off on order
4. Work issue 1 → PR → merge to master → pull master
5. Work issue 2 → PR → merge to master → pull master
6. Work issue 3 → PR → merge to master → pull master
7. Report all results

**Between each issue**: pull master so the next issue starts from a clean, up-to-date base. This eliminates cross-branch conflicts entirely.

### Parallel (only when explicitly requested)

If the user says "work these in parallel", run a dependency check FIRST:

```bash
# For each issue, list the files that will likely be touched
# Then check for overlap
```

**Present the overlap analysis to the user:**

```
Issue #13 touches: src/components/*.tsx, src/screens/maps/MapMainScreen.tsx
Issue #14 touches: src/stores/__tests__/*.ts, backend/**/__tests__/*.ts
Issue #15 touches: src/screens/reports/ReportsMainScreen.tsx, backend/src/handlers/reports.ts

Overlap: #13 and #15 both touch screens — risk of merge conflicts.
Recommendation: Run #14 in parallel with #13, then #15 after both merge.
```

**If files overlap**: tell the user, recommend serial for the overlapping set. Do not proceed in parallel without explicit user confirmation after seeing the overlap.

**If files are truly independent**: proceed in parallel, but each agent must:
- Work only in its own worktree
- Not modify any file outside its planned scope
- Not touch CLAUDE.md, serverless.yml, package.json, or navigation files (these are always shared)

### Error recovery

If any issue fails mid-work (type errors, test failures, agent crash):

1. **Clean up immediately** — remove the worktree, delete the local branch
2. **Do not leave debris** — no half-pushed branches, no orphan worktrees
3. **Report the failure** to the user with what went wrong
4. **Continue with remaining issues** — one failure doesn't block the others (in serial mode, the failed issue's changes never landed on master, so subsequent issues are unaffected)

```bash
# Cleanup on failure:
cd <main-repo-path>
git worktree remove .worktrees/issue-<number>-<slug> --force 2>/dev/null
git branch -D issue/<number>-<slug> 2>/dev/null
git worktree list  # verify clean
```

---

## Routing

```
Issue arrives → parse → check for existing PR
  │
  ├─ PR with review feedback → REVIEW RESPONSE path
  │
  └─ No PR → classify complexity
       ├─ Trivial/Small → SHORT path (4 steps)
       ├─ Medium → STANDARD path (6 steps)
       └─ Large → FULL path (superpowers pipeline)
```

---

## Phase 0: Parse and Route

**Always runs first, regardless of complexity.**

```bash
# 1. Check main repo state
git status --short
git branch --show-current   # Must be master — warn user if not
git worktree list

# 2. Read the issue — ALWAYS read comments, especially the most recent ones
gh issue view <number> --json title,body,labels,assignees,comments
```

**Read the most recent comments on the issue before doing anything else.** Issues may have been reopened, scope may have changed, prior work may be partially done. The latest comments are the ground truth — not the issue title or body alone.

```bash
# 3. Check for existing PR + reviews
gh pr list --head "issue/<number>" --json number,title,state,headRefName,reviews
```

If an open PR exists with review comments → go to **Review Response Path**.
Otherwise → classify and route.

### Classify Complexity

| Complexity | Signal | Examples |
|------------|--------|----------|
| **Trivial** | One-line change, config, rename | Fix typo, update constant, swap import |
| **Small** | Single-file logic, isolated bug | Fix filter, add validation check |
| **Medium** | Multi-file, new component, new endpoint | New screen, API + frontend integration |
| **Large** | New subsystem, architectural, security-sensitive | New report type with backend + frontend + LLM |

Tell the user: _"Classifying as **small** — single file change in the handler. Agree?"_

---

## SHORT Path (Trivial / Small)

### Step 1: Plan

Present a brief plan: what changes, which files, how to verify. Wait for sign-off.

Post the plan as a GitHub issue comment:
```bash
gh issue comment <number> --body-file <plan-file>
```

### Step 2: Worktree + Implement

```bash
# Ensure .worktrees is gitignored
git check-ignore -q .worktrees 2>/dev/null || echo ".worktrees" >> .gitignore

# Create worktree
git worktree add .worktrees/issue-<number>-<slug> -b issue/<number>-<slug>
cd .worktrees/issue-<number>-<slug>
```

Make the change. Commit:
```bash
git add <specific files>
git commit -m "fix: description

Part of #<number>"
```

**Never use `Closes #<number>`** unless the issue is fully resolved by this PR. Multi-phase issues, ongoing work, and issues with remaining TODOs should use `Part of #<number>` to avoid auto-closing.

### Step 3: Verify + Push + PR

```bash
npx tsc --noEmit
npm test
# If backend changed: cd backend && npx tsc --noEmit && npm test
```

**Check for untracked files your code depends on** — CI only sees committed files.

If clean:
```bash
git push -u origin issue/<number>-<slug>
gh pr create --head issue/<number>-<slug> --base master --title "Short title" --body-file <body>
```

Wait for CI:
```bash
gh pr checks <pr-number> --watch
```

Dispatch **1 review agent** (Haiku for trivial, Sonnet for small) with the issue text, changed files, and **worktree path**. Reviewer reads local files and answers: "Does this achieve what the issue asked? Any issues?"

### Step 4: Cleanup + Report

**Immediately after PR creation and review** (do not defer):
```bash
cd <main-repo-path>
git worktree remove .worktrees/issue-<number>-<slug>
git worktree list   # Verify only main repo remains
```

Report to user: PR URL, what was done, review result, whether issue auto-closes on merge.

---

## STANDARD Path (Medium)

### Step 1: Explore + Plan

Read the files that will be affected. Ask 1-3 clarifying questions if needed.

Propose approach with key trade-offs. Break into numbered tasks with files and verification. Wait for sign-off.

Post plan to issue:
```bash
gh issue comment <number> --body-file <plan-file>
```

### Step 2: Create Worktree

```bash
git check-ignore -q .worktrees 2>/dev/null || echo ".worktrees" >> .gitignore
git worktree add .worktrees/issue-<number>-<slug> -b issue/<number>-<slug>
cd .worktrees/issue-<number>-<slug>
npm install  # if needed
```

### Step 3: Implement

All work inside the worktree. Follow TDD when adding new behavior: failing test → make it pass → commit.

Use subagents only for genuinely independent sub-tasks. Pass the **worktree path** to any subagent — they must not create their own worktrees or branches.

### Step 4: Verify

**Invoke `superpowers:verification-before-completion`.**

```bash
npx tsc --noEmit
npm test
cd backend && npx tsc --noEmit && npm test  # if backend changed
git status  # check for untracked dependencies
```

No completion claims without fresh output evidence.

### Step 5: Push + PR + Review

```bash
git push -u origin issue/<number>-<slug>
gh pr create --head issue/<number>-<slug> --base master --title "Title" --body-file <body>
gh pr checks <pr-number> --watch
```

Dispatch **1 Opus review agent** with issue text, CLAUDE.md, changed file paths, and worktree path. Reviewer reads local files.

If reviewer finds issues: fix → re-verify → re-review.

### Step 6: Cleanup + Report

**Immediately** after PR + review:
```bash
cd <main-repo-path>
git worktree remove .worktrees/issue-<number>-<slug>
git worktree list
```

Report: PR URL, implementation summary, review result, CI status.

---

## FULL Path (Large)

### Step 1: Design

1. **Invoke `superpowers:brainstorming`** — full exploration, clarifying questions, 2-3 approaches, design doc
2. **Invoke `superpowers:writing-plans`** — detailed plan with TDD tasks, exact file paths, code blocks
3. Post plan to issue

### Step 2: Create Worktree

Same as STANDARD Step 2.

### Step 3: Implement

**Invoke `superpowers:subagent-driven-development`** to execute the plan.

All subagents work in the **same worktree**. Pass the worktree path explicitly. No subagent creates its own branch or worktree.

### Step 4: Verify

Same as STANDARD Step 4.

### Step 5: Push + PR + Review

Push and create PR as above.

**Invoke `superpowers:requesting-code-review`** — full review protocol.

If review finds issues: fix → re-verify → re-review.

### Step 6: Cleanup + Report

Same as STANDARD Step 6.

---

## Review Response Path

An open PR exists with review feedback. **Skip all planning and implementation steps.**

```bash
gh pr view <pr-number> --json reviews,comments
gh api repos/{owner}/{repo}/pulls/<pr-number>/comments
```

Triage each item:

| Category | Action |
|----------|--------|
| **Must fix** | Fix immediately |
| **Should fix** | Fix before merge |
| **Suggestion** | Ask user — fix or defer |
| **Incorrect** | Invoke `superpowers:receiving-code-review`, push back with evidence |

Find or recreate the worktree:
```bash
git worktree list | grep "issue-<number>"
# If gone, recreate WITHOUT -b (branch exists):
git worktree add .worktrees/issue-<number>-<slug> issue/<number>-<slug>
```

Fix → verify → push → comment on PR with summary → re-request review → cleanup → report.
