# Artifact Evaluations: ct-security

```yaml
artifact_evaluation:
  evaluator: "ct-security"
  source_artifact: "design.md"
  usefulness_score: 9
  clarity_score: 9
  useful_elements:
    - "§Security Considerations — explicit threat model with likelihood/impact/mitigation table"
    - "§Decision 4 — honest disclosure that schema validation is prompt-level only"
    - "§Decision 2 — detailed tool restrictions per agent with explicit MUST NOT lists"
    - "§Decision 7 — Security Blocker Policy with clear escalation path"
    - "§Decision 6 — SQL and YAML schema definitions for verification ledger with evidence gating queries"
    - "§Decision 8 — Risk classification criteria with escalation rules"
    - "§Failure & Recovery — comprehensive failure mode catalog with retry budgets"
  missing_information:
    - "No parameterized query mandate for SQL operations — design shows string-interpolated patterns but doesn't specify safe query construction"
    - "No YAML safe-parsing specification — no restriction on YAML features (anchors, aliases, custom tags) that could be exploited"
    - "No schema versioning protocol — schema_version: 1.0 exists but no migration, compatibility, or deprecation rules"
    - "No independent validation of risk classification — single-agent heuristic with cascading security effects"
    - "No specification for whether model routing fallback should block 🔴 work or proceed with degraded confidence"
  information_not_used:
    - "§Decision 9 (Approval Mode) — detailed approval prompt examples were not relevant to security analysis beyond mode-switching behavior"
    - "§Decision 10 (File & Output Structure) — file naming conventions and directory layout had minimal security implications"
    - "§Testing Strategy — test cases are for future validation, not current security posture"
  inaccuracies:
    - "Threat model rates 'Agent writes outside its file boundaries' as Low likelihood, but the prior orchestrator-tool-restriction feature confirmed tool enforcement is advisory-only — this should be Low-Medium"
    - "Threat model mitigation for LLM hallucinated verification ('Gates check COUNT, not prose claims') doesn't address fabricated structured records, only prose fabrication"
  impact_on_work:
    - "The honest disclosure that validation is prompt-level only (§Decision 4) was the most impactful element — it enabled the primary finding about enforcement gaps"
    - "Had to cross-reference the prior orchestrator-tool-restriction feature to confirm that YAML tools: field is advisory-only — this isn't stated in the current design"
```

```yaml
artifact_evaluation:
  evaluator: "ct-security"
  source_artifact: "feature.md"
  usefulness_score: 7
  clarity_score: 8
  useful_elements:
    - "§NFR-7 (Security) — clear requirement that security findings must halt pipeline"
    - "§CR-5 (Evidence Gating) — machine-checkable gates requirement helped identify the COUNT-vs-truth gap"
    - "§CR-6 (Risk Classification) — per-file classification requirement with 🔴 escalation rules"
    - "§EC-3 through EC-9 — edge cases directly relevant to security failure scenarios"
    - "§FR-7.5 — explicit requirement that security findings halt regardless of retry budget"
  missing_information:
    - "NFR-7 (Security) is a single paragraph — no detailed security requirements beyond 'security findings halt pipeline' and 'security blocker policy enforced'"
    - "No requirement for runtime enforcement of tool restrictions or schema validation — the spec accepts prompt-level as sufficient without stating this explicitly"
    - "No requirement for verification evidence integrity (tamper detection, chain of custody)"
    - "No requirement for independent risk classification validation"
  information_not_used:
    - "§User Stories — US-1/US-2/US-3 flow descriptions were redundant with design.md pipeline structure"
    - "§Architectural Directions comparison matrix — already consumed via design.md's synthesis analysis"
    - "§Test Scenarios TS-1 through TS-20 — future test cases, not current security analysis inputs"
  inaccuracies:
    - "N/A — none identified"
  impact_on_work:
    - "Feature.md's edge cases (EC-3, EC-4, EC-9) provided the security-relevant failure scenarios that informed findings about schema validation, ledger corruption, and model routing fallback"
```
