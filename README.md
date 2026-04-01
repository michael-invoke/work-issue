# work-issue

A Claude Code plugin that automates GitHub issue workflows. Give it an issue number and it handles the rest: reads the issue, classifies complexity, creates an isolated worktree, implements the fix/feature with effort scaled to complexity, runs verification and code review, then creates a PR.

## Installation

```bash
claude /plugin install michael-invoke/work-issue
```

Or install directly from the repo URL:

```bash
claude /plugin install https://github.com/michael-invoke/work-issue
```

## Usage

In Claude Code, just reference an issue:

```
/work-issue #42
```

Or more naturally:

```
work on issue 42
pick up #15
```

The skill triggers automatically when you reference a GitHub issue number.

## What it does

1. **Checks local state** - uncommitted work, stashes, current branch
2. **Reads the issue** - title, body, labels, comments via `gh`
3. **Understands existing code** - reads actual files that will be affected
4. **Classifies complexity** - Trivial / Small / Medium / Large
5. **Plans proportionally** - brief plan for small fixes, full brainstorming for large features
6. **Creates isolated worktree** - safe for parallel execution of multiple issues
7. **Implements** - directly for small issues, subagent-driven for large ones
8. **Verifies** - typecheck + tests before any completion claims
9. **Code reviews** - always by a separate agent, scaled to complexity
10. **Creates PR** - with issue linkage and test plan
11. **Waits for CI** - fixes failures if needed
12. **Finishes the branch** - merge, keep, or discard

## Dependencies

This plugin works best with the [superpowers](https://github.com/obra/superpowers) plugin installed, which provides the brainstorming, TDD, verification, and code review skills referenced for Medium/Large issues. For Trivial/Small issues, it works standalone.

## License

MIT
