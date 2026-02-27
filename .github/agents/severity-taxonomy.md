# Severity Taxonomy — Reference

This is a reference document for all agents. Defines the unified severity taxonomy used
across all pipeline stages. No agent may use a different severity vocabulary.

> **Authority:** feature.md CR-13 — Unified Severity Taxonomy.

---

## Severity Levels

### Blocker

**Definition:** Security vulnerability, data loss risk, or fundamental design flaw that makes the artifact unsafe to proceed with.

**Pipeline impact:** Immediate pipeline halt. No retry, no deferral, no downgrade.

**Examples:**

- SQL injection vulnerability in generated code
- Authentication bypass in auth module
- Hardcoded secrets or credentials in source
- Data loss or corruption path in storage logic
- Cryptographic weakness (e.g., broken algorithm, weak key)

**Resolution:** Must fix the issue and re-run the review. The only path forward is remediation.

---

### Critical

**Definition:** Severe functional issue, unrecoverable state risk, or fundamental correctness problem that must be addressed before the pipeline can complete.

**Pipeline impact:** Mandatory fix before proceeding. Routes to `NEEDS_REVISION` with priority.

**Examples:**

- Logic error that produces incorrect results in core functionality
- Missing error handling that leads to unrecoverable state
- Specification violation in a required behavior
- Race condition or concurrency bug in shared resource access
- Schema/API contract violation breaking downstream consumers

**Resolution:** Implementer addresses the finding. Re-verify after fix. If found during review: triggers `NEEDS_REVISION` routing.

---

### Major

**Definition:** Significant quality or correctness issue that should be addressed but does not prevent pipeline progression.

**Pipeline impact:** `NEEDS_REVISION` routing — fix is expected but pipeline can proceed if iteration budget is exhausted.

**Examples:**

- Missing test coverage for a specified acceptance criterion
- Suboptimal algorithm with measurable performance impact
- Inconsistent naming or API convention violation
- Missing input validation on non-security boundary
- Incomplete documentation for a public interface

**Resolution:** Fix during current iteration if replan budget allows. If iteration limit reached, log as known issue with `confidence: Low`.

---

### Minor

**Definition:** Style issue, optimization opportunity, or minor inconsistency with no functional impact.

**Pipeline impact:** Log and continue. No pipeline routing change.

**Examples:**

- Code style inconsistency (formatting, naming preference)
- Optimization opportunity with negligible real-world impact
- Minor documentation typo or wording improvement
- Redundant import or unused variable
- Cosmetic UI inconsistency

**Resolution:** Logged as finding. Optionally addressed if time permits. Never blocks pipeline.

---

## Security Blocker Policy

> **Source:** design.md §Decision 7 — inherited from Forge's r-security pattern, elevated to pipeline-level.

**Any security finding with `severity: Blocker` from ANY review model → immediate pipeline ERROR.**

This policy applies to:

- **Design review (Step 3b):** Any reviewer finding a security Blocker halts the pipeline.
- **Code review (Step 7):** Any reviewer finding a security Blocker halts the pipeline.
- **Early pushback (Step 0):** Blocker-level concern in pushback evaluation halts before researcher dispatch.

**Rules:**

1. Security Blockers are **NEVER** downgraded to a lower severity.
2. Security Blockers are **NEVER** deferred to "known issues."
3. Security Blockers are **NEVER** overridden by majority-approve verdicts.
4. A single Blocker from a single reviewer is sufficient to halt — regardless of other verdicts.
5. The only path forward is fixing the security issue and re-running the review.

**SQL detection:**

```sql
SELECT COUNT(*) FROM anvil_checks
WHERE run_id = '{run_id}' AND round = {r} AND verdict = 'blocker';
-- If result > 0 → pipeline ERROR (immediate halt)
```

---

## Severity-Driven Routing Summary

| Severity | Pipeline Action           | Retry Allowed | Proceeds Without Fix  | Confidence Impact |
| -------- | ------------------------- | ------------- | --------------------- | ----------------- |
| Blocker  | Immediate pipeline ERROR  | No            | Never                 | N/A (halted)      |
| Critical | NEEDS_REVISION (priority) | Yes (1)       | No                    | Low if unresolved |
| Major    | NEEDS_REVISION            | Yes (1)       | After iteration limit | Low               |
| Minor    | Log and continue          | N/A           | Always                | None              |

---

## Severity Usage Table

This table documents which agents produce severity values and where they appear.

| Agent / Role         | Produces Severity | Where Severity Appears                                       | Notes                                    |
| -------------------- | ----------------- | ------------------------------------------------------------ | ---------------------------------------- |
| Adversarial Reviewer | Yes               | Review findings (Markdown), verdict YAML, `anvil_checks` SQL | Per-finding severity + overall verdict   |
| Verifier             | Yes               | Verification reports, `anvil_checks` SQL                     | Per-check severity for failed checks     |
| Spec Agent           | Yes (pushback)    | Pushback findings in spec output                             | Severity on pushback concerns            |
| Planner              | Indirect          | Risk classification (🟢/🟡/🔴) drives task sizing            | Maps to verification depth, not severity |
| Implementer          | No                | N/A                                                          | Consumer of severity (from verifier)     |
| Designer             | No                | N/A                                                          | Consumer of severity (from reviewers)    |
| Researcher           | No                | N/A                                                          | Does not produce quality judgments       |
| Knowledge Agent      | Indirect          | Evidence bundle aggregates severity from other agents        | Aggregator, not originator               |
| Orchestrator         | Indirect          | SQL gate queries check for blocker/approve verdicts          | Reads severity for routing, doesn't set  |

### Completion Contract Integration

Every agent's completion contract includes a severity field:

```yaml
completion:
  status: DONE | NEEDS_REVISION | ERROR
  severity: Blocker | Critical | Major | Minor | null
  # severity is required for review/verification agents
  # severity is null for non-review agents
```

The `severity` field in the completion contract represents the **highest severity finding** in that agent's output. The orchestrator uses this for routing decisions.

---

## Relationship to Risk Classification

The severity taxonomy (Blocker/Critical/Major/Minor) is **distinct from** the risk classification system (🟢/🟡/🔴):

| System              | Purpose                       | Set By               | Applies To         |
| ------------------- | ----------------------------- | -------------------- | ------------------ |
| Severity Taxonomy   | Quality of artifacts/code     | Reviewers, Verifiers | Findings, verdicts |
| Risk Classification | Inherent risk of file changes | Planner              | Files, tasks       |

Risk classification drives **verification depth** (standard vs. large). Severity taxonomy drives **pipeline routing** (proceed, revise, halt). They are complementary but do not substitute for each other.
