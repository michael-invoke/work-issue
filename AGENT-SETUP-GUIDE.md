# Agent Setup Guide: work-issue + Dippy + Hooks

Run these steps on a fresh machine to replicate the full Claude Code workflow. This guide is written for an AI agent to execute — paste the whole thing or run step by step.

## Prerequisites

- **Claude Code** installed and authenticated
- **Python 3.9+** (Anaconda, miniconda, or system Python)
- **Node.js** (with npm/npx)
- **GitHub CLI** (`gh`) installed and authenticated
- **Git** configured

## Step 1: Install the work-issue skill

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/michael-invoke/work-issue.git tmp-clone
mv tmp-clone/work-issue ./work-issue
rm -rf tmp-clone
```

Or if setting up the full repo:
```bash
cd ~/.claude/skills
git init
git remote add origin https://github.com/michael-invoke/work-issue.git
git fetch origin
git checkout -b main origin/main
```

## Step 2: Install Dippy

```bash
cd ~
git clone https://github.com/ldayton/Dippy.git
```

If you have `uv`:
```bash
cd ~/Dippy && uv sync
```

If not, install uv first:
```bash
pip install uv
cd ~/Dippy && uv sync
```

### Verify Python can run Dippy

Find your Python path. Test with:
```bash
<YOUR_PYTHON3> ~/Dippy/bin/dippy-hook <<< '{"tool_name":"Bash","tool_input":{"command":"git status"}}'
```

You should see: `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "allow", ...}}`

**Windows note**: The `python3` command may be hijacked by the Windows Store app alias. Either:
- Disable it in Settings > Apps > Advanced app settings > App execution aliases
- Or copy `python.exe` to `python3.exe` in your Python directory

## Step 3: Create Dippy config

Write this to `~/.dippy/config`:

```
# Shell builtins
allow export
allow source
allow cd
allow pwd
allow echo
allow printf
allow set
allow unset
allow alias
allow which
allow where
allow type
allow true
allow false
allow test
allow [
allow sleep
allow wait
allow nohup
allow env

# Package managers & runtimes
allow npm install
allow npm ci
allow npm run
allow npm test
allow npx
allow npx tsc
allow npx test
allow node
allow uv
allow pip install
allow conda
allow python
allow python3

# File operations (mv/cp allowed, rm is NOT)
allow mv
allow sed -i
allow cp
allow ls
allow dir
allow cat
allow head
allow tail
allow wc
allow sort
allow uniq
allow find
allow mkdir

# Git & GitHub
allow git
allow gh

# Test runners
allow jest
allow vitest
allow mocha

# GIS tools
allow ogr2ogr
allow ogrinfo
allow gdal
allow gdalinfo
allow gdalwarp
allow mapshaper
allow tippecanoe

# Safe redirects
allow-redirect .gitignore
```

Customize by adding `allow <command>` for tools specific to your project. Add `deny <command> "reason"` to block with a message that steers Claude.

## Step 4: Configure Claude Code settings

Write this to `~/.claude/settings.json`. **Adapt paths** to match your machine.

```json
{
  "env": {
    "PATH": "<NODE_DIR>;<GH_CLI_DIR>;<PYTHON_DIR>;<PYTHON_SCRIPTS_DIR>;${PATH}"
  },
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(gh:*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "<PYTHON3_PATH> <HOME>/Dippy/bin/dippy-hook"
          },
          {
            "type": "command",
            "if": "Bash(git worktree add:*)",
            "command": "echo '{\"continue\":true,\"hookSpecificOutput\":{\"hookEventName\":\"PreToolUse\",\"permissionDecision\":\"allow\",\"permissionDecisionReason\":\"REMINDER: Post the implementation plan to the GitHub issue before proceeding with implementation.\"}}'",
            "statusMessage": "Checking plan gate..."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(gh pr create:*)",
            "command": "echo '{\"continue\":true,\"hookSpecificOutput\":{\"hookEventName\":\"PostToolUse\",\"additionalContext\":\"REMINDER: A PR was just created. You MUST dispatch a code review agent immediately. Do not wait for CI — launch the review now.\"}}'",
            "statusMessage": "Checking PR review gate..."
          }
        ]
      }
    ]
  }
}
```

### Path placeholders to replace

| Placeholder | Example (Windows/Anaconda) | Example (macOS/Homebrew) |
|---|---|---|
| `<NODE_DIR>` | `C:\\Program Files\\nodejs` | `/usr/local/bin` (usually already on PATH) |
| `<GH_CLI_DIR>` | `C:\\Program Files\\GitHub CLI` | `/usr/local/bin` |
| `<PYTHON_DIR>` | `C:\\Users\\you\\anaconda3` | `/usr/local/bin` |
| `<PYTHON_SCRIPTS_DIR>` | `C:\\Users\\you\\anaconda3\\Scripts` | (not needed) |
| `<PYTHON3_PATH>` | `C:/Users/you/anaconda3/python3.exe` | `python3` |
| `<HOME>` | `C:/Users/you` | `/Users/you` or `~` |

**Merge** with any existing settings — don't overwrite plugins, model, or other config.

## Step 5: Verify

### Test Dippy hook
```bash
echo '{"tool_name":"Bash","tool_input":{"command":"git status"}}' | <PYTHON3_PATH> ~/Dippy/bin/dippy-hook
# Expected: permissionDecision "allow"

echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' | <PYTHON3_PATH> ~/Dippy/bin/dippy-hook
# Expected: permissionDecision "ask"
```

### Test settings.json
```bash
node -e "JSON.parse(require('fs').readFileSync(require('os').homedir()+'/.claude/settings.json','utf8')); console.log('Valid JSON')"
```

### Test skill is loaded
Start a new Claude Code session and check that `/work-issue` appears as an available skill.

## Step 6: Per-project settings (IMPORTANT)

**Do NOT put hooks in project-level `.claude/settings.json`**. Hooks belong in the user-level file only (`~/.claude/settings.json`). Project settings override user settings — duplicate hooks in both will cause double-firing, and stale project hooks will override your fixes.

Project `.claude/settings.json` should only contain:
- Project-specific permissions
- Project-specific env vars

```json
{
  "permissions": {
    "allow": ["Read", "Edit", "Write", "Glob", "Grep", "Agent"]
  },
  "hooks": {}
}
```

## How It All Works Together

```
User says "#42" or "/work-issue"
  │
  ▼
work-issue skill activates
  │
  ├─ Classifies issue complexity (trivial → large)
  ├─ Posts implementation plan to GitHub issue
  ├─ Creates git worktree (plan-gate reminder fires)
  ├─ Implements in worktree
  ├─ Verifies (tsc, tests)
  ├─ Creates PR (review-gate reminder fires)
  ├─ Dispatches Sonnet/Opus review agent immediately
  ├─ Checks CI after review
  └─ Cleans up worktree
  │
  ▼
Every Bash command passes through Dippy first
  │
  ├─ Safe command (git, ls, npm) → auto-approved
  ├─ Dangerous command (rm -rf) → prompts user
  └─ Unknown command → prompts user
```

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Hooks block instead of remind | Old prompt hooks in project settings | Clear project `hooks: {}`, use user-level only |
| Hooks fire on wrong commands | `type: "prompt"` ignores `if` filter | Use `type: "command"` hooks only |
| `python3: command not found` | Windows Store alias intercepts | Copy `python.exe` → `python3.exe` or disable alias |
| `gh: command not found` | Not on PATH in Claude's shell | Add to `env.PATH` in settings.json |
| `npx: command not found` | Node.js not on PATH | Add Node.js dir to `env.PATH` |
| Dippy blocks safe command | Not in `~/.dippy/config` | Add `allow <command>` |
| Dippy parse error on Windows path | Trailing `\"` confuses parser | Use forward slashes in paths |
| Settings changes not taking effect | Session loaded old settings | Restart session or open `/hooks` |

## Key Rules

- **Never use Haiku agents** — Sonnet minimum, Opus for complex work
- **Review agents dispatch immediately** after PR creation, before CI
- **Hooks are advisory only** — `"continue": true`, never block
- **Serial issue workflow by default** — one at a time, merge before next
- **Worktrees only** — never switch branches in main repo
