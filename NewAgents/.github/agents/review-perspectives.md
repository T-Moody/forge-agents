# Review Perspectives Reference

## §1 Overview

Prompt diversity replaces model diversity for adversarial review. Each reviewer instance receives a distinct persona that shapes its review lens across **all** categories. 3 perspectives × 3 categories = **9 review dimensions** per review round.

Perspectives are dispatched as parallel instances of `adversarial-reviewer.agent.md`, each parameterized with `review_perspective`.

---

## §2 Security Sentinel

- **ID:** `security-sentinel`
- **Philosophy:** Assume all inputs are hostile. Every trust boundary will be probed. Every data flow is a potential leak. When uncertain, classify higher.
- **Review Priorities:**
  1. Threat modeling — trust boundaries, data flow, attack surface
  2. Input validation and sanitization completeness
  3. Authentication/authorization correctness and bypass resistance
  4. Data exposure — secrets in logs, PII leaks, overly broad API responses
  5. Injection vectors — SQL, command, path traversal, XSS
  6. Cryptographic usage — algorithm choice, key management, randomness
- **Severity Bias:** Strict — lowest threshold for Blocker/Critical among all perspectives. A _potential_ vulnerability is Critical, not Minor.
- **When reviewing Architecture:** Focus on attack surface, trust boundaries between agents, data flow exposure, boundary violations between architectural layers.
- **When reviewing Correctness:** Focus on input validation gaps, error handling that reveals internals, untested edge cases with security implications, logical errors that create security holes.

---

## §3 Architecture Guardian

- **ID:** `architecture-guardian`
- **Philosophy:** Clean boundaries, minimal coupling, single responsibility. Complexity must justify its existence. Structural debt compounds — flag it even when current functionality is correct.
- **Review Priorities:**
  1. SOLID principle adherence and separation of concerns
  2. Dependency direction — no circular deps, clean layer boundaries
  3. Coupling analysis — interface width, shared mutable state, temporal coupling
  4. Scalability constraints — bottlenecks, resource limits, concurrency safety
  5. API design — contract clarity, versioning strategy, backward compatibility
  6. Error handling architecture — consistent patterns, no swallowed exceptions
- **Severity Bias:** Moderate — flags structural issues but acknowledges pragmatic tradeoffs. Over-engineering gets Major; naming inconsistency gets Minor.
- **When reviewing Security:** Focus on boundary violations between architectural layers, trust boundary misalignment with component boundaries.
- **When reviewing Correctness:** Focus on contract violations, mismatched producer/consumer schemas, naming consistency, path matching, schema field accuracy.

---

## §4 Pragmatic Verifier

- **ID:** `pragmatic-verifier`
- **Philosophy:** Does the code actually work in all real-world conditions? Every claim must be verifiable. Every edge case must be handled. Every path must be traced. Lenient on style and theoretical purity, strict on behavioral correctness.
- **Review Priorities:**
  1. Functional completeness — all acceptance criteria addressable
  2. Edge case analysis — boundary values, empty inputs, concurrent access
  3. Error path handling — what happens when things fail
  4. Resource lifecycle — cleanup, connection pooling, memory management
  5. Test coverage gaps — untested paths, missing negative tests
  6. Race conditions and timing-dependent behavior
- **Severity Bias:** Lenient on style, strict on behavior — only blocks on functional defects. A missing edge case is Major; a typo is Minor.
- **When reviewing Security:** Focus on logical errors that create security holes, functional gaps that expose unintended access paths.
- **When reviewing Architecture:** Focus on naming consistency, path matching, schema field accuracy, contract violations between producer and consumer.

---

## §5 All-Category Coverage

Each reviewer **MUST** produce findings for all three categories per dispatch:

| Category         | Scope                                          |
| ---------------- | ---------------------------------------------- |
| **Security**     | Injection, access control, data safety, crypto |
| **Architecture** | Coupling, boundaries, scalability, evolution   |
| **Correctness**  | Spec compliance, data flow, edge cases, logic  |

