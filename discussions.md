# GitHub Discussions

This document defines how GitHub Discussions are used in Shipyard for planning, communication, and retrospectives.

## Discussion Categories

Configure these categories in your repository's Discussion settings:

### 1. Planning

**Purpose**: Feature planning and specification
**Emoji**: 📋
**Description**: Propose new features and plan implementations
**Format**: Open ended

**Who uses it**:
- Humans create Discussions to propose features
- PM monitors for new Discussions to plan
- All agents can comment during planning
- Coder references for implementation context

**Workflow**:
```
1. Human or PM creates Discussion describing a need
2. Community/agents discuss and refine
3. PM adds status:planning label when it begins work
4. PM creates spec and Issues
5. PM closes/answers the Discussion
6. Discussion serves as permanent record
```

### 2. Architecture

**Purpose**: Technical design decisions
**Emoji**: 🏗️
**Description**: Discuss system architecture and technical approaches
**Format**: Open ended

**Who uses it**:
- Coder proposes architectural changes
- Reviewer raises technical concerns
- Humans provide guidance

**Workflow**:
```
1. Anyone creates Discussion for architectural question
2. Technical discussion happens
3. Decision is reached and documented
4. Implementation proceeds based on decision
5. Discussion serves as ADR (Architecture Decision Record)
```

### 3. Retrospective

**Purpose**: Post-feature and periodic reviews
**Emoji**: 🔄
**Description**: Review completed work and process improvements
**Format**: Open ended

**Who uses it**:
- PM or humans create after significant features merge
- All agents can contribute lessons learned
- Humans review and adjust process

**Workflow**:
```
1. PM or human creates Retrospective after milestone
2. Links all related PRs, Issues, Discussions
3. Summarizes what went well and what didn't
4. Proposes process improvements
5. Human reviews and implements changes
```

### 4. Questions

**Purpose**: General questions and clarifications
**Emoji**: ❓
**Description**: Ask questions about the codebase or process
**Format**: Question / Answer

**Who uses it**:
- Any agent needing clarification
- Humans with questions about the system
- PM routing unclear requirements

### 5. Announcements

**Purpose**: Important updates (human managed)
**Emoji**: 📣
**Description**: Important updates and announcements
**Format**: Announcement

**Who uses it**:
- Humans only (agents should not post here)
- Major releases, breaking changes, process updates

## Discussion Templates

### Planning Discussion Template

```markdown
## Problem Statement
[What problem are we solving? Why does it matter?]

## Current Behavior
[How does the system behave today?]

## Desired Behavior
[How should the system behave?]

## Success Criteria
[How will we know this is complete?]

## Constraints
[Any limitations or requirements?]

## Open Questions
- [Question 1]
- [Question 2]

## Related
- [Link to related Issues/PRs/Discussions]
```

### Architecture Discussion Template

```markdown
## Context
[What is the architectural question or challenge?]

## Options Considered

### Option A: [Name]
**Description**: [How it works]
**Pros**: [Benefits]
**Cons**: [Drawbacks]

### Option B: [Name]
**Description**: [How it works]
**Pros**: [Benefits]
**Cons**: [Drawbacks]

## Recommendation
[Which option and why?]

## Decision
[To be filled after discussion]

## Consequences
[What changes as a result of this decision?]
```

### Retrospective Discussion Template

```markdown
## Summary
[What was accomplished?]

## Linked Items
- Discussion: #XX
- Issues: #XX, #XX
- PRs: #XX, #XX

## Timeline
[When did work start and complete?]

## What Went Well
- [Item 1]
- [Item 2]

## What Could Improve
- [Item 1]
- [Item 2]

## Action Items
- [ ] [Improvement 1]
- [ ] [Improvement 2]

## Metrics
- PRs created: X
- Review cycles: X
```

## PM Polling Query

PM uses the GitHub GraphQL API to find Discussions in the Planning category that have no status label (unprocessed):

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
        createdAt
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

Client-side filter: select Discussions where no label name starts with `status:`.

To get the `categoryId`, use:

```graphql
query {
  repository(owner: "OWNER", name: "REPO") {
    discussionCategories(first: 10) {
      nodes {
        id
        name
      }
    }
  }
}
```

## Agent Behavior in Discussions

### PM in Discussions

```
When PM completes planning:

[PM] Planning complete for this Discussion.

## Spec Summary
[Brief summary of what was decided]

## Created Issues
- #123: [Title] - [Brief description]
- #124: [Title] - [Brief description]

## Scope Decisions
- Included: [What's in scope]
- Deferred: [What's out of scope for now]

This Discussion is now closed. Created Issues are tagged status:ready.
```

### Coder in Discussions

```
When Coder starts work:

[CODER] Beginning implementation of Issue #123.

Referencing this Discussion for context.
Will update here if clarification needed.
```

## Setup Script

Enable and configure Discussions:

```bash
#!/bin/bash
# Enable Discussions (must be done via GitHub UI or API)

# Note: Discussion categories must be created via GitHub UI
# Settings > Features > Discussions > Set up discussions

echo "Manual steps required:"
echo "1. Go to repository Settings"
echo "2. Enable Discussions under Features"
echo "3. Create categories: Planning, Architecture, Retrospective, Questions, Announcements"
echo "4. Set appropriate formats and descriptions"
```

## Best Practices

### For Humans

1. **Start with a Discussion**: Don't jump straight to Issues for new features
2. **Be specific**: Clear problem statements lead to better solutions
3. **Stay engaged**: Respond to questions from agents
4. **Close the loop**: Confirm when intent is met

### For Agents

1. **Reference context**: Always link to source Discussions
2. **Be verbose**: Explain reasoning, not just actions
3. **Respect the thread**: Keep comments on topic
4. **Document decisions**: Make Discussions a useful historical record

### General

1. One topic per Discussion
2. Use appropriate category
3. Link related items bidirectionally
4. Keep Discussions open as reference even after completion (PM closes when planning is done, but the record persists)
