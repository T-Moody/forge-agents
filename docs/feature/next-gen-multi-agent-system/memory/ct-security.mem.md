# Memory: ct-security

## Status

DONE: security — 12 findings (3 High, 6 Medium, 3 Low). Core risk: all security controls (schema validation, tool restrictions, file boundaries, verification integrity) are prompt-level only with no runtime enforcement. Evidence gating verifies quantity not truthfulness. Risk classification has no independent validation.

## Key Findings

- Schema validation and tool restrictions are both prompt-level only — the design's security model has no runtime enforcement layer, making every typed-schema guarantee dependent on LLM compliance
- Verification ledger integrity is unprotected — COUNT-based evidence gates verify record quantity, not truthfulness, and agents self-generate the records they're gated on
- Risk classification by a single agent (Planner) with no independent validation means under-classification cascades to reduced verification depth and reviewer count for critical files
- Knowledge Agent can modify instruction files without approval in autonomous mode, with only a prompt-level safety filter preventing weakening of security constraints
- SQL injection risk from string-interpolated queries inherited from Anvil patterns; no parameterized query mandate exists

## Highest Severity

High (3 findings: prompt-level-only schema validation, advisory-only tool restrictions, unprotected verification ledger integrity)

## Decisions Made

(none — CT-Security identifies problems, does not make decisions)

## Artifact Index

- [ct-review/ct-security.md](../ct-review/ct-security.md)
  - §Finding 1: Schema Validation Prompt-Level Only — core design thesis undermined by enforcement gap
  - §Finding 2: Tool Restrictions Advisory-Only — all agent access control is prompt compliance
  - §Finding 3: Verification Ledger No Integrity Protection — COUNT gates verify quantity not truth
  - §Finding 4: Completion Contract Spoofing — agents self-report status without independent check
  - §Finding 5: SQL Injection Risk — string interpolation in agent-constructed queries
  - §Finding 6: Risk Classification No Independent Validation — single-agent heuristic, cascading effects
  - §Finding 7: Knowledge Agent Autonomous Instruction Modification — no approval gate
  - §Finding 8: YAML Parsing Safety Not Specified — no safe-parsing subset mandated
  - §Finding 9: File Path Boundaries Unenforceable — no programmatic write-path restriction
  - §Finding 10: Model Routing Fallback — same-model review for 🔴 work
  - §Finding 11: Schema Versioning Absent — no migration/compatibility protocol
  - §Finding 12: Autonomous Mode Removes Human Security Gates — pushback non-blocking
  - §Cross-Cutting Observations — Verifier context overflow, schemas.md single point of failure, runtime enforcement ceiling
  - §Requirement Coverage — 13 requirements mapped with coverage status
