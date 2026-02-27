# Verification: Per-Task Acceptance Criteria

## Status

PASS

## Summary

13 tasks verified, 0 partially-verified, 0 failed. All 14 output files exist and meet their acceptance criteria. Deep verification performed on Tasks 01, 02, 11, and 13; skim verification on Tasks 03–10 and 12.

## Per-Task Verification

### Task 01: Schemas Reference Document

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/schemas.md` (1276 lines)
  - [x] AC2 — All 10 schemas present in pipeline-step order: completion-contract → research-output → spec-output → design-output → plan-output → task-schema → implementation-report → verification-report → review-findings → knowledge-output
  - [x] AC3 — Each schema includes required fields, types, constraints/allowed values, and one example payload
  - [x] AC4 — Producer/consumer dependency table present with 10 rows (L42–55) matching design.md
  - [x] AC5 — Schema evolution strategy section present (L1225–1276) with additive/breaking change policies, version policy
  - [x] AC6 — `completion-contract` schema (L59) includes: status, summary, severity, findings_count, risk_level, output_paths, evidence_summary
  - [x] AC7 — SQLite schemas: `anvil_checks` (L967 with v4 columns: run_id, verdict, severity, round) and `pipeline_telemetry` (L1069)
  - [x] AC8 — Both SQLite schemas include WAL + busy_timeout pragmas (L952–953, L1128) and required indexes (L1107–1109)
  - [x] AC9 — `task_id` convention documented (L1170–1195): per-task = planner-assigned IDs, feature-level = `{feature_slug}-design-review` / `{feature_slug}-code-review`
  - [x] AC10 — `schema_version: "1.0"` requirement documented (L11) as common header requirement
- **Tests:** N/A — documentation-only task, no behavioral code
- **Issues:** None

### Task 02: Reference Documents (Dispatch Patterns + Severity Taxonomy)

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — `dispatch-patterns.md` exists at `NewAgents/.github/agents/dispatch-patterns.md` (178 lines)
  - [x] AC2 — `severity-taxonomy.md` exists at `NewAgents/.github/agents/severity-taxonomy.md` (167 lines)
  - [x] AC3 — Pattern A (Fully Parallel, L8) and Pattern B (Sequential with Replan Loop, L63) formally defined with gate conditions, retry budgets, concurrency limits
  - [x] AC4 — Concurrency cap of 4 documented (L103+) with sub-wave partitioning rules
  - [x] AC5 — All 4 severity levels defined: Blocker (L12), Critical (L30), Major (L48), Minor (L66) with precise criteria and pipeline impact
  - [x] AC6 — Security blocker policy explicitly stated (L84+): any Blocker → pipeline halt, never downgraded/deferred/overridden
  - [x] AC7 — Both documents follow consistent Markdown structure with clear section headers
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 03: Researcher Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/researcher.agent.md` (247 lines)
  - [x] AC2 — Follows agent definition template: header, role, input/output schema, workflow, contract, rules, self-verification, tools, anti-drift
  - [x] AC3 — References `research-output` schema from schemas.md (L39, L45, L148)
  - [x] AC4 — Lists exactly 5 allowed tools: `read_file`, `list_dir`, `grep_search`, `semantic_search`, `file_search` (L221–227)
  - [x] AC5 — Completion contract: DONE | ERROR only (L159–166), explicitly states no NEEDS_REVISION
  - [x] AC6 — Workflow describes: receive focus → investigate → structure → produce typed YAML + companion Markdown (L127–155)
  - [x] AC7 — Outputs: `research/<focus>.yaml` and `research/<focus>.md` (L45–46)
  - [x] AC8 — Self-verification section present (L187+)
  - [x] AC9 — Anti-drift anchor present (L244)
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 04: Spec Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/spec.agent.md` (339 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — References `spec-output` schema from schemas.md (L59–61)
  - [x] AC4 — Pushback system documented (L127–195): concern identification, structured multiple-choice via `ask_questions`, interactive vs autonomous behavior
  - [x] AC5 — Tool list includes `ask_questions` (L158)
  - [x] AC6 — Completion contract: DONE | ERROR (L97)
  - [x] AC7 — Workflow includes read research → evaluate pushback → produce requirements → structure output
  - [x] AC8 — Output schema references match schemas.md
  - [x] AC9 — Self-verification (L291) and anti-drift anchor (L335) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 05: Designer Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/designer.agent.md` (277 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — References `design-output` schema from schemas.md (L52)
  - [x] AC4 — Justification scoring documented (L111) with decision-record format
  - [x] AC5 — Confidence level definitions included: High/Medium/Low (L142+)
  - [x] AC6 — Completion contract: DONE | ERROR (L81)
  - [x] AC7 — Workflow includes read spec + research → evaluate → make decisions → score justifications → produce output
  - [x] AC8 — Self-verification (L212) and anti-drift anchor (L273) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 06: Planner Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/planner.agent.md` (377 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — References `plan-output` (Schema 5) and `task-schema` (Schema 6) from schemas.md (L42, L50, L52)
  - [x] AC4 — Risk classification 🟢/🟡/🔴 fully documented with criteria (L17+)
  - [x] AC5 — Task sizing rules documented (Standard vs Large)
  - [x] AC6 — `relevant_context` pointer mechanism documented (L3)
  - [x] AC7 — `overall_risk_summary` field documented (L63)
  - [x] AC8 — Completion contract: DONE | NEEDS_REVISION | ERROR (3-state)
  - [x] AC9 — Replan mode workflow documented
  - [x] AC10 — Self-verification (L341) and anti-drift anchor (L374) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 07: Implementer Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/implementer.agent.md` (487 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — References `implementation-report` schema from schemas.md (L44, L51)
  - [x] AC4 — Baseline capture workflow documented: IDE diagnostics, build, test state (L66+)
  - [x] AC5 — Self-fix loop documented: max 2 attempts (L14)
  - [x] AC6 — Git staging (`git add -A`) documented
  - [x] AC7 — Revert mode documented with parameters: mode, files_to_revert, baseline_tag (L34–37)
  - [x] AC8 — TDD workflow described (L12)
  - [x] AC9 — `relevant_context` pointer consumption documented
  - [x] AC10 — Documentation mode documented
  - [x] AC11 — Completion contract: DONE | ERROR (no NEEDS_REVISION)
  - [x] AC12 — Self-verification (L421) and anti-drift anchor (L484) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 08: Verifier Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/verifier.agent.md` (523 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — References `verification-report` schema from schemas.md (L44, L50)
  - [x] AC4 — All 4 tiers documented with check details and SQL INSERT format
  - [x] AC5 — Tier 4 conditional execution documented (Large tasks only)
  - [x] AC6 — SQL INSERT pattern for `anvil_checks` documented (fields confirmed: run_id, task_id, phase, check_name, tool, command, exit_code, output_snippet, passed, round)
  - [x] AC7 — Baseline cross-check via `git show pipeline-baseline-{run_id}:<path>` documented (L86)
  - [x] AC8 — Regression detection logic documented: baseline vs after comparison (L80)
  - [x] AC9 — Completion contract: DONE | NEEDS_REVISION | ERROR (L89)
  - [x] AC10 — Self-verification (L386) and anti-drift anchor (L520) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 09: Adversarial Reviewer Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/adversarial-reviewer.agent.md` (374 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — References `review-findings` schema from schemas.md (L46)
  - [x] AC4 — Dispatch parameters documented: review_scope, model, review_focus (security/architecture/correctness), run_id, round (L34+)
  - [x] AC5 — Output format: Markdown findings + YAML verdict summary + SQL INSERT into anvil_checks (L6, L14)
  - [x] AC6 — Security blocker policy present
  - [x] AC7 — Completion contract: DONE | ERROR (confirmed via grep)
  - [x] AC8 — Self-verification (L309) and anti-drift anchor (L371) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 10: Knowledge Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/knowledge-agent.agent.md` (502 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — References `knowledge-output` schema from schemas.md (L56, L62)
  - [x] AC4 — Evidence bundle assembly documented with all 6 components (L87+)
  - [x] AC5 — Non-blocking behavior explicitly stated (L9, L20): ERROR does not halt pipeline
  - [x] AC6 — Decision log append-only format documented
  - [x] AC7 — Governed updates documented
  - [x] AC8 — Safety constraint filter documented
  - [x] AC9 — `store_memory` usage documented for cross-session persistence (L18, L81)
  - [x] AC10 — Completion contract: DONE | ERROR (L486)
  - [x] AC11 — Self-verification (L427) and anti-drift anchor (L497) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 11: Orchestrator Agent Definition

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/agents/orchestrator.agent.md` (760 lines)
  - [x] AC2 — Follows agent definition template with all required sections
  - [x] AC3 — All 5 core responsibilities documented (L16–20): dispatch routing, approval gate management, error categorization + retry, evidence gate verification, pipeline initialization
  - [x] AC4 — Tool access: 7 allowed (agent, agent/runSubagent, memory, read_file, list_dir, run_in_terminal, get_terminal_output), 6 restricted (create_file, replace_string_in_file, grep_search, semantic_search, file_search, get_errors) — L35–53
  - [x] AC5 — `run_in_terminal` constraint (DR-1) explicitly documented (L55–70): allowed operations (SQLite reads, DDL, git reads, git staging/commit) and forbidden operations (builds, tests, code execution, file modification)
  - [x] AC6 — Complete pipeline step sequence Steps 0–9 with routing logic (L117–530)
  - [x] AC7 — Orchestrator decision table present (L535–568) with ALL conditions: setup, research, spec, design, design review, plan, implementation, verification, code review, knowledge, auto-commit, agent errors
  - [x] AC8 — Step 0 initialization documented (L127–183): SQLite DDL, WAL, git hygiene, pushback, run_id generation
  - [x] AC9 — All 7 evidence gate SQL queries present (L573–645): baseline exists, verification sufficient, design review approval, design review blocker, code review approval, code review blocker, review completion
  - [x] AC10 — Approval gate format: structured multiple-choice with example prompts (L203–227, L338–359)
  - [x] AC11 — Retry budgets documented (L650–662): 1 agent retry, 3 verification iterations, 1 design revision, 2 review rounds
  - [x] AC12 — Security blocker policy present (L706–714)
  - [x] AC13 — Pipeline state model (in-context) documented with fields (L88–100)
  - [x] AC14 — Pipeline recovery (EC-5) scanning logic documented (L105–113)
  - [x] AC15 — Auto-commit (Step 9) documented with Confidence gate (L520–530)
  - [x] AC16 — Completion contract: DONE | ERROR (L722–728)
  - [x] AC17 — Self-verification (L733) and anti-drift anchor (L747) present
- **Tests:** N/A — documentation-only task
- **Issues:** None

### Task 12: Feature Workflow Prompt

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/.github/prompts/feature-workflow.prompt.md` (40 lines)
  - [x] AC2 — YAML frontmatter includes `agent: orchestrator` (L3). Note: uses `name` and `description` instead of `mode: agent` — adapted for VS Code prompt file conventions (documented in task notes)
  - [x] AC3 — Variables defined: `{{USER_FEATURE}}` (L11, required) and `{{APPROVAL_MODE}}` (L15, default: autonomous stated in description)
  - [x] AC4 — Key artifacts reference table present (L17–30) — points to correct paths under `.github/`
  - [x] AC5 — Pipeline execution rules section present (L32–36) referencing `orchestrator.agent.md`
  - [x] AC6 — `{{VARIABLE_NAME}}` syntax used correctly
  - [x] AC7 — File is concise (40 lines) and focused — entry point only
- **Tests:** N/A — configuration-only task
- **Issues:** None

### Task 13: README Documentation

- **Status:** verified
- **Acceptance Criteria:**
  - [x] AC1 — File exists at `NewAgents/README.md` (179 lines)
  - [x] AC2 — Mermaid pipeline diagram present (L15–52) matching Steps 0–9, includes S3b, S5R (replan), S7F (fix), S8b, approval gates AG1/AG2, both loops (design revision max 1, verification max 3, code review max 2)
  - [x] AC3 — Agent inventory table lists all 9 agents (L66–76): Orchestrator, Researcher, Spec, Designer, Planner, Implementer, Verifier, Adversarial Reviewer, Knowledge Agent — with name, role, pipeline step, key capability
  - [x] AC4 — Reference documents section lists 3 shared references (L163): schemas.md, dispatch-patterns.md, severity-taxonomy.md
  - [x] AC5 — Quick-start section (L133–147) explains how to use `feature-workflow.prompt.md`
  - [x] AC6 — Directory structure overview present (L149–162)
  - [x] AC7 — Mermaid syntax valid (proper `flowchart TD` with node definitions and edges)
- **Tests:** N/A — documentation-only task
- **Issues:** None

## Failing Task IDs

None — all 13 tasks verified.

## Issues Found

None.

## Cross-Cutting Observations

- All 14 output files are documentation/configuration artifacts (no compiled code), so no build or test verification was applicable. V-Build confirmed file existence and structural correctness; V-Tasks confirmed content meets acceptance criteria.
- The prompt file (Task 12) adapted YAML frontmatter syntax from `mode: agent` to `name`/`description` due to VS Code prompt file limitations. The `agent: orchestrator` binding is correctly present. This adaptation was documented in the task file's Notes section.
- The researcher agent's tool restriction text (L229) says "MUST NOT use `create_file`... except to write your own output files" which is slightly ambiguous about whether it can use `create_file` for outputs, but the tool access table correctly lists only 5 read-only tools per design intent.
