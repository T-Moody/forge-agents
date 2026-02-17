# Initial Request: Forge Architecture Upgrade

## Feature Request

Design and implement a major architecture upgrade for the Forge multi-agent orchestration framework that improves performance, parallelism, and long-term learning while maintaining deterministic grounding in artifacts.

## Current System

Forge currently includes:

- A phase-based orchestration pipeline
- Parallel research agents
- Spec → Design → Critical Thinking → Planning → Implementation → Verification → Review workflow
- Explicit artifacts (research docs, feature.md, design.md)
- Parallel implementation waves with dependency graphs

## Current Problems

1. Agents repeatedly read large artifacts unnecessarily, slowing execution.
2. There is no persistent operational memory shared across agents.
3. Critical Thinking is a single sequential bottleneck.
4. Verification is a single agent performing multiple independent tasks.
5. Review and knowledge improvement tasks run sequentially instead of in parallel.
6. Agents do not share a consistent model for context retrieval and learning.
7. The reviewer is a single agent when it could be split up.
8. We have nothing updating instructions to keep them up to date.

## Memory System Requirements

Design a lightweight operational memory system that:

- Does NOT duplicate or summarize full artifact content.
- Keeps artifacts as the single source of truth.
- Stores only:
  - Artifact navigation metadata (index and section mapping)
  - Recent decisions and context
  - Lessons learned and implementation issues
  - Recent artifact updates
- Ensures agents read memory FIRST before accessing artifacts.
- Minimizes unnecessary artifact reads.
- Remains small, structured, and phase-aware.
- Defines clear read/write responsibilities for each agent type.
- All agents must be updated to use the new memory workflow consistently.

## Workflow Parallelization Requirements

1. **Split Critical Thinker into 4 parallel sub-agents** with an aggregation strategy that does not create a new bottleneck.
2. **Replace single Verifier with a parallel Verification Cluster (4 subagents)** with aggregation.
3. **After implementation waves complete**, run in parallel: the Verification Cluster and aggregation.
4. **Split Reviewer into 4 parallel sub-agents** forming a Review Cluster that includes Knowledge Evolution:
   - Suggest improvements/add/remove to Copilot instructions
   - Suggest improvements/add/remove to skills
   - Capture reusable workflow patterns
   - Write safe, reviewable updates
5. **Research phase** should use 4 parallel agents (currently only 3).

## Deliverables

1. Memory architecture and file structure
2. How all agents integrate with memory
3. Updated orchestration flow diagram
4. Artifact indexing strategy
5. Parallel execution model
6. Knowledge evolution safeguards
7. Implementation-level guidance

## Agents to Update

All agents in: `NewAgentsAndPrompts/`

- critical-thinker.agent.md
- designer.agent.md
- documentation-writer.agent.md
- feature-workflow.prompt.md
- implementer.agent.md
- orchestrator.agent.md
- planner.agent.md
- researcher.agent.md
- reviewer.agent.md
- spec.agent.md
- verifier.agent.md

## Scope

Full control to refactor agents as needed. Any additional bottlenecks or parallelization opportunities should be identified and addressed.
