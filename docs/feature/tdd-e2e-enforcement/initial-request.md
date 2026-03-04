# Initial Request — TDD & E2E Enforcement

**Feature Slug:** tdd-e2e-enforcement
**Run ID:** 2026-03-04T00:00:00Z
**Schema Version:** 1.0

---

## Raw User Prompt (Verbatim)

# Objective

Improve the agent system located at:

X:\programProjects\OrchestratorAgents\NewAgents

The goal is to make TDD and E2E testing core enforcement mechanisms of the workflow while preserving agent parallelism, determinism, performance, and context safety.

This system is used across multiple applications, so the solution must be dynamic, configurable, and framework-agnostic.

This is a massive change and it's okay to delete, update, add files and agents any way needed

Do additional online research for cutting edge multiagent systems, e2e testing with multi agents systems or agents, tdd with agents and anything system that ensures agents produce correct and tested code

---

# Core Design Principles

1. Testing is the primary proof of completion.
2. No work is considered complete without passing verification.
3. Orchestrator does not perform implementation work.
4. Agents must remain deterministic and concise.
5. Risk-based workflow lanes must be enforced.
6. Parallel agents must not interfere with each other.
7. Subagents must never exceed context limits.
8. Interactive mode must escalate ambiguity early.

---

# 1. Make TDD Mandatory

All implementers must follow these TDD rules:

## TDD RULES

### TDD Red

- Write failing tests FIRST.
- Confirm the tests FAIL.

### TDD Green

- Write the MINIMAL amount of code required to pass.
- Avoid over-engineering.
- Confirm tests PASS.

### TDD Verify

- Run:
  - get_errors
  - typecheck
  - unit tests
  - failure mode mitigations

### Failure Handling

- Transient errors → handle.
- Persistent errors → escalate.
- Test failures → fix all or escalate.
- Security issues → fix immediately or escalate.
- Vulnerabilities → fix before handoff.

### Test Writing Guidelines

- Do NOT test what the type system guarantees.
- Test behavior, not implementation details.
- Avoid brittle tests.
- Use only public interfaces.
- Do NOT expose internals or create test-only hooks.
- Never leave TODO/TBD in final code.

### Code Quality Enforcement

- YAGNI
- KISS
- DRY
- Prefer functional patterns
- Avoid over-engineering
- Maintain lint compatibility

---

# 2. E2E Testing as a First-Class Citizen

This is NOT just adding test files.

The system must support agents actually running the application and testing it like a real developer trying to break it.

## Dynamic E2E Contract

Each app must define a machine-readable E2E contract:

- How to build the app
- How to run the app
- Required environment variables
- How to detect readiness
- Base URL
- Shutdown mechanism
- Test execution command
- Parallel test support (yes/no)
- Isolation requirements

Example (note that this may not be correct for playwright, this is just and example):

```yaml
app_type: web
start_command: dotnet run
ready_check: http://localhost:5000/health
base_url: http://localhost:5000
e2e_runner: playwright
e2e_command: npx playwright test
supports_parallel: false
requires_isolated_instance: true
skill?
```

For web apps, default recommendation:
Use Playwright CLI (https://github.com/microsoft/playwright-cli)

Agents must not assume Playwright.
They must consult the E2E contract.

Setting up e2e will vary greatly from app to app. In interactive mode and setting up e2e the agents should always do we searches for the latest information about whatever stack the project is. In interactive mode the agents should work with the user to get a skill set up that can be reused over and over for e2e testing and the agent should have a way to reference the skill, probabaly in the contract

In automatic mode we may not want to set up e2e

---

# 3. Live App Testing Architecture

Implementers:

- Write unit + integration tests (TDD).
- DO NOT run live E2E tests.

Verifiers:

- Spin up isolated app instances.
- Execute E2E tests.
- Attempt adversarial workflows.
- Simulate a developer trying to break the system. This means acutally running the app and navigating or making api calls
- Log failures deterministically.

Never allow implementers and verifiers to both run live app instances unless isolated.

---

# 4. Parallel Agent Safety

Agents run in parallel. Prevent resource conflicts.

Required controls:

- Instance isolation (unique ports or containers)
- File locks on shared artifacts
- Max concurrent E2E limit
- Resource throttling
- Deterministic test seeds
- Timeouts on all external calls

If multiple verifiers attempt live testing:

- Queue them
- Or spawn isolated instances

Never allow race conditions on the same app instance.

---

# 5. Risk-Based Workflow Lanes

Low Risk:

- Config change
- Docs
- Non-runtime refactor
  → Unit tests only

Medium Risk:

- Business logic changes
  → Unit + integration

High Risk:

- UI flow
- API contract
- State transitions
- Security-sensitive
  → Full TDD + E2E verification

The orchestrator must classify changes and select workflow lane.

---

# 6. Subagent Context Governance

Subagents must:

- Operate within bounded context windows.
- Read only necessary files.
- Prefer semantic search and targeted reads.
- Never load entire repositories blindly.

Enforce:

- Max lines per read.
- Structured summarization.
- Context compaction.

---

# 7. Interactive Mode Requirements

Agents must not work blindly.

If:

- Spec unclear
- Contract missing
- Tests ambiguous
- Risk classification uncertain

They must escalate early.

If subagents cannot ask questions:

- Orchestrator must ask on their behalf.

Spec agents MUST push back on ambiguity.

---

# 8. Orchestrator Rules

The orchestrator:

- Routes tasks.
- Enforces workflow lanes.
- Aggregates results.
- Escalates ambiguity.
- Manages concurrency.

The orchestrator must NOT:

- Write implementation code.
- Bypass workflow.
- Perform ad-hoc reasoning outside defined lanes.

Keep orchestrator deterministic and minimal.

---

# 9. Copilot Best Practices Integration

Implementers should behave like real developers:

- Update Copilot instructions when patterns drift.
- Refine prompt scaffolding.
- Add guardrails when hallucinations appear.
- Ask clarifying questions in interactive mode.
- Avoid speculative abstraction.

Use:

- Small files.
- Explicit interfaces.
- Clear contracts.
- Deterministic naming.
- Strong typing.
- CI enforcement.

---

# 10. Additional Enhancements

Recommended:

A. Test Replay Logs
Store:

- Inputs
- Seeds
- Browser traces
- Screenshots
- Console logs

B. Workflow Completion Gate
Task is COMPLETE only if:

- Unit tests pass
- Verification checks pass
- Required E2E passes
- No security issues
- No lint violations

C. Agent Performance Guardrails

- Hard timeouts per stage
- Max retries
- Max E2E attempts
- Escalation thresholds

D. Deterministic Output Contracts
All agents must output:

- Status
- Evidence
- Logs (if failed)
- Structured JSON

No prose unless interactive mode.

---

# Non-Negotiables

- Testing proves completion.
- No silent failures.
- No speculative architecture.
- No infinite loops.
- No long-running blind work.
- All ambiguity escalated early.
- All security issues fixed before handoff.