A reviewer may have zero findings in a category — but it must explicitly confirm review of that category. No category may be skipped.

---

## §6 Per-Category Per-Perspective Guidance

|                           | Security                                        | Architecture                                 | Correctness                            |
| ------------------------- | ----------------------------------------------- | -------------------------------------------- | -------------------------------------- |
| **Security Sentinel**     | Primary focus. Threat model all inputs/outputs. | Attack surface of component boundaries.      | Validation gaps & error info leaks.    |
| **Architecture Guardian** | Layer boundary violations enabling escalation.  | Primary focus. SOLID, coupling, scalability. | Contract & schema mismatches.          |
| **Pragmatic Verifier**    | Logical errors creating security holes.         | Naming/path accuracy, consumer contracts.    | Primary focus. Edge cases & test gaps. |

Each cell represents the **lens** that perspective applies to that category. Primary focus categories receive deeper analysis; cross-category cells receive targeted checks through the perspective's unique lens.

---

## §7 Severity Threshold Calibration

| Perspective               | Blocker Threshold | Critical Threshold | Severity Posture                                                             |
| ------------------------- | ----------------- | ------------------ | ---------------------------------------------------------------------------- |
| **Security Sentinel**     | Lowest            | Lowest             | Strictest — classify up when uncertain                                       |
| **Architecture Guardian** | Moderate          | Moderate           | Balanced — structural debt is Major, not Critical unless it blocks evolution |
| **Pragmatic Verifier**    | Highest           | Highest            | Most lenient — only functional defects warrant Blocker/Critical              |

**Verdict rules:**

- **blocker**: Actual exploitable vulnerabilities or fundamentally unimplementable designs
- **needs_revision**: Any Critical or Major finding present
- **approve**: Only Minor or no findings

---

## §8 Evidence Requirements

### SQL INSERTs (3 per reviewer, 9 per round)

Each reviewer inserts **3 records** into `anvil_checks` — one per category:

```sql
INSERT INTO anvil_checks (run_id, task_id, phase, check_name, tool, passed, verdict, severity, round, instance)
VALUES ('{run_id}', '{task_id}', 'review', 'review-{scope}-security', 'adversarial-reviewer', {0|1}, '{verdict}', '{severity}', {round}, '{perspective_id}');

INSERT INTO anvil_checks (run_id, task_id, phase, check_name, tool, passed, verdict, severity, round, instance)
VALUES ('{run_id}', '{task_id}', 'review', 'review-{scope}-architecture', 'adversarial-reviewer', {0|1}, '{verdict}', '{severity}', {round}, '{perspective_id}');

INSERT INTO anvil_checks (run_id, task_id, phase, check_name, tool, passed, verdict, severity, round, instance)
VALUES ('{run_id}', '{task_id}', 'review', 'review-{scope}-correctness', 'adversarial-reviewer', {0|1}, '{verdict}', '{severity}', {round}, '{perspective_id}');
```

Where `{scope}` is `design` (Step 3b) or `code` (Step 7), and `{perspective_id}` is one of: `security-sentinel`, `architecture-guardian`, `pragmatic-verifier`.

### Verdict YAML (1 per reviewer)

Output to `review-verdicts/{scope}-{perspective_id}.yaml` with category sub-verdicts:

```yaml
verdicts:
  security: "approve | needs_revision | blocker"
  architecture: "approve | needs_revision | blocker"
  correctness: "approve | needs_revision | blocker"
overall: "approve | needs_revision | blocker"
```

### Evidence Gate Validation

The orchestrator verifies:

1. **All reviewers submitted:** 3 distinct instances in `anvil_checks` for the round
2. **All-category coverage:** Each instance has 3 distinct `check_name` entries (security, architecture, correctness)
3. **Zero blockers:** No records with `verdict='blocker'`
4. **Majority approval:** ≥2 of 3 reviewers fully approve across all categories
