# Critical Review: Security & Backwards Compatibility

## Findings

### [Severity: High] Schema Validation Is Prompt-Level Only — No Programmatic Enforcement

- **What:** The entire typed YAML schema validation system — the backbone of the design's security and reliability story — is enforced by instructing the orchestrator LLM to check required fields. Design.md §Decision 4 explicitly states: _"Validation is prompt-level (the orchestrator is instructed to check required fields), not runtime-enforced. This is a pragmatic concession to the VS Code agent framework's capabilities."_ This means schema validation can fail silently if the orchestrator LLM drifts, hallucinates, or is overwhelmed by context. A malformed agent output that passes "validation" could propagate downstream unchecked, causing incorrect routing, false evidence gating results, or cascading agent failures.
- **Where:** design.md §Decision 4 (Communication & State), §Security Considerations (Input Validation Strategy)
- **Likelihood:** Medium — LLM context drift is acknowledged in the threat model; complex pipelines increase the probability the orchestrator skips validation steps
- **Impact:** High — schema validation is the primary defense against malformed inter-agent communication. Its failure undermines typed YAML's entire value proposition over Markdown prose.
- **Assumption at risk:** That instructing an LLM to "validate required fields" is equivalent to actual validation. The design's core thesis — typed schemas make communication reliable — rests on enforcement that doesn't exist at runtime.

### [Severity: High] Tool Restrictions Are Advisory-Only — No Runtime Enforcement

- **What:** All agent tool access controls are enforced via prompt instructions in `.agent.md` files. The prior feature `orchestrator-tool-restriction` confirmed that VS Code's YAML `tools:` field is advisory-only with no runtime enforcement (see `docs/feature/orchestrator-tool-restriction/design.md` §13.3). This means: the Orchestrator could call `create_file` or `run_in_terminal` despite being restricted to read-only+dispatch; Researchers and Reviewers could modify files despite read-only constraints; the Verifier could modify source code, breaking verification independence; any agent could write outside its designated file boundaries. The design lists tool restrictions per agent (design.md §Decision 2) but treats them as security controls without acknowledging the enforcement gap in its threat model.
- **Where:** design.md §Decision 2 (Agent Detail: tools and tool restrictions for all 9 agents), §Security Considerations (Agent-level access control)
- **Likelihood:** Low-Medium — requires context drift, which is mitigated by anti-drift anchors; but the Implementer agent (with full read-write access and `run_in_terminal`) is the most likely to act outside boundaries during complex multi-file tasks
- **Impact:** High — a Researcher/Reviewer writing files could corrupt pipeline state; a Verifier modifying source breaks the independence assumption that underlies the entire verification architecture; the orchestrator writing files directly bypasses the delegation model
- **Assumption at risk:** That prompt-level tool restrictions provide meaningful access control. The design's threat model rates "Agent writes outside its file boundaries" as Low likelihood / Medium impact, but the absence of runtime enforcement makes this an accepted rather than mitigated risk.

### [Severity: High] Verification Ledger Has No Integrity Protection

- **What:** The verification ledger (SQL or YAML) is the foundation of evidence gating. But both the Implementer (writes `phase: baseline`) and the Verifier (writes `phase: after`) contribute records to the same ledger, and there is no cryptographic integrity, chain-of-custody, or tamper detection. COUNT-based evidence gates (e.g., `SELECT COUNT(*) FROM verification_ledger WHERE phase='after' >= 2`) verify that ENOUGH records exist, not that the records are TRUTHFUL. An agent could fabricate verification records — the Implementer could write fake baseline records, the Verifier could claim checks passed that were never run. In YAML fallback mode, the ledger is an append-only file with no write-access restriction, so any file-writing agent could add entries. The design's own threat model identifies "LLM hallucinating verification results" as Medium likelihood / High impact, but the mitigation ("Gates check COUNT, not prose claims") does not address fabricated structured records — only fabricated prose.
- **Where:** design.md §Decision 6 (Verification Architecture — Evidence Gating Mechanism, SQL/YAML schemas), §Security Considerations (Threat Model row 1)
- **Likelihood:** Medium — LLMs hallucinating tool execution results is a known failure mode; the shift from prose to structured records helps but doesn't eliminate fabrication
- **Impact:** High — fabricated verification evidence undermines the entire evidence-gating strategy, allowing unverified code to pass through the pipeline with false assurance
- **Assumption at risk:** That COUNT-based gating on agent-generated records is tamper-resistant. The design assumes agents will honestly record verification results because they're structured INSERT/records rather than prose, but the agent controls both the execution and the recording.

