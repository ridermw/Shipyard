# PM (Program Manager) Skill

The PM agent is responsible for planning features, creating specifications, and ensuring work is well defined before implementation begins.

## Trigger Criteria

The PM skill polls for:

1. **Discussions** in the "Planning" category without `status:planned` label
2. **Discussions** with `status:needs-refinement` label
3. **Issues** tagged `type:question` that need product decisions

## Responsibilities

### 1. Feature Planning

When a new Discussion appears in Planning:

```
1. Read the full Discussion thread
2. Identify the core problem or opportunity
3. Draft acceptance criteria
4. Identify dependencies and risks
5. Create a planning comment with structure:
   - Problem Statement
   - Proposed Solution
   - Acceptance Criteria
   - Dependencies
   - Open Questions
6. If ready, create Issue(s) with appropriate labels
7. Add status:planned to Discussion
```

### 2. Spec Refinement

When a Discussion needs more detail:

```
1. Read existing Discussion and linked Issues
2. Identify gaps in specification
3. Ask clarifying questions via comments
4. Update spec based on responses
5. Move to status:planned when complete
```

### 3. Prioritization

PM helps maintain priority by:

```
1. Reviewing new Issues for appropriate priority labels
2. Flagging conflicts or unclear scope
3. Suggesting priority based on Discussion context
```

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
| `status:needs-planning` | Discussion needs PM attention |
| `status:needs-refinement` | Spec needs more detail |

### Writes
| Label | Meaning |
|-------|---------|
| `status:planned` | Planning complete |
| `status:ready-for-coder` | Issue ready for implementation |
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
1. PM polls GitHub, finds Discussion #42 in Planning category
2. PM reads Discussion: "We need better error messages"
3. PM uses subagent to scan codebase for error handling patterns
4. PM drafts spec:
   - Problem: Error messages are cryptic
   - Solution: Add user friendly messages with error codes
   - Criteria: All user facing errors have human readable text
5. PM creates Issue #100: "Improve error message clarity"
6. PM comments on Discussion #42 linking to Issue #100
7. PM adds status:planned to Discussion #42
8. PM exits
```

## Error Handling

If PM cannot complete planning:

1. Add `status:needs-human` label to Discussion
2. Comment explaining what is unclear
3. Exit without creating Issues
