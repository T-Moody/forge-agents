# Initial Request: Agent System Refactor

**Feature Slug:** agent-system-refactor
**Run ID:** 2026-03-08T12:00:00Z
**APPROVAL_MODE:** (pending user selection)

---

## Raw User Prompt (Verbatim)

You are assisting in a **major architectural refactor** of an autonomous multi-agent software development system in this repository.

This system is built around an **Orchestrator** that dispatches agents defined in `.github/agents` with prompts in `.github/prompts`.

You are allowed to perform a **large refactor or redesign** if needed. Do not preserve the existing architecture if it prevents a better design.

The goal is to create a **clean, modern, deterministic multi-agent development pipeline**.

Do not assume what I have currently is the best. AI tends to be too optimistic.

Here are the agents X:\programProjects\OrchestratorAgents\NewAgents

---

REFACTOR CONTEXT (IMPORTANT)

This run is specifically to **refactor and improve the multi-agent orchestration system itself**.

During THIS run, you should perform **web research about modern multi-agent orchestration architectures**, autonomous coding agents, and best practices used by cutting-edge systems.

Examples to research:

- autonomous coding agent pipelines
- multi-agent orchestration architectures
- deterministic agent workflows
- DAG-based agent scheduling
- autonomous software engineering systems
- best practices for multi-agent design

This orchestration architecture research is **only required for this refactor**. You should follow leads and do deep research on this

Look up how to do good research and use that to help you be a good researcher. Look for patterns and evidence of orchestrators or multiagent systems producing full applications

Future normal runs should not research orchestration architecture.

---

NORMAL SOFTWARE DEVELOPMENT RESEARCH (PERMANENT FEATURE)

For future normal runs of the system, agents should support **web research for software development tasks**.

Agents should assume their internal knowledge may be outdated and should research when necessary.

Examples:

- researching new framework versions (e.g., .NET 10 features)
- understanding library APIs
- checking current best practices
- validating integration patterns

The orchestrator should ask the developer at startup:

1. Interactive mode? (yes/no)
2. Allow web research? (yes/no)

If web research is enabled, development agents should research the tech stack as needed.

This research applies to:

- Research agents
- Spec agents
- Design agents
- Implementation agents (when necessary)
- Knowledge agents

---

PIPELINE STRUCTURE

Here is my first idea of pipeline, but if you find something better than you have full freedom to do it. IN fact you are allowed to create a new folder and start from scratch:

0. Setup
1. Research
2. Specification
3. Design
4. Design Review
5. Planning
6. Implementation Waves
7. Test Runner
8. Verification
9. Knowledge Extraction
10. Instruction Optimization
11. Commit

---

TEMPORARY DESIGN REVIEW EXPANSION (THIS REFACTOR ONLY)

During this **architecture refactor run**, expand the Design Review phase to do 2 extra design review passes

These additional review passes are **only intended for this refactor**.

Future normal development runs should revert to a **single standard Design Review phase**.

---

IMPLEMENTER AGENTS

Implementers should perform strict TDD.

Responsibilities:

1. Write failing unit tests
2. Implement minimal code
3. Run unit tests
4. Repeat until tests pass
5. Produce implementation report

Rules:

- Implementers may run unit tests
- Implementers must NOT run E2E tests
- Implementers must NOT run integration environments
- Implementers must NOT perform verification logic

Implementers run in parallel within a wave.

Only targeted **unit tests** should run during implementer waves.

---

TEST RUNNER AGENT

Introduce a **Test Runner agent** responsible for executing the running system.

Responsibilities:

- start the application
- run integration tests
- run E2E tests
- run API contract tests
- perform live system testing
- perform exploratory QA checks
- stop the application

The Test Runner should support skill-based execution such as:

- playwright-ui
- http-api
- cli
- integration-tests
- live-qa

The Test Runner should also perform **live validation**, not just automated tests.

For example:

- verify endpoints respond correctly
- simulate user interactions
- validate UI flows
- confirm application behavior

Use the existing QA agent concept as inspiration.

Only **one Test Runner should run at a time**.

---

VERIFIER AGENT

Verifier agents should only evaluate evidence.

Verifier responsibilities:

- read implementation reports
- read test runner reports
- validate completion contracts
- determine pass/fail

Verifier agents must never execute code or run tests.

---

PLANNER IMPROVEMENT

Planner must declare file ownership for each task to ensure safe parallel implementation.

Example:

tasks:

- id: add-health-endpoint
  files:
  - Controllers/HealthController.cs

