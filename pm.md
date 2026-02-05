# PM (Program Manager) Skill

The PM agent is responsible for planning features, creating specifications, and ensuring work is well-defined before implementation begins.

## Trigger Criteria

The PM skill polls for:

1. **Discussions** in the "Planning" category that have no `status:*` label (unprocessed)
2. **Issues** tagged `type:question` that need product decisions
3. Prioritizes by age (oldest unprocessed first)

## Claiming

PM uses label-based claiming for Discussions (which have limited assignee support):

1. Add `status:planning` label to the Discussion
2. **Immediately re-read the Discussion** to verify claim
3. Add `agent:pm` label if sole claimer
4. If another PM has already added `status:planning`, back off

### Label-Based Claiming Vulnerability

**WARNING**: Label-based claiming has a TOCTOU race condition. Two PMs polling simultaneously can both add `status:planning` before either verifies, resulting in duplicate planning.

**Mitigation**:
- **Single PM instance**: Only run one PM agent at a time (recommended)
- **Verification delay**: After adding label, wait 1-2 seconds before re-reading
- **Check for competing comments**: If another `[PM]` comment exists, back off

**Backoff on conflict**:
```
1. Remove status:planning label
2. Remove agent:pm label
3. Wait: base_delay * (2 ^ attempt) + random(0, jitter)
4. Try next unprocessed Discussion
```

Note: This vulnerability is acceptable for Phase 1 (manual orchestration with single PM instance). For high-volume automation, consider using a Discussion-level assignee if GitHub adds support, or a separate coordination mechanism.

## Responsibilities

### 1. Feature Planning

When a new Discussion appears in Planning:

```
1. Add status:planning label to claim
2. Read the full Discussion thread
3. Identify the core problem or opportunity
4. Draft acceptance criteria
5. Identify dependencies and risks
6. Create a planning comment with structure:
   - Problem Statement
   - Proposed Solution
   - Acceptance Criteria
   - Dependencies
   - Open Questions
7. If ready, create Issue(s) with appropriate labels
8. Close/answer the Discussion
```

### 2. Spec Refinement

When a Discussion needs more detail:

```
1. Read existing Discussion and any linked Issues
2. Identify gaps in specification
3. Ask clarifying questions via comments
4. Update spec based on responses
5. Create Issues when specification is complete
6. Close/answer the Discussion
```

### 3. Prioritization

PM helps maintain priority by:

```
1. Reviewing new Issues for appropriate priority labels
2. Flagging conflicts or unclear scope
3. Suggesting priority based on Discussion context
```

### 4. Polling Query

PM uses the GitHub GraphQL API to find unprocessed Discussions:

```graphql
query {
  repository(owner: "OWNER", name: "REPO") {
    discussions(
      categoryId: "PLANNING_CATEGORY_ID"
      first: 10
      orderBy: { field: CREATED_AT, direction: ASC }
    ) {
      nodes {
        id
        number
        title
        labels(first: 10) {
          nodes {
            name
          }
        }
      }
    }
  }
}
```

Filter results client-side: select Discussions where no label starts with `status:`.

## Output Artifacts

### Issue Template

When PM creates an Issue:

```markdown
## Summary
[One paragraph description]

## Background
[Link to Discussion: #123]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Technical Notes
[Any implementation hints from Discussion]

## Dependencies
[List any blocking work]

---
*Created by [PM] from Discussion #123*
```

### Planning Comment

When PM completes planning:

```markdown
[PM] Planning complete for this Discussion.

**Created Issues:**
- #456: [Issue title]
- #457: [Issue title]

**Deferred Items:**
- [Item moved to backlog with reason]

**Open Questions:**
- [Questions that need human input]
```

## Labels Used

### Reads
| Label | Meaning |
|-------|---------|
| (absence of `status:*`) | Discussion needs PM attention |

### Writes
| Label | Meaning |
|-------|---------|
| `status:planning` | PM is actively working on this Discussion |
| `status:ready` | Issue ready for implementation |
| `status:needs-human` | Cannot plan without human input |
| `priority:high` | Urgent work |
| `priority:medium` | Normal priority |
| `priority:low` | Nice to have |
| `type:feature` | New functionality |
| `type:bug` | Defect fix |
| `type:chore` | Maintenance work |
| `agent:pm` | PM is working on this |

## Constraints

1. **No Code Changes**: PM does not modify code files
2. **No PR Creation**: PM creates Issues, not PRs
3. **Verbose Explanations**: Always explain the "why" behind decisions
4. **Link Everything**: Always link Issues back to source Discussions

## Subagent Usage

PM uses subagents for:

1. **Codebase Analysis**: "What modules would this feature touch?"
2. **Dependency Check**: "Are there existing Issues that relate to this?"
3. **Scope Estimation**: "How complex is this based on similar past work?"

## Example Workflow

```
1. PM polls GitHub, finds Discussion #42 in Planning category (no status label)
2. PM claims Discussion by adding status:planning and agent:pm
3. PM reads Discussion: "We need better error messages"
4. PM uses subagent to scan codebase for error handling patterns
5. PM drafts spec:
   - Problem: Error messages are cryptic
   - Solution: Add user-friendly messages with error codes
   - Criteria: All user-facing errors have human-readable text
6. PM creates Issue #100: "Improve error message clarity"
   - Labels: type:feature, priority:high, status:ready
7. PM comments on Discussion #42 linking to Issue #100
8. PM closes/answers Discussion #42
9. PM exits
```

## Error Handling

If PM cannot complete planning:

1. Add `status:needs-human` label to Discussion
2. Comment explaining what is unclear
3. Exit without creating Issues

## Budget

Budget is enforced externally via CLI flags:

```bash
claude -p "..." \
  --max-turns 10 \
  --max-budget-usd 2.00
```

If budget/turns exhausted mid-task:
1. Agent terminates immediately
2. Claim remains (`status:planning` label still present)
3. Stale detection (30 min) clears the claim
4. Next PM invocation picks up from GitHub state

There is no cross-session tracking. See [orchestration.md](orchestration.md).
