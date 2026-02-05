# GitHub Templates

This document specifies the Issue and PR templates to be implemented in a future phase. These templates help ensure consistent structure across work items.

## Status

**Implementation Status**: Planned for Phase 2 (CLI Foundation)

Templates will be created when `sy init` scaffolds a repository.

## Issue Templates

### Feature Request Template

**File**: `.github/ISSUE_TEMPLATE/feature_request.md`

```yaml
---
name: Feature Request
about: Propose a new feature (usually created by PM from Discussion)
title: '[FEATURE] '
labels: type:feature, status:ready
assignees: ''
---

## Summary
<!-- One paragraph description of the feature -->

## Background
<!-- Link to Discussion where this was planned -->
Discussion: #

## Acceptance Criteria
<!-- Checklist that defines "done" -->
- [ ]
- [ ]
- [ ]

## Technical Notes
<!-- Any implementation hints from planning -->

## Dependencies
<!-- Other Issues that must complete first -->

## Priority
<!-- high / medium / low -->

---
*Created from Discussion #*
```

### Bug Report Template

**File**: `.github/ISSUE_TEMPLATE/bug_report.md`

```yaml
---
name: Bug Report
about: Report a defect
title: '[BUG] '
labels: type:bug, status:ready
assignees: ''
---

## Description
<!-- What is broken? -->

## Steps to Reproduce
1.
2.
3.

## Expected Behavior
<!-- What should happen? -->

## Actual Behavior
<!-- What happens instead? -->

## Environment
<!-- Relevant environment details -->

## Possible Cause
<!-- If you have ideas about the cause -->

## Priority
<!-- high / medium / low -->
```

### Chore Template

**File**: `.github/ISSUE_TEMPLATE/chore.md`

```yaml
---
name: Chore
about: Maintenance, refactoring, or technical debt
title: '[CHORE] '
labels: type:chore, status:ready
assignees: ''
---

## Summary
<!-- What maintenance work is needed? -->

## Motivation
<!-- Why is this work important? -->

## Scope
<!-- What files/areas will be affected? -->

## Acceptance Criteria
- [ ]
- [ ]

## Priority
<!-- high / medium / low -->
```

### Config Template

**File**: `.github/ISSUE_TEMPLATE/config.yml`

```yaml
blank_issues_enabled: false
contact_links:
  - name: Planning Discussion
    url: https://github.com/OWNER/REPO/discussions/categories/planning
    about: Start with a Discussion for new features
  - name: Questions
    url: https://github.com/OWNER/REPO/discussions/categories/questions
    about: Ask questions about the project
```

## Pull Request Template

**File**: `.github/PULL_REQUEST_TEMPLATE.md`

```markdown
## Summary
<!-- Brief description of changes -->

## Issue
<!-- Link to the Issue this PR addresses -->
Closes #

## Changes
<!-- List of changes made -->
-
-
-

## Testing
<!-- How was this tested? -->
- [ ] Unit tests pass
- [ ] Manual testing performed
- [ ]

## Documentation
<!-- Documentation updates included in this PR -->
- [ ] Documentation updated for user-facing changes (or N/A)
- [ ] CHANGELOG.md updated (if applicable)

## Checklist
- [ ] Code follows project conventions
- [ ] Tests added/updated
- [ ] No unrelated changes included
- [ ] Acceptance criteria from Issue are met

## Screenshots
<!-- If applicable -->

---
*[AGENT] Created implementing Issue #*
```

## Directory Structure

After `sy init`:

```
.github/
├── ISSUE_TEMPLATE/
│   ├── feature_request.md
│   ├── bug_report.md
│   ├── chore.md
│   └── config.yml
└── PULL_REQUEST_TEMPLATE.md
```

## Template Variables

Templates may include variables that `sy init` replaces:

| Variable | Replacement |
|----------|-------------|
| `OWNER` | Repository owner |
| `REPO` | Repository name |
| `YEAR` | Current year |

## Agent Template Usage

### PM Creating Issues

PM populates templates programmatically:

```python
# Pseudocode for PM creating Issue
issue_body = f"""
## Summary
{spec_summary}

## Background
Discussion: #{discussion_number}

## Acceptance Criteria
{format_criteria(criteria)}

## Technical Notes
{technical_notes}

## Dependencies
{format_dependencies(deps)}

## Priority
{priority}

---
*Created by [PM] from Discussion #{discussion_number}*
"""
```

### Coder Creating PRs

Coder populates PR template:

```python
# Pseudocode for Coder creating PR
pr_body = f"""
## Summary
{change_summary}

## Issue
Closes #{issue_number}

## Changes
{format_changes(changes)}

## Testing
- [x] Unit tests pass
- [x] Manual testing performed

## Documentation
- [x] Documentation updated for user-facing changes
- [x] CHANGELOG.md updated

## Checklist
- [x] Code follows project conventions
- [x] Tests added/updated
- [x] No unrelated changes included
- [x] Acceptance criteria from Issue are met

---
*[CODER] Created implementing Issue #{issue_number}*
"""
```

## Validation

Templates should be validated during `sy init`:

1. All template files exist
2. YAML frontmatter is valid
3. Required sections are present
4. Labels referenced in templates exist

## Future Enhancements

Potential future template features:

1. **Dynamic templates**: Select template based on Issue labels
2. **Auto-population**: Pre-fill fields from linked Discussion
3. **Validation rules**: Enforce required fields before submission
4. **Template versioning**: Track template changes over time
