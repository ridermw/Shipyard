# Hooks

This document specifies how Claude Code hooks enforce constraints that skills alone cannot.

## Skills vs Hooks

| Mechanism | Purpose | Enforcement |
|-----------|---------|-------------|
| **Skills** | Provide persona, context, and instructions | Soft — Claude follows instructions but can deviate |
| **Hooks** | Block or modify tool calls | Hard — hook runs before tool executes |

Skills tell Claude *what* to do. Hooks *prevent* Claude from doing forbidden things.

### Example

**Skill instruction** (soft):
> "Never approve your own PRs"

Claude will generally follow this, but prompt injection or confusion could cause deviation.

**Hook enforcement** (hard):
> Block any `gh pr review --approve` command

The tool call is intercepted and rejected before it executes.

## Hook Configuration

Hooks are configured in `.claude/settings.json` at the repository level:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "<tool-pattern>",
        "hooks": [
          {
            "type": "command",
            "command": "<shell-command>"
          }
        ]
      }
    ]
  }
}
```

### Hook Types

| Type | Behavior |
|------|----------|
| `PreToolUse` | Runs before a tool executes; can block the call |
| `PostToolUse` | Runs after a tool executes; can modify output |

For constraint enforcement, use `PreToolUse` with exit codes:
- Exit 0: Allow the tool call
- Exit 2: Block the tool call (shows error message to agent)

## Required Hooks

### 1. No Self-Approval

Agents must not approve their own PRs. This is a security constraint.

```json
{
  "matcher": "Bash(gh pr review * --approve *)",
  "hooks": [{
    "type": "command",
    "command": "echo '[HOOK] Blocked: agents cannot approve PRs via gh pr review --approve' >&2 && exit 2"
  }]
}
```

**Note**: This blocks all `gh pr review --approve` commands. For Shipyard, approvals are done by the Reviewer agent via `gh pr review --approve`, so this hook should only be active for agents that should never approve (e.g., Coder).

**Per-agent configuration**: Use separate `.claude/` directories or environment-based configuration to apply different hooks to different agents.

### 2. No Force Push to Main

Prevent accidental or malicious force pushes to protected branches.

```json
{
  "matcher": "Bash(git push * --force *)",
  "hooks": [{
    "type": "command",
    "command": "if echo \"$CLAUDE_TOOL_INPUT\" | grep -qE '(main|master)'; then echo '[HOOK] Blocked: force push to main/master' >&2 && exit 2; fi"
  }]
}
```

### 3. No Destructive Git Commands

Block commands that could lose work.

```json
{
  "matcher": "Bash(git reset --hard *)",
  "hooks": [{
    "type": "command",
    "command": "echo '[HOOK] Blocked: git reset --hard is destructive' >&2 && exit 2"
  }]
},
{
  "matcher": "Bash(git clean -f *)",
  "hooks": [{
    "type": "command",
    "command": "echo '[HOOK] Blocked: git clean -f is destructive' >&2 && exit 2"
  }]
}
```

### 4. Required Label Check

Ensure agents only modify items they've properly claimed.

```json
{
  "matcher": "Bash(gh issue edit *)",
  "hooks": [{
    "type": "command",
    "command": ".shipyard/hooks/verify-claim.sh"
  }]
},
{
  "matcher": "Bash(gh pr edit *)",
  "hooks": [{
    "type": "command",
    "command": ".shipyard/hooks/verify-claim.sh"
  }]
}
```

Where `.shipyard/hooks/verify-claim.sh`:

```bash
#!/bin/bash
# Verify the agent has claimed this item before modifying it

# Extract issue/PR number from command
NUMBER=$(echo "$CLAUDE_TOOL_INPUT" | grep -oE '\b[0-9]+\b' | head -1)

if [ -z "$NUMBER" ]; then
  exit 0  # Can't determine number, allow (fail open)
fi

# Check if we're assigned
ASSIGNEES=$(gh issue view "$NUMBER" --json assignees -q '.assignees[].login' 2>/dev/null || \
            gh pr view "$NUMBER" --json assignees -q '.assignees[].login' 2>/dev/null)

CURRENT_USER=$(gh api user -q '.login')

if echo "$ASSIGNEES" | grep -q "^$CURRENT_USER$"; then
  exit 0  # We're assigned, allow
else
  echo "[HOOK] Blocked: must claim item before modifying (not assigned to #$NUMBER)" >&2
  exit 2
fi
```

## Recommended Hooks

### 5. Prevent Committing Secrets

Block commits that might contain sensitive files.

```json
{
  "matcher": "Bash(git add *)",
  "hooks": [{
    "type": "command",
    "command": ".shipyard/hooks/check-secrets.sh"
  }]
}
```

Where `.shipyard/hooks/check-secrets.sh`:

```bash
#!/bin/bash
# Block adding files that likely contain secrets

BLOCKED_PATTERNS=".env credentials.json *.pem *.key id_rsa"

for pattern in $BLOCKED_PATTERNS; do
  if echo "$CLAUDE_TOOL_INPUT" | grep -q "$pattern"; then
    echo "[HOOK] Blocked: cannot add sensitive file matching '$pattern'" >&2
    exit 2
  fi
done

