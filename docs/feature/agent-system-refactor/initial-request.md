# Initial Request: Agent System Refactor (Run 3)

**Feature Slug:** agent-system-refactor
**Run ID:** 2026-03-09T18:00:00Z
**APPROVAL_MODE:** interactive
**Design Web Research:** enabled

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

---

## Run 2 Requirements (2026-03-09 — Verbatim User Input)

We are missing some things that I think we should take a look at:

1. An e2e contract or guidance to help dev set up e2e skill so it can be used by the agents. A lot of developers using these agents don't use e2e testing so the hope was for the agents to autodetect skills if already setup or if not we had various defaults to help them set up. For example an app with UI would default to the playwright cli skill, and apis would use client secret, client id, tenant id and so on to get token since a lot of apps use azure so we could get the secrets from key vault if auth is needed. Also we presented multiple choice to ask the user if they wanted to set this up instead of silently failing
2. You need to use the subagent tool correctly in agents. You must look it up in vs code docs. You have to say use subagent tool and call agents by their exact names
3. I don't see any pushbacks or clarifying questions. We used to allow spec to ask clarifying questions in interactive. Our architect should do this now
4. Our planner should ask multiple choice question to accept or refine the plan in interactive
5. Subagents should not be doing git commands that can conflict with other subagents. For example git add . would add all files even if other agents don't want that
6. I don't think we should autocommit code at the end. Devs might want to look at the changed code first
7. I think it would be valuable to have an optional step that updates documentation. Maybe a multiple choice at the end to use findings from the knowledge agent to update documentation. For example look at all copilot instructions files and make sure they aren't redundant, out of date or getting too large. They should be targeted to exactly where needed
8. Should design do web research?

Additional context: Do not assume you know anything and assume subagents knowledge is outdated. Require all subagents to do research in this run. Do web research on skills, e2e practices for agents, github copilot and claude skills, copilot instructions, agent formatting, multi-agent systems, github copilot in vs code tools, vscode experimental features. Review NewAgents and V2. Look at existing research in docs/feature/agent-system-refactor and dig deeper.

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

---

## Run 3 Requirements (2026-03-09T18:00:00Z — Verbatim User Input)

Let's continue work on docs/feature/agent-system-refactor where we took NewAgents and applied best practices from the web to make a new agent pack v2.

We are missing some things that I think we should take a look at:

1. An e2e contract or guidance to help dev set up e2e skill so it can be used by the agents. A lot of developers using these agents don't use e2e testing so the hope was for the agents to autodetect skills if already setup or if not we had various defaults to help them set up. For example an app with UI would default to the playwright cli skill, and apis would use client secret, client id, tenant id and so on to get token since a lot of apps use azure so we could get the secrets from key vault if auth is needed. Also we presented multiple choice to ask the user if they wanted to set this up instead of silently failing
2. You need to use the subagent tool correctly in agents. You must look it up in vs code docs. You have to say use subagent tool and call agents by their exact names
3. I don't see any pushbacks or clarifying questions. We used to allow spec to ask clarifying questions in interactive. Our architect should do this now
4. Our planner should ask multiple choice question to accept or refine the plan in interactive
5. Subagents should not be doing git commands that can conflict with other subagents. For example git add . would add all files even if other agents don't want that
6. I don't think we should autocommit code at the end. Devs might want to look at the changed code first
7. I think it would be valuable to have an optional step that updates documentation. Maybe a multiple choice at the end to use findings from the knowledge agent to update documentation. For example look at all copilot instructions files and make sure they aren't redundant, out of date or getting too large. They should be targeted to exactly where needed
8. Should design do web research?
9. The e2e tester should not just run the e2e test suite. It should also manually test the app. For example I want it to test things as if it were a real dev trying to ensure the feature/bug is fixed and also trying to break it. Here is an example of a qa agent that I have in another app. I don't want everything from it as this agent has a lot, but you can see how it's more like a real dev and not just running existing tests: X:\programProjects\TourneyPal\.github\agents\tourneypal-qa.agent.md
10. The implementer agent should also follow tdd code best practices. My old implementer had these best practices:
    - YAGNI (You Aren't Gonna Need It)
    - KISS (Keep It Simple)
    - DRY (Don't Repeat Yourself)
    - Prefer Functional Programming patterns
    - Avoid over-engineering
    - Maintain lint compatibility
    - Write the MINIMAL amount of code required to pass the tests
    - Avoid over-engineering

I don't want to accidentally over complicate these agents, but I want them to be powerful. Review NewAgents and V2. Also do web research on skills, e2e practices for agents, github copilot and claude skills, copilot instructions, agent formatting, multi-agent systems, github copilot in vs code tools, vscode experimental features, look at our existing research that we did in docs/feature/agent-system-refactor to see what things we researched and maybe dig deeper into that web research. I am trying to use established standards and best practices to build the most complete, powerful, deterministic multiagent system.

Do not assume you know anything and assume subagents knowledge is outdated. Require all subagents to do research in this run.

### Reference Material

QA Agent Example: X:\programProjects\TourneyPal\.github\agents\tourneypal-qa.agent.md

- 4 parallel subagent model for QA exploration
- 7 testing phases: Happy Path, State Mutation, Edge Cases, Navigation Chaos, Mobile/Tablet, UI Validation, State Replay
- Playwright CLI skill integration
- Autonomous bug discovery with backlog generation
- Context rotation for long-running sessions
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
