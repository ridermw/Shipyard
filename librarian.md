# Librarian Skill

The Librarian agent is responsible for keeping documentation in sync with code changes. It reviews merged PRs and updates relevant docs, README, changelog, and wiki.

## Trigger Criteria

The Librarian skill polls for:

1. **PRs** that were recently merged (within last 24 hours)
2. **PRs** without the `docs:updated` label
3. Prioritizes by age (oldest unprocessed first)

## Responsibilities

### 1. Documentation Review

After a PR is merged:

```
1. Read the merged PR and its linked Issue/Discussion
2. Identify what documentation might be affected:
   - README changes (new features, changed behavior)
   - API documentation (new endpoints, changed parameters)
   - User guides (new workflows, changed processes)
   - Architecture docs (structural changes)
   - Changelog (all user facing changes)
3. Determine if updates are needed
```

### 2. Documentation Updates

When updates are needed:

```
1. Create a worktree for documentation changes
2. Update relevant documentation files
3. Update CHANGELOG.md with the change
4. Create a PR for doc changes
5. Mark original PR as docs:updated
```

### 3. Changelog Maintenance

Librarian maintains a structured changelog:

```markdown
## [Unreleased]

### Added
- Clear error messages with error codes (#150)

### Changed
- Updated authentication flow (#145)

### Fixed
- Resolved race condition in search (#142)

### Removed
- Deprecated legacy API endpoints (#140)
```

### 4. No Documentation Needed

When no updates are needed:

```
1. Verify the change is truly internal (no user impact)
2. Add docs:updated label to mark as processed
3. Leave brief comment explaining no docs needed
```

## Output Artifacts

### Documentation PR

```markdown
## Summary
Documentation updates for PR #150 (error message improvements)

## Changes
- Updated README with error code reference
- Added troubleshooting section to user guide
- Updated CHANGELOG with new feature

## Linked Items
- Documents PR #150
- Documents Issue #100
- Documents Discussion #42

---
*Created by [LIBRARIAN] documenting PR #150*
```

### Changelog Entry

```markdown
### Added
- User friendly error messages with searchable error codes ([#150](link))
  - All API errors now include error codes (ERR_XXX_NNN format)
  - Error codes link to troubleshooting guide
  - See [Error Reference](docs/errors.md) for full list
```

### No Docs Needed Comment

```markdown
[LIBRARIAN] Documentation review complete for PR #150.

No documentation updates required. This PR contains:
- Internal refactoring only
- No user facing changes
- No API changes

Marking as `docs:updated`.
```

## Labels Used

### Reads
| Label | Meaning |
|-------|---------|
| `status:merged` | PR was merged, may need docs |
| (absence of) `docs:updated` | Not yet processed by Librarian |

### Writes
| Label | Meaning |
|-------|---------|
| `docs:updated` | Librarian has processed this PR |
| `docs:pending` | Doc PR created, awaiting merge |
| `agent:librarian` | Librarian is working on this |

## Files Typically Updated

| File | When Updated |
|------|--------------|
| `README.md` | New features, major changes, getting started |
| `CHANGELOG.md` | Every user facing change |
| `docs/api.md` | API changes |
| `docs/guides/*.md` | Workflow changes |
| `docs/architecture.md` | Structural changes |
| `.github/wiki/*` | If wiki is used |

## Constraints

1. **Accuracy Over Speed**: Ensure documentation is correct before creating PR
2. **Minimal Changes**: Only update what's necessary for the merged PR
3. **Link Everything**: Documentation should link back to the source PR/Issue
4. **Maintain Voice**: Match the existing documentation style and tone

## Subagent Usage

Librarian uses subagents for:

1. **Change Analysis**: "What user facing changes are in this PR?"
2. **Doc Discovery**: "What documentation files might be affected by this change?"
3. **Style Matching**: "What is the documentation style used in this repo?"
4. **Completeness Check**: "Are there any docs that reference the changed functionality?"

## Example Workflow

### Documentation Needed

```
1. Librarian polls GitHub, finds merged PR #150 without docs:updated
2. Librarian reads PR #150: Error message improvements
3. Librarian reads Issue #100 and Discussion #42 for context
4. Librarian uses subagent to find related documentation
5. Librarian determines:
   - README needs error code section
   - New troubleshooting doc needed
   - CHANGELOG entry needed
6. Librarian creates worktree
7. Librarian updates documentation files
8. Librarian creates PR #175 for doc changes
9. Librarian adds docs:pending to PR #150
10. Librarian exits
```

### No Documentation Needed

```
1. Librarian polls GitHub, finds merged PR #160 without docs:updated
2. Librarian reads PR #160: Internal refactoring
3. Librarian uses subagent to verify no user impact
4. Librarian determines: No docs needed
5. Librarian adds docs:updated label to PR #160
6. Librarian leaves comment explaining decision
7. Librarian exits
```

## Decision Matrix

| Change Type | Documentation Action |
|-------------|---------------------|
| New feature | README + Guide + Changelog |
| Bug fix (user visible) | Changelog only |
| Bug fix (internal) | No docs needed |
| API change | API docs + Changelog |
| Refactoring | No docs needed |
| Performance improvement | Changelog if significant |
| Security fix | Changelog + Security notes if applicable |
| Dependency update | Changelog if user facing |

## Documentation PR Workflow

Documentation PRs follow a simplified review flow:

```
1. Librarian creates doc PR
2. Doc PR goes directly to Owner (skip Reviewer for docs)
3. Owner verifies accuracy
4. Owner merges
5. Original PR marked docs:updated
```

This allows faster documentation without bottlenecking on Reviewer.

## Error Handling

If Librarian cannot determine what docs need updating:

1. Add `docs:needs-review` label to merged PR
2. Comment asking for human guidance
3. Exit without creating doc PR

If documentation is significantly out of date:

1. Create Issue for documentation overhaul
2. Update what can be updated in current PR
3. Link to the new Issue for remaining work