exit 0
```

### 6. Verify Ownership Before Force Push

Ensure agent still owns a PR before force-pushing rebased commits.

```json
{
  "matcher": "Bash(git push * --force*)",
  "hooks": [{
    "type": "command",
    "command": ".shipyard/hooks/verify-pr-ownership.sh"
  }]
}
```

Where `.shipyard/hooks/verify-pr-ownership.sh`:

```bash
#!/bin/bash
# Verify we still own the PR before force-pushing

BRANCH=$(git branch --show-current)
PR_NUMBER=$(gh pr list --head "$BRANCH" --json number -q '.[0].number')

if [ -z "$PR_NUMBER" ]; then
  exit 0  # No PR for this branch, allow
fi

ASSIGNEES=$(gh pr view "$PR_NUMBER" --json assignees -q '.assignees[].login')
CURRENT_USER=$(gh api user -q '.login')

if echo "$ASSIGNEES" | grep -q "^$CURRENT_USER$"; then
  exit 0  # We're assigned, allow force push
else
  echo "[HOOK] Blocked: cannot force push to PR #$PR_NUMBER (not assigned)" >&2
  exit 2
fi
```

## Complete Hook Configuration

Full `.claude/settings.json` for Shipyard:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git push * --force *)",
        "hooks": [{
          "type": "command",
          "command": "if echo \"$CLAUDE_TOOL_INPUT\" | grep -qE 'origin.*(main|master)'; then echo '[HOOK] Blocked: force push to main/master' >&2 && exit 2; fi && .shipyard/hooks/verify-pr-ownership.sh"
        }]
      },
      {
        "matcher": "Bash(git reset --hard *)",
        "hooks": [{
          "type": "command",
          "command": "echo '[HOOK] Blocked: git reset --hard is destructive' >&2 && exit 2"
        }]
      },
      {
        "matcher": "Bash(git clean -f *)",
        "hooks": [{
          "type": "command",
          "command": "echo '[HOOK] Blocked: git clean -f is destructive' >&2 && exit 2"
        }]
      },
      {
        "matcher": "Bash(gh issue edit *)",
        "hooks": [{
          "type": "command",
          "command": ".shipyard/hooks/verify-claim.sh"
        }]
      },
      {
        "matcher": "Bash(gh pr edit *)",
        "hooks": [{
          "type": "command",
          "command": ".shipyard/hooks/verify-claim.sh"
        }]
      },
      {
        "matcher": "Bash(git add *)",
        "hooks": [{
          "type": "command",
          "command": ".shipyard/hooks/check-secrets.sh"
        }]
      }
    ]
  }
}
```

## Per-Agent Hooks

Different agents need different constraints:

| Hook | PM | Coder | Reviewer |
|------|----|----|----------|
| No self-approval | N/A | Yes | N/A (Reviewer approves) |
| No force push to main | Yes | Yes | Yes |
| No destructive git | Yes | Yes | Yes |
| Verify claim | Yes | Yes | Yes |
| Check secrets | N/A | Yes | N/A |

To apply different hooks per agent:

### Option 1: Separate Config Directories

```
.claude/
├── settings.json          # Shared settings
├── pm/
│   └── settings.json      # PM-specific hooks
├── coder/
│   └── settings.json      # Coder-specific hooks
└── reviewer/
    └── settings.json      # Reviewer-specific hooks
```

Run agents with `CLAUDE_CONFIG_DIR`:

```bash
CLAUDE_CONFIG_DIR=.claude/coder claude -p "..."
```

### Option 2: Environment-Based Hooks

Use environment variables in hook scripts:

```bash
#!/bin/bash
# verify-claim.sh

if [ "$SHIPYARD_AGENT" = "reviewer" ]; then
  # Reviewer uses different claim logic (reviewer field, not assignee)
  ...
fi
```

## Hook Development Tips

### Testing Hooks

Test hooks locally before deployment:

```bash
# Simulate a blocked command
CLAUDE_TOOL_INPUT="gh pr review 123 --approve" .shipyard/hooks/no-self-approve.sh
echo "Exit code: $?"
```

### Debugging Hooks

Add logging to hooks:

```bash
#!/bin/bash
echo "[DEBUG] Hook triggered: $0" >> /tmp/shipyard-hooks.log
echo "[DEBUG] Input: $CLAUDE_TOOL_INPUT" >> /tmp/shipyard-hooks.log

# ... hook logic ...
```

### Hook Performance

Hooks run synchronously before each tool call. Keep them fast:

- Avoid network calls when possible
- Cache GitHub API responses
- Use `exit 0` early for allowed cases

## Limitations

### What Hooks Cannot Do

1. **Enforce conversation-level rules**: Hooks see individual tool calls, not conversation history
2. **Prevent all prompt injection**: A sufficiently clever injection might avoid tool calls
3. **Replace human oversight**: Hooks are defense-in-depth, not a complete solution

### What Skills Should Still Do

Even with hooks, skills should include constraint instructions:

- Hooks catch mistakes; skills prevent them
- Clear instructions reduce hook triggers
- Some constraints are too complex for hooks (e.g., "only modify files related to the Issue")

## Summary

| Layer | Purpose | Example |
|-------|---------|---------|
| **Skills** | Context and instructions | "You are Coder. Implement Issues." |
| **Hooks** | Hard enforcement | Block `gh pr review --approve` |
| **Branch Protection** | Final safety net | Require review before merge |

All three layers work together. Skills guide behavior, hooks prevent mistakes, branch protection catches anything that slips through.
