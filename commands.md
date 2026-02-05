# Shipyard CLI

This document specifies the `shipyard` (alias: `sy`) command line interface.

## Overview

The CLI provides a convenient way to manage Shipyard orchestration. It wraps common operations like initializing repos, checking status, and running agents.

## Installation

```bash
# Future: npm install
npm install -g shipyard

# Or use npx
npx shipyard init

# Alias available
sy init
```

## Commands

### sy init

Initialize a repository for Shipyard.

```bash
sy init [options]

Options:
  --force       Overwrite existing configuration
  --no-labels   Skip creating GitHub labels
  --no-templates Skip creating issue/PR templates
```

**What it does**:
1. Creates `.shipyard/` directory structure
2. Creates default `config.yaml`
3. Creates GitHub labels (via `gh` CLI)
4. Creates issue/PR templates
5. Enables GitHub Discussions (provides instructions if manual steps needed)

**Example**:
```bash
cd my-repo
sy init

# Output:
# ✓ Created .shipyard/config.yaml
# ✓ Created 27 GitHub labels
# ✓ Created issue templates
# ✓ Created PR template
# 
# Next steps:
# 1. Enable Discussions in GitHub Settings
# 2. Create Discussion categories (see docs/github/discussions.md)
# 3. Run `sy status` to verify setup
```

### sy status

Show the current state of all work items and agents.

```bash
sy status [options]

Options:
  --agent <name>  Filter by agent (pm, coder, reviewer, owner, librarian)
  --json          Output as JSON
```

**Example**:
```bash
sy status

# Output:
# Shipyard Status
# ===============
#
# Discussions (Planning):
#   #42 "Better error messages"     status:planned
#   #45 "Search improvements"       status:needs-planning
#
# Issues (Ready for Coder):
#   #100 [FEATURE] Error messages   priority:high
#   #101 [BUG] Login timeout        priority:medium
#
# PRs (In Review):
#   #150 Implement error messages   status:ready-for-review
#
# PRs (Approved):
#   (none)
#
# Docs Pending:
#   #145 Auth flow update           docs:pending
```

### sy queue

Show work items available for a specific agent.

```bash
sy queue <agent>

Arguments:
  agent         Agent name: pm, coder, reviewer, owner, librarian
```

**Example**:
```bash
sy queue coder

# Output:
# Coder Queue
# ===========
#
# Ready for implementation:
#   #100 [FEATURE] Error messages   priority:high   Discussion:#42
#   #101 [BUG] Login timeout        priority:medium
#
# Awaiting fixes (changes requested):
#   #150 Error messages PR          priority:high   Comments:3
```

### sy run

Manually trigger an agent skill.

```bash
sy run <agent> [options]

Arguments:
  agent         Agent name: pm, coder, reviewer, owner, librarian

Options:
  --dry-run     Show what would happen without executing
  --item <id>   Process specific item (Discussion/Issue/PR number)
```

**Example**:
```bash
sy run coder

# Output:
# Starting Coder agent...
# Found 2 items in queue
# Processing: #100 [FEATURE] Error messages
# 
# [Claude Code skill executes]
#
# Completed: PR #150 created
# Remaining in queue: 1
```

```bash
sy run coder --dry-run

# Output:
# [DRY RUN] Coder would process:
# 1. #100 [FEATURE] Error messages (priority:high)
#    - Would create worktree
#    - Would implement based on Issue spec
#    - Would create PR linking to #100
```

### sy update

Update Shipyard skills to latest version.

```bash
sy update [options]

Options:
  --check       Check for updates without installing
  --force       Overwrite local skill modifications
```

**Example**:
```bash
sy update

# Output:
# Checking for updates...
# Current version: 0.1.0
# Latest version: 0.2.0
#
# Updates available:
#   - shipyard-pm: New planning templates
#   - shipyard-coder: Improved worktree handling
#   - shipyard-reviewer: Better code analysis
#
# Updating skills...
# ✓ Updated 3 skills
#
# Note: Local modifications in .shipyard/skills/ preserved
```

### sy logs

View recent agent activity.

```bash
sy logs [options]

Options:
  --agent <name>  Filter by agent
  --since <time>  Show logs since time (e.g., "1h", "1d")
  --follow        Stream new logs
```

**Example**:
```bash
sy logs --since 1h

# Output:
# Agent Logs (last 1 hour)
# ========================
#
# 14:30:22 [CODER] Claimed Issue #100
# 14:30:45 [CODER] Created worktree feature-100
# 14:45:12 [CODER] Created PR #150
# 14:45:15 [CODER] Completed Issue #100
# 15:00:01 [REVIEWER] Claimed PR #150
# 15:05:33 [REVIEWER] Requested changes on PR #150
```

### sy config

View or modify Shipyard configuration.

```bash
sy config [key] [value]

Arguments:
  key           Configuration key (dot notation)
  value         New value (omit to read current value)

Options:
  --list        List all configuration
```

**Example**:
```bash
sy config --list

# Output:
# Shipyard Configuration
# ======================
# 
# agents.pm.enabled: true
# agents.coder.enabled: true
# agents.reviewer.enabled: true
# agents.owner.enabled: true
# agents.librarian.enabled: true
# 
# budget.warning_threshold: 0.8
# budget.stop_threshold: 0.9
#
# polling.interval: 300
```

```bash
sy config budget.warning_threshold 0.7

# Output:
# Updated budget.warning_threshold: 0.8 → 0.7
```

### sy worktrees

Manage agent worktrees.

```bash
sy worktrees [subcommand]

Subcommands:
  list          List all Shipyard worktrees
  clean         Remove worktrees for merged PRs
  remove <name> Remove specific worktree
```

**Example**:
```bash
sy worktrees list

# Output:
# Shipyard Worktrees
# ==================
#
# feature-100      PR:#150 (open)      Created: 2 hours ago
# feature-95       PR:#145 (merged)    Created: 1 day ago
#
# Cleanup available: 1 worktree (merged PRs)
# Run `sy worktrees clean` to remove
```

## Configuration File

**File**: `.shipyard/config.yaml`

```yaml
# Shipyard Configuration
version: "1"

# Agent settings
agents:
  pm:
    enabled: true
    model: sonnet
  coder:
    enabled: true
    model: sonnet
  reviewer:
    enabled: true
    model: sonnet
  owner:
    enabled: true
    model: sonnet
  librarian:
    enabled: true
    model: sonnet

# Budget management
budget:
  warning_threshold: 0.8    # Warn at 80% usage
  stop_threshold: 0.9       # Stop at 90% usage

# Polling settings (future use)
polling:
  enabled: false
  interval: 300             # Seconds between polls

# Worktree settings
worktrees:
  directory: .worktrees
  cleanup_on_merge: true

# GitHub settings
github:
  discussions_enabled: true
  required_labels: true
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Configuration error |
| 3 | GitHub API error |
| 4 | No work available |
| 5 | Budget exceeded |

## Environment Variables

| Variable | Description |
|----------|-------------|
| `SHIPYARD_CONFIG` | Path to config file (default: `.shipyard/config.yaml`) |
| `SHIPYARD_DEBUG` | Enable debug output |
| `GITHUB_TOKEN` | GitHub auth (uses `gh` CLI auth by default) |

## Implementation Notes

The CLI is implemented as bash scripts wrapping:
1. `gh` CLI for GitHub operations
2. `git` for worktree management
3. `claude` CLI for running skills

Future versions may be rewritten in a compiled language for speed.