### [Severity: Medium] Completion Contract Spoofing — No Independent Status Verification

- **What:** Every agent self-reports its completion status via the `completion-contract.schema`. There is no independent verification that an agent's claimed status matches reality. An agent could return `status: DONE` with `findings_count: 0` and `security_blockers: 0` while having found actual issues. The Adversarial Reviewer could return `overall_verdict: approve` despite finding security problems. The design's security blocker policy ("Any security finding with severity: Blocker from any review model → immediate pipeline ERROR") depends entirely on the reviewer agent honestly reporting the finding — if the reviewer hallucinate-approves, the policy has no fallback.
- **Where:** design.md §Decision 4 (Completion Contract Schema), §Decision 7 (Security Blocker Policy)
- **Likelihood:** Low-Medium — most LLM failures produce errors or malformed output rather than plausible-but-false completions; but approval hallucination is a known LLM failure mode
- **Impact:** High — a spoofed DONE from the Verifier or a spoofed approve from the Adversarial Reviewer bypasses the two primary quality gates in the pipeline
- **Assumption at risk:** That LLM agents will accurately self-assess the quality and completeness of their own work. The self-verification step (CR-11) helps but is itself prompt-level.

### [Severity: Medium] SQL Injection Risk in Verification Ledger

- **What:** The verification ledger SQL schema accepts string fields (`task_id TEXT`, `check_name TEXT`, `command TEXT`, `output_snippet TEXT`) and the design shows string-interpolated SQL patterns (e.g., `WHERE task_id = '{task_id}'` in Anvil's existing SQL at `Anvil/anvil.agent.md` lines 263-275). The `output_snippet` field stores the first 500 chars of command output, which could contain crafted SQL if a build tool, test framework, or dependency outputs adversarial content. Since agents construct SQL statements (not the runtime), an LLM constructing a query with unescaped string interpolation creates a SQL injection vector. The design does not mandate parameterized queries, prepared statements, or input sanitization for any SQL operations.
- **Where:** design.md §Decision 6 (SQL Schema, YAML Schema), Anvil/anvil.agent.md lines 61-75 and 263-275 (existing SQL patterns being carried forward)
- **Likelihood:** Low — requires adversarial content in build/test output that the LLM passes unescaped into SQL; most LLMs use safe patterns by default
- **Impact:** Medium — SQL injection could corrupt the verification ledger, fabricate evidence records, or drop the table entirely. In a single-user local agent context, the blast radius is limited to the pipeline run.
- **Assumption at risk:** That LLMs will always use parameterized queries when constructing SQL. The existing Anvil patterns use string interpolation (`'{task_id}'`), and the new design inherits these patterns.

### [Severity: Medium] Risk Classification Has No Independent Validation

- **What:** The per-file risk classification (🟢/🟡/🔴) is assigned by a single agent (Planner) based on heuristic criteria. There is no second opinion, no independent validation, and no cross-check against the actual file content. Under-classification has cascading security effects: a 🔴 file (auth/crypto/payments) misclassified as 🟡 receives only 1 adversarial reviewer instead of 3, only 2 verification signals instead of 3, and avoids the Large verification depth. The classification criteria in design.md §Decision 8 are examples ("auth/crypto/payments, data deletion, schema migrations"), not exhaustive rules — the Planner LLM must infer classification from file paths and context.
- **Where:** design.md §Decision 8 (Risk Classification System, Classification Rules), §Decision 7 (Trigger Conditions — reviewer count depends on risk)
- **Likelihood:** Medium — file path heuristics miss semantic criticality (e.g., `utils.ts` containing auth logic); LLMs may under-classify to reduce pipeline complexity
- **Impact:** Medium — under-classification directly reduces verification depth and adversarial review coverage for the highest-risk files
- **Assumption at risk:** That a single agent can reliably classify file-level risk from paths and descriptions. No cross-validation exists.

### [Severity: Medium] Knowledge Agent Can Modify Instruction Files Without Approval in Autonomous Mode

- **What:** The Knowledge Agent has the capability to modify instruction files (`.github/instructions/`). In autonomous mode, these modifications are logged but not gated by any approval mechanism. The design mentions a "safety constraint filter prevents weakening existing security checks" but this filter is itself prompt-level. A drifted or hallucinating Knowledge Agent could: add permissive rules to instruction files, weaken existing security constraints, introduce patterns that cause cascading issues in future pipeline runs. Additionally, the Knowledge Agent's `decisions.yaml` is append-only, but nothing prevents the agent from using `replace_string_in_file` to modify existing entries.
- **Where:** design.md §Decision 2 (Knowledge Agent detail — "Governed updates"), §Security Considerations (Data Protection — Knowledge Agent governance)
- **Likelihood:** Low — Knowledge Agent operates at the end of the pipeline with full context, reducing drift risk; but autonomous mode has no human check
- **Impact:** Medium — instruction file modifications persist across pipeline runs, meaning a single bad modification could affect all future executions
- **Assumption at risk:** That prompt-level safety filters reliably prevent instruction weakening. The "safety constraint filter" has no programmatic enforcement.

### [Severity: Medium] YAML Parsing Safety Not Specified

- **What:** The entire inter-agent communication system uses YAML, but the design does not specify safe parsing requirements. YAML supports complex features (anchors, aliases, merge keys, custom tags) that can be exploited for "billion laughs" attacks or unexpected type coercion. While the orchestrator "reading" YAML is an LLM (not a programmatic parser), any future programmatic tooling, test harnesses, or CI/CD integration that parses these YAML files could be vulnerable. The design should mandate a safe-parsing subset (e.g., no custom tags, no anchors/aliases) and specify that all schemas use only scalar types, sequences, and mappings.
- **Where:** design.md §Decision 4 (Typed YAML selection), §Schemas Reference (14 schema definitions)
- **Likelihood:** Low — current system is LLM-to-LLM with no programmatic YAML parsing; risk increases if tooling evolves
- **Impact:** Medium — YAML deserialization attacks in programmatic tooling could cause denial-of-service or arbitrary code execution
- **Assumption at risk:** That YAML is safe by default. The design chose YAML for readability and LLM familiarity without addressing parser safety.

### [Severity: Medium] File Path Boundaries Are Unenforceable

- **What:** Agents are assigned specific output paths (e.g., Researcher writes to `research/<focus>.yaml`, Implementer writes to `implementation-reports/<task-id>.yaml`). But since tool restrictions are prompt-level only and agents have access to `create_file`/`replace_string_in_file`, there is no mechanism preventing an agent from writing to the pipeline manifest, other agents' output files, or the verification ledger. The YAML-fallback verification ledger is particularly vulnerable since it's a regular YAML file that any file-writing agent could append to or modify.
- **Where:** design.md §Decision 10 (Pipeline Runtime File Structure), §Security Considerations (Threat Model — "Agent writes outside its file boundaries")
- **Likelihood:** Low — requires agent drift combined with hallucinated file paths; agents typically follow their prescribed paths
- **Impact:** Medium — writing to the pipeline manifest could corrupt routing; writing to the verification ledger could fabricate evidence; overwriting other agents' outputs could cause silent data loss
- **Assumption at risk:** That agents will self-enforce file path boundaries via prompt compliance.

### [Severity: Low] Model Routing Fallback Eliminates Adversarial Diversity for High-Risk Work

- **What:** When multi-model routing is unavailable (EC-9), the fallback is same-model review with different prompts. For 🔴-classified work, this means the three-reviewer security guarantee degrades to three instances of the same model reacting to different prompt framings. Correlated failures (all three miss the same vulnerability) become significantly more likely with a single model. The design acknowledges "Reduce quality guarantee" but doesn't specify whether 🔴-classified work should refuse to proceed without genuine multi-model review, or whether the security blocker policy still applies with degraded confidence.
- **Where:** design.md §Decision 7 (Model Routing Fallback — EC-9), feature.md §EC-9
- **Likelihood:** Medium — model routing availability is platform-dependent and explicitly uncertain (Assumption A-2)
- **Impact:** Medium — correlated blind spots in same-model review could miss security vulnerabilities in critical files
- **Assumption at risk:** That same-model-different-prompt provides meaningful adversarial diversity. The design treats it as degraded-but-functional without quantifying the degradation for 🔴 work.

### [Severity: Low] Schema Versioning Absent — Future Backwards Compatibility Risk

- **What:** Agent output schemas include `schema_version: "1.0"` but the design specifies no versioning protocol: no migration strategy between versions, no orchestrator version checking, no forward/backward compatibility rules, no deprecation process. The first schema change will be an uncharted process. If the orchestrator expects `schema_version: "1.0"` fields and receives `"2.0"`, there's no defined behavior. If a pipeline resumes with old-format outputs, there's no compatibility guarantee.
- **Where:** design.md §Decision 4 (Agent Output Schema Pattern — `schema_version: "1.0"`), §Schemas Reference
- **Likelihood:** High — schemas will evolve as the system matures
- **Impact:** Low — impacts future maintainability rather than current security; no existing consumers to break today
- **Assumption at risk:** That schema version 1.0 will be sufficient indefinitely, or that versioning can be added later without pain.

### [Severity: Low] Autonomous Mode Removes All Human Security Gates

- **What:** In autonomous mode, all approval gates auto-proceed with default selections, pushback concerns are logged but never block, and the entire pipeline can run with 🔴 classifications and security concerns without any human review. The design intends this, and security blockers from adversarial review still halt the pipeline programmatically (via the security blocker policy). But the Spec pushback system — designed to catch dangerous edge cases and implicit assumptions — has zero blocking power in autonomous mode.
- **Where:** design.md §Decision 9 (Mode Switching Behavior), §Decision 3 (Step 2 — Pushback "auto-log, proceed")
- **Likelihood:** High — autonomous mode is the default (`default: "autonomous"` in the prompt file)
- **Impact:** Low — the adversarial review + security blocker policy provides an automated safety net; the risk is that pushback concerns about security implications are ignored in the most common execution mode
- **Assumption at risk:** That automated security blocker detection by adversarial reviewers is sufficient without human review of pushback concerns.

---

## Cross-Cutting Observations

- **Scalability concern (CT-Scalability scope):** The unified Verifier agent (merging V-Build, V-Tests, V-Tasks, V-Feature) must execute the full tiered cascade within a single context window. For complex features with 10+ tasks, the Verifier's context may overflow with cumulative build output, test results, and evidence records. The design acknowledges this as "the most complex single agent" but provides no context budget or overflow strategy specific to the Verifier.

- **Maintainability concern (CT-Maintainability scope):** The 14-schema reference document (`schemas.md`) is a single point of failure for all agent contracts. A schema error in this shared document could silently break multiple agents. No schema-per-agent alternative was evaluated for resilience.

- **Strategy concern (CT-Strategy scope):** The design's security model accepts prompt-level enforcement as the ceiling for all security controls — tool restrictions, schema validation, file boundaries, safety filters. This is an honest assessment of the VS Code agent framework's limitations, but raises the question: should the design explicitly prioritize runtime enforcement for any controls (e.g., via a lightweight validation script that the orchestrator runs via `run_in_terminal` before routing), or is prompt-level the permanent architectural ceiling?

---

## Requirement Coverage

| Requirement                             | Coverage Status   | Notes                                                                                                   |
| --------------------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------- |
| NFR-7: Security                         | Partially covered | Security blocker policy is well-defined; but all enforcement is prompt-level with no runtime validation |
| CR-5: Evidence Gating                   | Covered with risk | COUNT-based gates are specified; but COUNT verifies quantity, not truthfulness of records               |
| CR-6: Risk Classification               | Covered with risk | Per-file classification specified; but single-agent assignment with no independent validation           |
| CR-11: Self-Verification                | Covered           | Every agent has self-verification step; but self-verification is itself prompt-level                    |
| CR-12: Anti-Drift Anchors               | Covered           | Every agent definition includes anti-drift anchor; this is the primary defense against context drift    |
| CR-15: Bounded Retry Budget             | Covered           | All retry/loop bounds explicitly specified with worst-case analysis                                     |
| FR-1.6: Orchestrator Tool Restrictions  | Covered with risk | Tools enumerated but enforcement is advisory-only per prior feature analysis                            |
| FR-4.3: Structured Evidence Records     | Covered with risk | SQL/YAML evidence records specified; but records are agent-generated with no integrity check            |
| FR-5.4: Different LLM Models            | Partially covered | Three models specified; fallback to same-model weakens the guarantee without blocking 🔴 work           |
| FR-7.5: Security Findings Halt Pipeline | Covered           | Security blocker policy clearly specified at pipeline level                                             |
| EC-3: Invalid Output Format             | Covered           | Schema validation + retry specified; but validation is prompt-level                                     |
| EC-4: SQL Ledger Corruption             | Covered           | YAML fallback specified; but YAML fallback has weaker access control (regular file)                     |
| EC-9: Model Routing Unavailable         | Partially covered | Fallback exists but doesn't address degraded security confidence for 🔴 work                            |
