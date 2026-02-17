# Task 16: Deprecate Superseded Agents

## Agent

implementer

## Depends On

14

## Description

Add deprecation notices to the 3 agent files superseded by the cluster architecture: `critical-thinker.agent.md`, `verifier.agent.md`, and `reviewer.agent.md`. Each file retains its full content but receives a prominent deprecation header pointing users to the replacement cluster agents. Files are NOT deleted — they are retained for reference and rollback during migration validation.

## Input Files

- `docs/feature/forge-architecture-upgrade/design.md` — §11 (Migration Strategy, Phase 4)
- `NewAgentsAndPrompts/critical-thinker.agent.md` — current file
- `NewAgentsAndPrompts/verifier.agent.md` — current file
- `NewAgentsAndPrompts/reviewer.agent.md` — current file

## Output File

- `NewAgentsAndPrompts/critical-thinker.agent.md` (modified — deprecation notice prepended)
- `NewAgentsAndPrompts/verifier.agent.md` (modified — deprecation notice prepended)
- `NewAgentsAndPrompts/reviewer.agent.md` (modified — deprecation notice prepended)

## Acceptance Criteria

1. Each file has a deprecation notice prepended BEFORE the YAML frontmatter
2. Deprecation notice follows this format:
   ```
   > **⚠️ DEPRECATED:** This agent has been superseded by the [Cluster Name] cluster.
   > Replacement agents: [list of replacement agent files].
   > This file is retained for reference during migration validation.
   > See `docs/feature/forge-architecture-upgrade/design.md` for details.
   ```
3. **critical-thinker.agent.md** → points to CT cluster: `ct-security.agent.md`, `ct-scalability.agent.md`, `ct-maintainability.agent.md`, `ct-strategy.agent.md`, `ct-aggregator.agent.md`
4. **verifier.agent.md** → points to V cluster: `v-build.agent.md`, `v-tests.agent.md`, `v-tasks.agent.md`, `v-feature.agent.md`, `v-aggregator.agent.md`
5. **reviewer.agent.md** → points to R cluster: `r-quality.agent.md`, `r-security.agent.md`, `r-testing.agent.md`, `r-knowledge.agent.md`, `r-aggregator.agent.md`
6. Original file content is NOT modified — only the deprecation header is added
7. The orchestrator (updated in task 14) no longer references these files for dispatch

## Implementation Guidance

**Approach:** Prepend a blockquote deprecation notice to each file. Do not modify any existing content below the notice.

**For each file:**

1. Read the current file content
2. Prepend the deprecation notice (blockquote format)
3. Add a blank line separator
4. Write the file back with the notice + original content

**Note:** These are simple additive modifications to markdown files — TDD does not apply.

## Status

- [x] Deprecation notice prepended to `critical-thinker.agent.md` → CT cluster
- [x] Deprecation notice prepended to `verifier.agent.md` → V cluster
- [x] Deprecation notice prepended to `reviewer.agent.md` → R cluster
- [x] Original file content preserved (additive only)
- TDD skipped: configuration/documentation-only task (markdown deprecation headers)

**Status: COMPLETE**
