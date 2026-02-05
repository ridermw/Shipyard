# Orchestration

This document specifies how Claude Code instances are invoked and coordinated. **External orchestration is required** — agents do not loop or exit on their own.

## Execution Model

### Key Insight: One-Shot Invocations

Claude Code agents are **one-shot processes**, not long-running daemons:

| Assumption in Early Specs | Reality |
|---------------------------|---------|
| Agents poll, loop, and exit gracefully | Each `claude -p` invocation handles one task, then terminates |
| Skills enforce constraints | Skills provide context; **hooks** enforce constraints |
| Agents manage their own lifecycle | External orchestration manages lifecycle via `--max-turns` |
| Agents self-monitor budget | `--max-budget-usd` enforces budget externally |

The workflow (poll → claim → execute → update) is correct, but the **execution** requires external orchestration.

### Claude Code CLI Flags

| Flag | Purpose | Recommended Value |
|------|---------|-------------------|
| `-p "prompt"` | Headless/print mode, no interactive UI | Required |
| `--max-turns N` | Force termination after N API round-trips | 10-25 depending on agent |
| `--max-budget-usd N` | Hard budget cap per invocation | 2.00-5.00 depending on agent |
| `--allowedTools "..."` | Pre-approve tools to avoid permission prompts | Agent-specific |
| `--no-session-persistence` | Don't save session for resume | Recommended |
| `--output-format json` | Structured output for parsing | Optional |

### Why `--max-turns` is Required

Claude Code does not have a native "exit" signal. Without `--max-turns`:

- An agent could loop indefinitely if it misunderstands the task
- Budget tracking happens after turns complete, not during
- There's no way to guarantee termination

With `--max-turns`:

- Agent terminates after N turns regardless of task state
- If task is incomplete, next invocation continues from GitHub state
- Predictable resource usage per invocation

## Orchestration Patterns

### Sequential Orchestration (Recommended for Phase 1)

Run agents one at a time. Simplest, safest, no race conditions.

```bash
#!/bin/bash
# shipyard-sequential.sh

set -e

# Run PM to process any new Discussions
claude -p "$(cat .shipyard/skills/pm/SKILL.md)

Poll GitHub Discussions in 'Planning' category with no status label.
If found: claim by adding status:planning, create Issues, close Discussion.
If none found: report no work available." \
  --max-turns 10 \
  --max-budget-usd 2.00 \
  --allowedTools "Bash(gh *)" \
  --no-session-persistence

# Run Coder to process ready Issues
claude -p "$(cat .shipyard/skills/coder/SKILL.md)

Poll GitHub Issues with status:ready label.
If found: claim via assignee, implement in worktree, create PR with status:review.
Also check for own PRs with status:changes-requested or status:blocked.
If none found: report no work available." \
  --max-turns 20 \
  --max-budget-usd 5.00 \
  --allowedTools "Bash(gh *),Bash(git *),Read,Edit,Write" \
  --no-session-persistence

# Run Reviewer to process PRs awaiting review
claude -p "$(cat .shipyard/skills/reviewer/SKILL.md)

Poll GitHub PRs with status:review label (exclude self-authored).
If found: claim via assignee, review code, approve or request changes.
If none found: report no work available." \
  --max-turns 15 \
  --max-budget-usd 3.00 \
  --allowedTools "Bash(gh *),Read" \
  --no-session-persistence

# Run merge for approved PRs
gh pr list --label status:approved --json number -q '.[].number' | \
  xargs -I {} gh pr merge {} --squash --auto
```

### Cron-Based Orchestration (Phase 2+)

Run the sequential script on a schedule:

```cron
# Run Shipyard every 5 minutes during business hours
*/5 9-17 * * 1-5 /path/to/shipyard-sequential.sh >> /var/log/shipyard.log 2>&1
```

### GitHub Actions Orchestration (Phase 2+)

Trigger on relevant GitHub events:

```yaml
# .github/workflows/shipyard.yml
name: Shipyard Orchestration

on:
  discussion:
    types: [created]
  issues:
    types: [labeled]
  pull_request:
    types: [labeled, synchronize]
  schedule:
    - cron: '*/5 * * * *'  # Every 5 minutes as fallback

jobs:
  pm:
    if: github.event_name == 'discussion' || github.event_name == 'schedule'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run PM Agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "..." --max-turns 10 --max-budget-usd 2.00

  coder:
    if: |
      (github.event_name == 'issues' && contains(github.event.label.name, 'status:ready')) ||
      (github.event_name == 'pull_request' && contains(github.event.label.name, 'status:changes-requested')) ||
      github.event_name == 'schedule'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Coder Agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "..." --max-turns 20 --max-budget-usd 5.00

  reviewer:
    if: |
      (github.event_name == 'pull_request' && contains(github.event.label.name, 'status:review')) ||
      github.event_name == 'schedule'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Reviewer Agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "..." --max-turns 15 --max-budget-usd 3.00
```

## Concurrent Execution

### Single-Instance Recommendation

For simplicity and safety, **run only one instance of each agent type at a time**:

| Configuration | Risk Level | Notes |
|---------------|------------|-------|
| One PM, one Coder, one Reviewer | Low | Recommended for most deployments |
| Multiple Coders | Medium | Safe (claim conflicts detected) but wasteful |
| Multiple Reviewers | Medium | Safe but reviews may conflict |
| Multiple PMs | High | Label-based claiming is vulnerable (see pm.md) |

