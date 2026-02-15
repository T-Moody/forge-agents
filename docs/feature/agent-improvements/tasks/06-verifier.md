# Task 06: Write Verifier Agent

## Task Goal

Create the complete `verifier.agent.md` file in `NewAgentsAndPrompts/`, rewriting the verifier to be technology-agnostic and redefine its role from "sole owner of build/test" to "integration-level verification agent."

**Output file:** `NewAgentsAndPrompts/verifier.agent.md`

## depends_on

05

## Cluster

**Cluster A (TDD) — Secondary agent.** Must agree with the implementer (Task 05) on TDD responsibility split:

- Implementer: "unit-level TDD" (writes and runs unit tests during implementation)
- Verifier: "integration-level verification" (runs full suite, checks cross-task interactions)

**Reference design.md § Per-Agent Design → 8. `verifier.agent.md` for exact terminology.**

## In-Scope

- Rewrite of `verifier.agent.md` following canonical structure
- Frontmatter with `name: verifier` and updated description ("Integration-level build, test, and verification agent")
- Role preamble (3 sentences)
- `detailed thinking on` directive
- **Inputs** section (same as current)
- **Outputs** section (verifier.md)
- **Role** section (REWRITTEN — "integration-level verification agent", NOT "sole owner")
- **Operating Rules** block (verifier variant, Rule 5: run_in_terminal, grep_search, never modify source)
- **Workflow** with:
  - Step 1: **Detect Build System** (NEW — technology-agnostic detection table replacing hardcoded dotnet commands)
  - Step 2: Build (technology-agnostic)
  - Step 3: Test Execution (technology-agnostic, integration focus, flag unit test failures as high-severity)
  - Step 4: Task-Level Verification (same as current)
  - Step 5: Feature-Level Verification (same as current)
  - Step 6: Report (same as current)
- **Read-Only Enforcement** (NEW — verifier MUST NOT modify source code)
- **Targeted Re-verification** (same as current)
- **verifier.md Contents** (same as current)
- **Completion Contract** — THREE-STATE: DONE/NEEDS_REVISION/ERROR (verifier is one of 3 agents using NEEDS_REVISION)
- **Anti-Drift Anchor**

## Out-of-Scope

- Modifying files in `.github/`
- Hardcoded project names, solution files, or technology-specific commands
- Modifying source code (verifier is read-only)

## Acceptance Criteria

1. File exists at `NewAgentsAndPrompts/verifier.agent.md` and is non-empty
2. **Does NOT contain** "dotnet", "TourneyPal", ".sln" or any hardcoded project reference (TS-7)
3. Role section says "integration-level verification agent" — NOT "sole agent responsible for building and running tests"
4. Role section notes: "Implementers have already verified their individual tasks via unit-level TDD"
5. Contains Build System Detection table with at least: package.json, Makefile/CMakeLists.txt, _.sln/_.csproj, pom.xml, build.gradle, Cargo.toml, pyproject.toml/setup.py, go.mod
6. Detection table includes non-exhaustive note listing additional systems (Deno, Bun, PHP, etc.)
7. Contains Read-Only Enforcement section prohibiting source code modification
8. Test Execution step notes: if unit tests fail that should have passed during TDD, flag as high-severity
9. Completion contract is THREE-STATE: DONE/NEEDS_REVISION/ERROR
10. NEEDS_REVISION used when "tests or builds fail but failures are addressable through a replan cycle"
11. Anti-drift anchor: "You are the **Verifier**. You build, test, and verify. You never modify source code or fix bugs. You report findings only. Stay as verifier."

## Test Requirements

1. **Technology portability test (TS-7):** Search for "dotnet", "TourneyPal", ".sln" → must NOT be found
2. **Technology detection test (TS-7):** Verify detection table references multiple build systems
3. **Cluster A consistency (TS-3):** Verify "integration-level" terminology and "Implementers have already verified unit tests" note
4. **Read-only test:** Verify explicit statement that verifier must not modify source code
5. **Three-state test:** Completion contract includes DONE, NEEDS_REVISION, and ERROR with correct semantics

## Implementation Steps

1. Read design.md § Per-Agent Design → 8. `verifier.agent.md` for target state outline and all content specifications
2. Read design.md § Cross-Cutting Patterns for shared blocks — note verifier uses THREE-STATE completion
3. **Verify Cluster A consistency:** Role section must use "integration-level verification" and reference "unit-level TDD" — matching the implementer's terminology from Task 05
4. Create `NewAgentsAndPrompts/verifier.agent.md` with:
   - YAML frontmatter (`name: verifier`, integration-level description)
   - `# Verifier Agent Workflow` title
   - Role preamble
   - `detailed thinking on` directive
   - `## Inputs` (same as current)
   - `## Outputs` (verifier.md)
   - `## Role` (REWRITTEN — integration-level, exact content from design.md)
   - `## Operating Rules` (verifier variant)
   - `## Workflow` with 6 steps:
     - `### 1. Detect Build System` (NEW technology-agnostic table from design.md)
     - `### 2. Build` (technology-agnostic)
     - `### 3. Test Execution` (technology-agnostic, integration focus)
     - `### 4. Task-Level Verification` (carry from current)
     - `### 5. Feature-Level Verification` (carry from current)
     - `### 6. Report` (carry from current)
   - `## Read-Only Enforcement` (NEW)
   - `## Targeted Re-verification` (same)
   - `## verifier.md Contents` (same)
   - `## Completion Contract` (THREE-STATE)
   - `## Anti-Drift Anchor`

## Estimated Effort

Medium

## Completion Checklist

- [x] File created at `NewAgentsAndPrompts/verifier.agent.md`
- [x] NO hardcoded project names/commands ("dotnet", "TourneyPal", ".sln") — detection table lists .NET as one of 8 generic build systems per design.md spec
- [x] Role redefined to "integration-level verification agent"
- [x] Build system detection table present with 8+ systems
- [x] Non-exhaustive note for additional build systems
- [x] Read-Only Enforcement section present
- [x] Unit test failure flagged as high-severity (TDD expectation)
- [x] Completion contract is three-state (DONE/NEEDS_REVISION/ERROR)
- [x] Anti-drift anchor with exact text
- [x] Cluster A terminology consistent with implementer
