# Shipyard Roadmap

This document outlines the implementation phases for Shipyard, from initial proof of concept through production readiness.

## Phase 0: Specification (Current)

**Goal**: Complete planning and documentation before writing code.

**Deliverables**:
1. ✅ Project structure and README
2. ✅ Core specification (SPEC.md)
3. ✅ Architecture documentation
4. ✅ Skill specifications for all agents
5. ✅ GitHub configuration (labels, discussions)
6. ✅ CLI command specification
7. ✅ This roadmap

**Exit Criteria**: All documentation reviewed and approved for implementation.

## Phase 1: Manual POC

**Goal**: Prove the orchestration pattern works with manual agent invocation.

**Approach**: Run each agent skill in a separate terminal, manually triggering them to process work items. No automation, just validation that the workflow functions.

**Deliverables**:
1. Basic skill files for each agent (PM, Coder, Reviewer)
2. Auto-merge logic (branch protection rules + merge script)
3. GitHub labels and discussion categories configured
4. A simple demo task that flows through all agents and merges
5. Documentation of manual invocation process

**Success Criteria**:
1. A Discussion can be planned by PM, resulting in an Issue
2. Coder can pick up the Issue and create a PR (including doc updates)
3. Reviewer can review and approve the PR
4. Approved PR auto-merges via branch protection rules

## Phase 2: CLI Foundation

**Goal**: Create the `shipyard` (sy) CLI for easier agent management.

**Deliverables**:
1. `sy init` command to scaffold a repo (14 labels, templates, config)
2. `sy status` to show pending work across all queues
3. `sy run <agent>` to invoke a specific agent skill
4. `sy queue` to view work items by agent type
5. `sy merge` to check and execute merges for approved PRs
6. Basic configuration file support (`.shipyard/config.yaml`)

**Success Criteria**:
1. Can initialize a new repo with one command
2. Can see the state of all work items
3. Can trigger agents without remembering skill paths

## Phase 3: Skill Refinement

**Goal**: Harden skills based on real usage patterns.

**Deliverables**:
1. Assignee-based optimistic locking for claim mechanism
2. Stale claim detection and automatic recovery (30 min timeout)
3. Merge conflict detection and Coder-driven rebase flow
4. Subagent patterns for context isolation
5. Per-session token budget enforcement
6. Agent identity prefixing in all outputs
7. Review cycle limit (3 cycles → `status:needs-human`)

**Success Criteria**:
1. Agents handle edge cases gracefully
2. Work items don't get stuck permanently (stale detection works)
3. Merge conflicts are resolved without human intervention
4. Context stays clean (subagents working)
5. Costs stay predictable (per-session caps enforced)

## Phase 4: Dogfooding

**Goal**: Use Shipyard to build Shipyard.

**Approach**: The framework builds its own features using its own orchestration.

**Deliverables**:
1. All new features planned via PM agent
2. All new code written via Coder agent (including documentation updates in feature PRs)
3. All PRs reviewed via Reviewer agent
4. Merges handled by branch protection rules

**Success Criteria**:
1. At least 5 features delivered entirely through Shipyard workflow
2. Friction points identified and documented
3. Skills updated based on real usage

## Phase 5: Automation

**Goal**: Enable hands-off operation with scheduled polling.

**Deliverables**:
1. GitHub Actions workflow for scheduled agent runs
2. Heartbeat/health monitoring
3. Stale work detection and alerting
4. Cost tracking and budget enforcement

**Success Criteria**:
1. Agents run on schedule without manual intervention
2. Stuck work items generate alerts
3. Budget limits prevent overspend

## Phase 6: Distribution

**Goal**: Make Shipyard easy for others to adopt.

**Deliverables**:
1. `npx shipyard init` for zero-install setup
2. Published skill packages
3. Comprehensive getting started guide
4. Example repositories demonstrating patterns
5. Video walkthrough of complete workflow

**Success Criteria**:
1. New user can set up Shipyard in under 10 minutes
2. Documentation answers common questions
3. Community can contribute improvements

## Future Considerations (Post v1.0)

These are interesting directions that inform design but are not committed:

**Distributed Execution**: Push work to other machines, enabling parallel agent execution across a fleet.

**Custom Agents**: User-defined agent personas beyond the core three, with custom polling and action logic (e.g., dedicated merge oversight, documentation management, or other specialized workflows).

**Cross-Repository**: Agents that coordinate work across multiple repositories (e.g., shared library updates).

**Learning and Memory**: Agents that improve over time based on feedback patterns.

**Integration Ecosystem**: Plugins for Slack notifications, Jira sync, custom webhooks.

## Measuring Success

Throughout all phases, we track:

1. **Cycle Time**: How long from Discussion to merged PR?
2. **Token Efficiency**: Cost per feature delivered
3. **Human Intervention Rate**: How often do humans need to unstick things?
4. **Quality Metrics**: PR rejection rate, post-merge bugs
5. **Developer Experience**: Is this actually pleasant to use?