- id: add-logging
  files:
  - Services/LoggerService.cs

The orchestrator must ensure tasks modifying the same file are never executed in the same wave.

---

DAG EXECUTION MODEL

The orchestrator should transition toward a **DAG-based execution model**.

Tasks declare dependencies and the orchestrator determines safe parallel waves automatically.

Goals:

- deterministic execution
- reproducible runs
- clear dependency tracking

---

SQL

We should consider removing sqlite if there are better ways like DAG or whatever else. This would be nice because then devs wouldn't have to have sqlite3 installed locally, but if it's useful and other agent systems use it then that's fine

---

KNOWLEDGE AGENT

The Knowledge agent performs reflection after verification.

Responsibilities:

- generate postmortem analysis
- analyze failures and retries
- extract patterns
- document lessons learned

---

INSTRUCTION OPTIMIZATION AGENT

Introduce a dedicated agent responsible for maintaining `.github/instructions`.

This agent should:

- scan all instruction files
- detect redundant rules
- remove outdated instructions
- merge overlapping rules
- condense instructions
- keep instructions small and focused

The agent should follow best practices for AI instruction design:

- concise rules
- minimal duplication
- clear scope
- high signal-to-noise ratio

Instruction updates should include:

- adding new rules
- removing obsolete rules
- consolidating duplicates
- improving clarity

Instruction optimization may run **in parallel with the Knowledge agent**.

---

CONSISTENCY REQUIREMENTS

All agents should follow a consistent structure:

- clear responsibilities
- deterministic outputs
- structured contracts
- predictable prompts

Prompts should follow a shared format so the orchestrator can manage them easily.

---

GENERAL PRINCIPLES

When redesigning the system, prioritize:

- deterministic behavior
- minimal agent overlap
- clear responsibilities
- reproducibility
- maintainability
- simplicity where possible

Avoid agents performing multiple unrelated roles.

---

REFACTOR AUTHORITY

You are allowed to:

- redesign agents
- redesign prompts
- simplify orchestrator logic
- introduce new agents
- remove unnecessary components
- restructure workflows

The goal is a **clean, reliable, modern multi-agent development system**.

Analyze the repository and propose the necessary architectural changes to implement this design.

CRITICAL RULE: USE ESTABLISHED PRACTICES ONLY

Do NOT invent new architectural patterns.

All architectural decisions must be based on **established, widely used practices** in modern autonomous coding systems and multi-agent orchestration.

Before proposing architecture, you must perform research on:

- modern multi-agent orchestration patterns
- DAG-based task scheduling
- deterministic agent pipelines
- autonomous coding agent systems
- parallel task orchestration
- software engineering agent workflows

Architecture choices must be justified using **commonly accepted industry patterns**.

Do not create speculative or experimental architectures unless they are already documented or widely discussed in modern agent system design.

If uncertain about a design decision, research first rather than guessing.

---

CRITICAL RULE: DO NOT INVENT FILE FORMATS

This repository uses **GitHub Copilot Agents** inside:

.github/agents
.github/prompts
.github/instructions

You must **research and follow the exact GitHub Copilot agent format** before modifying or creating agents.

Do NOT guess the structure.

Before creating or modifying agents, confirm:

- correct agent file structure
- required fields
- correct YAML format
- how prompts are referenced
- how instructions are referenced

All agent definitions must follow **the official GitHub Copilot agent format used in Visual Studio Code**.

---

ENVIRONMENT CONTEXT

This system runs inside:

Visual Studio Code
GitHub Copilot
Local development environment on a developer machine

Agents are executed locally by the orchestrator.

Do not assume:

- cloud orchestration
- distributed clusters
- remote agent runtimes

Design the system for **local execution on a developer workstation**.

---

RESEARCH REQUIREMENT BEFORE DESIGN

Before proposing architectural changes, research:

- modern multi-agent orchestration systems
- best practices for autonomous software engineering agents
- DAG orchestration for task scheduling
- parallel agent pipelines
- deterministic execution pipelines

Use the research findings to guide the redesign.

All architectural changes should be traceable to **established practices** rather than invented solutions.

---

RESEARCH-GUIDED IMPLEMENTATION

During this refactor:

1. Research existing agent orchestration architectures.
2. Identify proven design patterns.
3. Apply those patterns to this repository.

Avoid inventing new patterns unless absolutely necessary.

The goal is to build a system that follows **modern best practices used in real autonomous coding systems**.