### Why Concurrency is Safe but Wasteful

When two agents of the same type poll simultaneously:

```
Coder-A: Poll → sees Issue #100 unclaimed
Coder-B: Poll → sees Issue #100 unclaimed
Coder-A: Assigns self to #100
Coder-B: Assigns self to #100
Coder-A: Re-reads, sees [A, B] → conflict! Backs off
Coder-B: Re-reads, sees [A, B] → conflict! Backs off
Result: Item #100 remains unclaimed, both agents wasted a cycle
```

With backoff (see labels.md), one agent eventually wins. But this wastes compute.

## Budget Allocation

Recommended budget allocation per agent type:

| Agent | `--max-turns` | `--max-budget-usd` | Rationale |
|-------|---------------|-------------------|-----------|
| PM | 10 | 2.00 | Planning is mostly reading and writing text |
| Coder | 20-25 | 5.00 | Implementation requires more exploration and iteration |
| Reviewer | 15 | 3.00 | Code review is analysis-heavy |

### Budget Exhaustion Handling

If `--max-budget-usd` is reached mid-task:

1. Claude Code terminates the session
2. GitHub state reflects whatever was committed before termination
3. If claim was not released, stale detection (30 min) will clear it
4. Next invocation picks up based on GitHub state

**Recommendation**: Set budget slightly higher than expected to allow graceful completion. Monitor actual usage to tune.

## Failure Handling

### Max-Turns Reached Mid-Task

If `--max-turns` is reached before task completion:

1. Agent stops immediately (no graceful shutdown)
2. Claim remains on item (assignee still set)
3. After 30 minutes with no status update, stale claim is cleared
4. Item returns to ready state for next invocation

**Mitigation**: Set `--max-turns` high enough for typical tasks. Monitor for frequent stale claims.

### Agent Crash

Same handling as max-turns: stale claim detection recovers the item.

### GitHub API Errors

If GitHub API fails during an operation:

1. Agent may retry (Claude Code has some built-in retry logic)
2. If persistent failure, agent likely terminates
3. Stale claim detection handles recovery

**Recommendation**: Check GitHub status before running orchestration. Add retry logic in orchestration script.

## Rate Limits

All agents run under a single GitHub user account and share API quota.

### GitHub API Limits

- **REST API**: 5,000 requests/hour (authenticated)
- **GraphQL API**: 5,000 points/hour
- **Search API**: 30 requests/minute

### Mitigation

1. **Don't run agents too frequently**: 5-minute intervals are usually sufficient
2. **Monitor rate limit headers**: `X-RateLimit-Remaining`
3. **Implement backoff in orchestration script**:

```bash
# Check rate limit before running agents
remaining=$(gh api rate_limit --jq '.rate.remaining')
if [ "$remaining" -lt 100 ]; then
  echo "Rate limit low ($remaining), skipping this run"
  exit 0
fi
```

## Stale Detection

Stale claim detection **MUST** be enabled for any automated deployment. Without it, crashed agents leave items permanently stuck.

### Implementation Options

1. **Run at agent startup**: Each agent checks for stale claims before polling for new work
2. **Dedicated cleanup job**: Run `sy status --stale --fix` on a schedule
3. **GitHub Action**: Scheduled workflow that clears stale claims

### Recommended: Startup Check

Add to orchestration script:

```bash
# Clear stale claims before running agents
sy status --stale --fix

# Then run agents...
```

Or inline with `gh`:

```bash
# Find items with assignee but no recent activity
gh issue list --assignee @me --json number,updatedAt -q \
  '.[] | select((now - (.updatedAt | fromdateiso8601)) > 1800) | .number' | \
  xargs -I {} gh issue edit {} --remove-assignee @me
```

## Configuration

Add to `.shipyard/config.yaml`:

```yaml
# Orchestration settings
orchestration:
  # Agent execution limits
  agents:
    pm:
      max_turns: 10
      max_budget_usd: 2.00
      allowed_tools:
        - "Bash(gh *)"
    coder:
      max_turns: 20
      max_budget_usd: 5.00
      allowed_tools:
        - "Bash(gh *)"
        - "Bash(git *)"
        - "Read"
        - "Edit"
        - "Write"
    reviewer:
      max_turns: 15
      max_budget_usd: 3.00
      allowed_tools:
        - "Bash(gh *)"
        - "Read"

  # Stale claim settings
  stale_detection:
    enabled: true          # MUST be true for automation
    timeout_minutes: 30
    check_on_startup: true

  # Rate limiting
  rate_limit:
    min_remaining: 100     # Skip run if below this
    backoff_minutes: 5     # Wait time when rate limited
```

## Summary

| Spec Assumption | Correct Implementation |
|-----------------|----------------------|
| "Agents exit gracefully" | Use `--max-turns` to force termination |
| "Skills enforce constraints" | Use hooks for enforcement, skills for context |
| "Agents poll and loop" | One-shot invocations, external script loops |
| "Agents self-monitor budget" | Use `--max-budget-usd` for external enforcement |
| "Stale detection is optional" | Stale detection is **mandatory** for automation |

The core workflow design is sound. This document clarifies the execution model.
