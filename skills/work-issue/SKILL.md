---
name: work-issue
description: Use when the user wants to work on a GitHub issue, references an issue number like "#42" or "issue 42", or says "/work-issue". Reads the issue, creates a branch in an isolated worktree, implements the fix/feature, creates a PR, and runs a proportional code review. Use this skill even if the user just pastes a GitHub issue URL or says something like "pick up issue 42" or "work on #15".
---

# Work Issue

GitHub issue -> classify -> isolate -> implement -> verify -> PR -> review -> finish. Scale effort to complexity. Safe for parallel execution.

## Ground Rules

- **Local filesystem is the source of truth.** Always read actual files, never assume state from GitHub diffs alone.
- **Check for uncommitted work first.** Before creating branches, run `git status`. Uncommitted files may be relevant. Ask the user about anything unexpected.
- **One branch per issue unless the user says otherwise.** Multiple items in one issue = one branch, one PR.
- **Worktree isolation is mandatory.** Every issue gets its own worktree so multiple `/work-issue` invocations can run in parallel without branch collisions.
- **Scale effort to complexity.** A one-line bug fix does not need a design doc. A new feature does.

## Step 1: Check Local State

Before anything else:

```bash
git status --short
git branch --show-current
git stash list
```

If there are uncommitted changes, untracked files, or stashes -- **tell the user** and ask how to proceed. Do not silently ignore them.

## Step 2: Parse and Read the Issue

Extract the issue number (accepts `42`, `#42`, `issue 42`, or a GitHub URL).

```bash
gh issue view <number> --json title,body,labels,assignees,comments
```

Summarize for the user: title, what's being asked, any prior discussion or existing plan comments.

## Step 3: Understand Existing Code

Before planning, **read the actual files** that will be affected. Use Glob/Grep/Read to understand current state. If the issue mentions a feature, check whether it already exists (partially or fully) in the local codebase -- including uncommitted or untracked files.

Do not assume the codebase matches what's on GitHub. Local state may be ahead.

## Step 4: Classify Complexity

Assess the issue and classify it. This determines the entire downstream workflow.

| Complexity | Signal | Examples |
|------------|--------|----------|
| **Trivial** | Config change, rename, reorder, one-line fix | Fix a typo, update a constant, swap an import |
| **Small** | Single-file logic change, isolated bug fix | Fix a filter expression, add a validation check |
| **Medium** | Multi-file feature, new component, cross-cutting change | Add a new screen, new API endpoint + frontend integration |
| **Large** | Architectural change, new subsystem, security-sensitive | New report type with backend + frontend + LLM + storage |

Present your classification to the user: _"I'd classify this as **medium** -- it touches the handler, store, and a new UI component. Sound right?"_

Adjust if the user disagrees.

## Step 5: Plan (scaled to complexity)

### Trivial / Small

Skip brainstorming and design docs. Present a brief plan directly:

- "Here's my read -- does this match your intent?"
- What the fix/change is
- Which file(s) will be touched
- How to verify

Wait for sign-off. Then **post the plan as a comment on the GitHub issue** before implementing:

```bash
gh issue comment <number> --body-file <plan-file>
```

### Medium

Lightweight brainstorming -- no design doc, but structured planning:

1. **Explore the problem space** -- ask 1-3 clarifying questions if needed (not the full brainstorming interview)
2. **Propose approach** -- present your recommended approach with key trade-offs
3. **Draft plan** -- break into numbered tasks with files, changes, and verification steps
4. Wait for user sign-off
5. **Post plan to issue** as a public record

### Large

Invoke the full superpowers pipeline:

1. **Invoke `superpowers:brainstorming`** -- full design exploration, clarifying questions one at a time, 2-3 approaches with trade-offs, design doc written and committed
2. **Invoke `superpowers:writing-plans`** -- detailed implementation plan with bite-sized TDD tasks, exact file paths, code blocks, no placeholders
3. **Post plan to issue** as a public record (in addition to the spec/plan docs)

**Important**: The brainstorming skill will naturally flow into writing-plans. Let it run its full process. Your job is to provide the GitHub issue context that superpowers doesn't know about.

## Step 6: Create Isolated Worktree

**Invoke `superpowers:using-git-worktrees`** to create an isolated workspace.

The branch name should follow the convention: `issue/<number>-<short-slug>`

The slug should be 2-4 words from the issue title, kebab-cased.

Example: issue #15 "River Forecast Digest" -> `issue/15-river-forecast-digest`

**Why worktrees are mandatory**: Multiple `/work-issue` invocations may run in parallel. Without worktree isolation, agents switch branches out from under each other. Each worktree gets its own working directory and branch, so there are zero collisions.

## Step 7: Implement (scaled to complexity)

### Trivial / Small

Implement directly in the worktree. No subagents needed.

- Make the change
- Run verification (Step 8)
- Commit

### Medium

Implement directly or use subagents for genuinely independent sub-tasks. Use your judgment -- don't over-engineer the process for a straightforward multi-file change.

Follow **TDD** when adding new behavior: write the failing test first, make it pass, then commit.

### Large

Use **`superpowers:subagent-driven-development`** to execute the plan:

