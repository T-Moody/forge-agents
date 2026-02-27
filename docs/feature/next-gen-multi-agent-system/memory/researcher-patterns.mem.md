# Memory: researcher-patterns

## Status

DONE: Comprehensive pattern analysis of Forge (23-agent pipeline) and Anvil (single evidence-first agent) systems covering completion contracts, dispatch patterns, memory lifecycle, error handling, verification cascades, SQL evidence tracking, adversarial review, and failure prevention principles.

## Key Findings

- Forge uses typed 3-state completion contracts (DONE/NEEDS_REVISION/ERROR) with deterministic routing, but most agents only use DONE/ERROR; NEEDS_REVISION is primarily a cluster-level concept evaluated by the orchestrator
- Anvil's SQL verification ledger with INSERT-before-report rule and COUNT-based gates provides machine-verifiable evidence that Forge's Markdown-prose verification cannot match
- Both systems share self-verification-before-return and non-blocking graceful degradation patterns, but use fundamentally different communication media (Markdown memory files vs SQL records)
- Anti-patterns identified: excessive memory merge overhead (~20+ merges per 32-dispatch run), inconsistent severity taxonomies across Forge clusters (CT/R/V each use different vocabularies), and prose where data would be more reliable
- Anvil's risk classification (🟢/🟡/🔴 per-file) driving verification depth (S/M/L) and adversarial review intensity is a pattern not present in Forge that should inform the new system

## Highest Severity

N/A

## Decisions Made

(none — research only)

## Artifact Index

- research/patterns.md — §1.1 Completion Contracts (3-state system usage across all agents), §1.2 Cluster Dispatch Patterns (A/B/C definitions and usage), §1.3 Memory Patterns (full lifecycle), §1.4 Error Handling (retry budgets, transient vs deterministic), §1.5 NEEDS_REVISION Routing (routing table with max loops), §1.6 Evaluation Schema (YAML format, collision avoidance), §1.7 Self-Verification (per-agent checks before return), §1.8 Tool Restrictions (per-role access profiles), §1.9 Telemetry (context-only → post-mortem write), §2.1–2.14 Anvil patterns (task sizing, risk classification, verification cascade, SQL ledger, adversarial review, evidence gating, pushback, baseline capture, confidence scoring, git hygiene, session recall, boost, learn), §3 Failure Prevention mapping, §4 Cross-System Comparison tables, §5 Anti-Patterns, §6 Reusable Patterns for New System
