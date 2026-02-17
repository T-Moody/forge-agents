# Task 10: Update r-knowledge.agent.md Inputs and Workflow

**Task Goal:** Reduce r-knowledge's direct inputs from 6 artifacts to 2 (`memory.md` + `initial-request.md`) and update Workflow Step 2 to use Artifact Index navigation instead of full-file reads.

**depends_on:** 04

**agent:** implementer

---

## In-Scope

- Update `NewAgentsAndPrompts/r-knowledge.agent.md` Inputs section
- Update `NewAgentsAndPrompts/r-knowledge.agent.md` Workflow Step 2

## Out-of-Scope

- Modifying r-knowledge YAML frontmatter (already done in Task 04)
- Modifying orchestrator dispatch table (handled by Task 11)
- Modifying any other R sub-agent file
- Removing r-knowledge's access to `decisions.md`, `.github/instructions/`, git diff, or codebase

---

## Acceptance Criteria

1. Inputs section lists `memory.md` and `initial-request.md` as primary inputs
2. Inputs section does NOT list `feature.md`, `design.md`, `plan.md`, or `verifier.md` as direct inputs
3. Inputs section includes a note directing Artifact Index navigation for other artifacts
4. Workflow Step 2 instructs Artifact Index navigation, not full-file reads
5. `decisions.md`, `.github/instructions/`, git diff, and codebase access are preserved

---

## Estimated Effort

Low

---

## Test Requirements

- `grep "feature.md" NewAgentsAndPrompts/r-knowledge.agent.md` in the Inputs section → 0 matches (may appear elsewhere in context text)
- `grep "Artifact Index" NewAgentsAndPrompts/r-knowledge.agent.md` → matches in both Inputs section and Workflow Step 2
- Workflow Step 2 does NOT instruct reading feature.md/design.md/plan.md/verifier.md in full

---

## Implementation Steps

### Step 1: Update Inputs Section (~L18–28)

**Find the current Inputs section (approximately):**

```markdown
## Inputs

- docs/feature/<feature-slug>/memory.md (read first — operational memory)
- docs/feature/<feature-slug>/initial-request.md
- docs/feature/<feature-slug>/feature.md
- docs/feature/<feature-slug>/design.md
- docs/feature/<feature-slug>/plan.md
- docs/feature/<feature-slug>/verifier.md
- docs/feature/<feature-slug>/decisions.md (if present)
- .github/instructions/ (if present)
- Git diff
- Entire codebase
```

**Replace with:**

```markdown
## Inputs

- docs/feature/<feature-slug>/memory.md (read first — operational memory; use Artifact Index for targeted navigation to other artifacts)
- docs/feature/<feature-slug>/initial-request.md
- docs/feature/<feature-slug>/decisions.md (if present)
- .github/instructions/ (if present)
- Git diff
- Entire codebase

> **Note:** `feature.md`, `design.md`, `plan.md`, and `verifier.md` are accessed via Artifact Index navigation — not read in full. Use the Artifact Index in `memory.md` to identify and read only the relevant sections.
```

### Step 2: Update Workflow Step 2 (~L80–82)

**Find the current Step 2 (approximately):**

```markdown
### 2. Read Pipeline Artifacts

Read `initial-request.md`, `feature.md`, `design.md`, `plan.md`, and `verifier.md` for full pipeline history and context. Use the Artifact Index from memory to navigate directly to relevant sections.
```

**Replace with:**

```markdown
### 2. Read Pipeline Artifacts

Read `initial-request.md` for original request context. Use the Artifact Index in `memory.md` to identify and read only the relevant sections of `feature.md`, `design.md`, `plan.md`, and `verifier.md` — do not read these files in full. If the Artifact Index lacks sufficient detail for a specific artifact, fall back to targeted reads using `grep_search` or `semantic_search` rather than reading the entire file.
```

### Step 3: Verify

- Confirm Inputs section no longer lists feature.md/design.md/plan.md/verifier.md as direct inputs
- Confirm Workflow Step 2 references Artifact Index navigation
- Confirm decisions.md, .github/instructions/, git diff, and codebase access are preserved

---

## Completion Checklist

- [x] Inputs section updated — 4 artifacts removed, Artifact Index note added
- [x] Workflow Step 2 updated — full reads replaced with Artifact Index navigation + fallback
- [x] Access to decisions.md, .github/instructions/, git diff, and codebase preserved
- [x] No other files modified
