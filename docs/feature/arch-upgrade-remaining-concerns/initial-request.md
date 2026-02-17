# Feature Request: Address Architecture Upgrade Remaining Concerns

## Target Directory

`X:\programProjects\OrchestratorAgents\NewAgentsAndPrompts`

## 1. Remove Memory Line Limits

The 200-line emergency prune cap on `memory.md` is actively harmful. The memory system exists
to prevent agents from reading large artifacts — if memory itself gets pruned, agents lose
context and fall back to direct artifact reads, defeating the purpose entirely.

Changes needed:

- **Remove the 200-line emergency prune rule** from `orchestrator.agent.md` (Global Rule 6
  and Memory Lifecycle Actions table).
- **Remove the emergency prune row** from the Memory Lifecycle Actions table.
- **Replace with structural growth controls**: Instead of a hard line cap, use smarter
  pruning — keep Artifact Index and Lessons Learned always, prune Recent Decisions and
  Recent Updates to only the current and previous phase (already done at checkpoints).
  No emergency prune trigger.
- **Update all agents** that reference the 200-line limit or emergency pruning (if any
  beyond orchestrator).

Rationale: memory.md is structured markdown with clear sections. Even at 400+ lines it's
cheaper for an agent to read than re-reading design.md + feature.md + plan.md + all task
files. A line cap creates a failure mode where the system degrades to exactly the behavior
it was designed to prevent.

## 2. Reduce Orchestrator Complexity

The orchestrator is 489 lines, exceeding its own 450-line target. The risk is that the AI
running as orchestrator loses track of rules mid-pipeline, especially around Pattern C
(replan loop) and memory invalidation.

Changes needed:

- **Extract the three dispatch patterns** (Pattern A, Pattern B, Pattern C) into a
  separate reference document (`docs/patterns.md` or similar) that the orchestrator
  references but doesn't inline. The orchestrator keeps a one-line summary per pattern
  and a pointer.
- **Simplify the Memory Lifecycle Actions table** — especially now that emergency prune
  is removed (see item 1).
- **Target: ≤400 lines** for the orchestrator after extraction.

## 3. Add Memory Write Safety Markers

The rule "parallel sub-agents read memory but do NOT write to it" is enforced only by
prompt instruction. If a sub-agent accidentally writes during parallel execution, memory
corruption could occur.

Changes needed:

- **Add a `memory_access: read-only` field** to the YAML frontmatter of every parallel
  sub-agent (all ct-_, v-tests, v-tasks, v-feature, r-_, researcher focused instances).
- **Add a `memory_access: read-write` field** to every sequential/aggregator agent
  (ct-aggregator, v-aggregator, r-aggregator, researcher synthesis, spec, designer,
  planner, implementer, documentation-writer).
- **Update orchestrator** to validate the `memory_access` field before dispatch — if
  dispatching in parallel, all agents in the wave MUST have `memory_access: read-only`.
  Log a warning if violated.
- This doesn't add a technical lock (VS Code Copilot has no file-locking mechanism),
  but it makes the contract explicit and machine-checkable in the dispatch logic.

## 4. Scope r-knowledge Inputs

r-knowledge has the widest input set of any sub-agent (initial-request.md, feature.md,
design.md, plan.md, verifier.md, AND memory.md). This risks context window pressure.

Changes needed:

- **r-knowledge should use memory.md as its PRIMARY input** and only read specific
  sections of other artifacts via Artifact Index navigation — not receive all 5 files
  as full inputs.
- **Update the orchestrator's R cluster dispatch** (Step 7.2) to pass r-knowledge only
  `memory.md` and `initial-request.md` as direct inputs, with instructions to use the
  Artifact Index for targeted reads of other files as needed.
- **Update r-knowledge.agent.md** workflow Step 2 to use Artifact Index navigation
  instead of listing 5 files to read in full.

## Priority Order

1. Remove memory limits (highest impact — eliminates a self-defeating failure mode)
2. Memory write safety markers (low effort, high clarity)
3. Scope r-knowledge inputs (medium effort, medium impact)
4. Reduce orchestrator complexity (highest effort, important for long-term reliability)

## Additional Requirement

Update README.md when finished.