- Fresh subagent per task
- Two-stage review after each task (spec compliance, then code quality)
- Model selection: cheap models for mechanical tasks, capable models for judgment tasks
- The plan from Step 5 drives the entire execution

**Important**: Subagent-driven-development already incorporates worktree setup. Since you created the worktree in Step 6, pass the worktree path to the subagents -- do not let them create a second worktree.

## Step 8: Verify Before Completing

**Invoke `superpowers:verification-before-completion`** -- no exceptions, regardless of complexity.

Every branch must pass before committing:

```bash
npx tsc --noEmit
npm test
# If backend files changed:
cd backend && npx tsc --noEmit && npm test
```

**Critical: local tsc sees ALL local files but CI only sees committed files.** Before trusting a clean typecheck:
1. Run `git status` -- check for untracked/uncommitted files your code might depend on
2. Verify every import references a file that either (a) exists on master or (b) is staged in your commit
3. If you added new files, make sure they're staged

**No completion claims without fresh verification evidence.** Run the command, read the output, THEN claim the result.

## Step 9: Code Review (always a separate agent)

**You wrote the code, so you cannot review it.** Dispatch a separate agent for every review.

Scale the reviewer to the complexity:

**Trivial**: 1 Haiku agent. Give it the issue text, changed files, and ask: "Read the local files. Does this change achieve what the issue asked for? Any import or type errors? 2-3 sentences."

**Small**: 1 Sonnet agent. Same prompt plus: "Check for missed edge cases or state that wasn't updated."

**Medium**: 1 Opus agent for semantic review. Give it the issue text, CLAUDE.md, and changed file paths. It must read the actual local files.

**Large**: **Invoke `superpowers:requesting-code-review`** -- full review protocol with spec compliance and code quality gates.

**All review agents must:**
- Read the actual local files (not GitHub diffs)
- Be given the original issue text so they know the intent
- Answer: "Does this change correctly achieve what the issue asked for?"
- Flag anything that looks wrong, including files that might be missing from the commit

**If reviewer finds issues**: Fix them, re-verify (Step 8), re-review. Do not skip the re-review.

## Step 10: Commit, Push, Create PR

```bash
git add <specific files>
git commit -m "feat: description here

Closes #<number>"
git push -u origin issue/<number>-<short-slug>
```

Create PR:

```bash
gh pr create \
  --title "Short title" \
  --body-file <pr-body-file>
```

PR body should include:
- Summary (1-3 bullets)
- `Closes #<number>`
- Test plan checklist

## Step 11: Wait for CI

```bash
gh pr checks <pr-number>
```

If CI fails, read the failed logs (`gh run view <id> --log-failed`), fix, re-verify, and push again. Do not proceed until CI is green.

## Step 12: Finish the Branch

**Invoke `superpowers:finishing-a-development-branch`** -- presents structured options:

1. Merge back to master locally
2. Push and create a Pull Request (if not already done)
3. Keep the branch as-is
4. Discard this work

This skill also handles worktree cleanup.

## Step 13: Report Back

Tell the user:
- PR URL (or merge commit if merged locally)
- What was implemented
- Review result (pass/fail, any issues found and fixed)
- Whether the issue will auto-close on merge

---

## Complexity Routing Summary

```
Issue arrives
  |
  +- Trivial/Small
  |    Plan briefly -> Worktree -> Implement directly -> Verify -> Review (Haiku/Sonnet) -> PR -> Finish
  |
  +- Medium
  |    Lightweight brainstorm -> Worktree -> Implement (TDD) -> Verify -> Review (Opus) -> PR -> Finish
  |
  +- Large
       superpowers:brainstorming -> superpowers:writing-plans -> Worktree
         -> superpowers:subagent-driven-development -> Verify -> superpowers:requesting-code-review
         -> PR -> superpowers:finishing-a-development-branch
```

## Superpowers Integration Map

| Step | Superpowers Skill | When Used |
|------|-------------------|-----------|
| Plan (large) | `superpowers:brainstorming` | Large issues only |
| Plan (large) | `superpowers:writing-plans` | Large issues only |
| Isolate | `superpowers:using-git-worktrees` | **Always** |
| Implement (large) | `superpowers:subagent-driven-development` | Large issues only |
| Verify | `superpowers:verification-before-completion` | **Always** |
| Review (large) | `superpowers:requesting-code-review` | Large issues only |
| Finish | `superpowers:finishing-a-development-branch` | **Always** |

## Parallel Execution

Multiple `/work-issue` invocations are safe to run simultaneously because:

1. Each gets its own **worktree** (isolated working directory + branch)
2. No shared mutable state between agents
3. Each agent commits to its own branch
4. PRs are created independently
5. Merge conflicts (if any) are resolved at PR merge time

To work multiple issues in parallel, the user dispatches each as a separate agent with `isolation: "worktree"`:

```
Agent 1 -> /work-issue #14 -> worktree: .worktrees/issue-14-test-coverage
Agent 2 -> /work-issue #15 -> worktree: .worktrees/issue-15-river-digest
Agent 3 -> /work-issue #13 -> worktree: .worktrees/issue-13-accessibility
```
