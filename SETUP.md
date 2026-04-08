# Work-Issue + Dippy Setup Guide

## Overview

This repo contains the **work-issue** Claude Code skill and integrates with **Dippy** for safe command auto-approval. Together they provide a complete issue-to-PR workflow with guardrails.

## Components

### 1. Work-Issue Skill (`work-issue/skill.md`)

A Claude Code skill that turns GitHub issues into PRs with proportional effort:

- **Routing**: Issues are classified as Trivial/Small/Medium/Large and routed to SHORT (4 steps), STANDARD (6 steps), or FULL (superpowers pipeline) paths
- **Worktrees**: Each issue gets an isolated git worktree — main repo always stays on master
- **Serial by default**: Multiple issues are worked one at a time; parallel only with explicit user approval and file overlap analysis
- **Plan-first**: Implementation plans are posted to the GitHub issue before any code is written
- **Review-first**: Code review agents are dispatched immediately after PR creation, before waiting for CI
- **Cleanup**: Worktrees are removed immediately after PR creation

### 2. Dippy (`~/Dippy`)

[Dippy](https://github.com/ldayton/Dippy) is a shell command hook that auto-approves safe commands and prompts for destructive ones. Installed from source, runs via anaconda Python.

**Global config** (`~/.dippy/config`) allows:
- Shell builtins: `export`, `cd`, `echo`, `source`, `env`, etc.
- Package managers: `npm`, `npx`, `uv`, `pip`, `conda`
- Runtimes: `node`, `python`, `python3`
- File ops: `mv`, `cp`, `sed -i`, `ls`, `cat`, `find`, `mkdir`
- Git/GitHub: `git`, `gh`
- Test runners: `jest`, `vitest`, `mocha`
- Redirects: `.gitignore` only

**Still blocked by Dippy** (requires manual approval):
- `rm`, `rm -rf`
- Redirects to files (except `.gitignore`)
- Unknown/unrecognized commands

### 3. Claude Code Settings (`~/.claude/settings.json`)

**Environment**:
- PATH includes: Node.js, GitHub CLI, Anaconda (python3, pip, uv)

**Permissions**:
- `Bash(git:*)` and `Bash(gh:*)` auto-allowed (Dippy is the safety layer)

**Hooks**:

| Hook | Event | Trigger | Behavior |
|------|-------|---------|----------|
| Dippy | PreToolUse:Bash | All bash commands | Auto-approve safe, prompt for dangerous |
| Plan gate | PreToolUse:Bash | `git worktree add` | Reminds (non-blocking) to post plan to issue first |
| Review gate | PostToolUse:Bash | `gh pr create` | Injects reminder to dispatch code review agent immediately |

**Important**: Hooks are advisory (remind), not blocking. Agents won't stop on hook reminders.

## Known Issues

- **Dippy + Windows paths**: Paths ending in `\"` (e.g., `"C:\path\to\dir\"`) cause a parse error in Dippy's Parable parser. Workaround: use forward slashes in paths.
- **Settings reload**: Hook changes in `settings.json` may require opening `/hooks` or restarting the session to take effect in already-running agents.
- **No Haiku agents**: Never use Haiku model for subagents — use Sonnet minimum, Opus for complex work.

## File Locations

| File | Purpose |
|------|---------|
| `~/.claude/skills/work-issue/skill.md` | The work-issue skill |
| `~/.claude/settings.json` | Global Claude Code settings, hooks, PATH |
| `~/.dippy/config` | Dippy allow/deny rules |
| `~/Dippy/` | Dippy installation (cloned from GitHub) |
