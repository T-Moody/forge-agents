# Memory: Adversarial Reviewer 3 (claude-opus-4.6 perspective)

## Key Insights from This Review

1. **Anvil's `code-review` is a platform-provided agent type, not a custom `.agent.md`.** Model routing (`agent_type: "code-review", model: "..."`) works because it's a built-in VS Code Copilot feature. Custom `.agent.md` agents dispatched via `runSubagent` have NO verified model routing support. This is the single most important unverified assumption in the design.

2. **Anvil's tight verify-fix loop (Step 4 → 5b → fix → re-verify within one agent) is structurally impossible in a multi-agent pipeline.** The design's Implementer → Verifier → Planner → Implementer round trip loses context at every boundary. This was Anvil's main advantage over Forge, and both Forge and the new design share this limitation.

3. **Anvil's Evidence Bundle (Step 5e) is a key user trust pattern — a structured proof of quality.** The new design generates scattered verification reports and review findings across multiple files/formats but never assembles them into a coherent user-facing deliverable.

4. **The `orchestrator-tool-restriction` feature (in this same workspace) explicitly prohibited `run_in_terminal` for the Forge orchestrator.** The new design reverses this for SQLite queries without acknowledging the contradiction or the original rationale.

5. **Model diversity ≠ perspective diversity.** Forge's CT cluster had 4 agents with different _focus areas_ (security, scalability, maintainability, strategy). The new adversarial reviewer dispatches the same prompt to different models. Three models asking the same question produces less diverse feedback than four prompts asking different questions.

6. **YAML as "typed schemas" provides no runtime enforcement over structured Markdown when both are validated by an LLM.** The design honestly admits this. The YAML layer's value is structural consistency for LLM output generation, not type safety.

7. **FR-4.7 (revert after 2 failed fix attempts) has been flagged in ct-strategy (v1), ct-maintainability (v2), and now this review (v3). It remains architecturally unassigned — the Verifier is read-only, the Implementer may have exited, and the Orchestrator's `run_in_terminal` access for reverts would further blur its coordinator role.**

8. **Pushback placement matters.** In Anvil, pushback fires before any work. In the pipeline, pushback fires at Step 2, after 4 researcher dispatches. This structural difference means the pipeline cannot prevent wasted compute on bad requests the way Anvil can.

9. **The v3 "always 3 reviewers" change diverges from the feature spec's CR-4 and FR-5.3, which specify risk-driven reviewer counts (1 for standard, 3 for Large/🔴).** The design acknowledges the simplification but doesn't note the spec divergence.

10. **Anvil's dropped operational features — git hygiene (Step 0b), session recall (Step 1b), auto-commit (Step 8) — are quality-of-life features that prevent common failure modes (dirty state, repeated regressions, uncommitted work). None have equivalents in the new pipeline.**
