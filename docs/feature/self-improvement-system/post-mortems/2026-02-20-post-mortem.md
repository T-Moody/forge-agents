# Post-Mortem Report

**Feature:** self-improvement-system
**Run Date:** 2026-02-20

````yaml
post_mortem_report:
  run_date: "2026-02-20"
  feature_slug: "self-improvement-system"
  summary: "37 agent dispatches, 0 errors, 0 retries, 2 revision loops (CT cluster + R-Quality), 22 artifact evaluations processed across 4 producer agents"

  recurring_issues:
    - issue: "Spec-design divergence on AC-5/FR-6.1 (tool restriction): feature.md specifies removing all tools but CT-approved design retains read tools — AC text never updated"
      affected_agents:
        ["v-feature", "ct-security", "ct-strategy", "r-knowledge"]
      frequency: 4
    - issue: "Spec-design divergence on FR-3.3 (improvement_recommendations): feature.md includes field in PostMortem schema but design.md removed it per CT-6 boundary resolution"
      affected_agents: ["documentation-writer-11", "ct-strategy", "ct-security"]
      frequency: 3
    - issue: "Design template error handling conflict: design.md §Evaluating Agent Changes template says 'skip evaluation' on failure vs evaluation-schema.md Rule 4 says 'write evaluation_error block' — propagated to 6 agent files"
      affected_agents: ["r-quality", "implementer-fix-eval-errors"]
      frequency: 2
    - issue: "Post-mortem.agent.md code fence format: wrapped in ```chatagent fence instead of bare --- frontmatter, deviating from all 18 other agent files"
      affected_agents:
        ["v-build", "r-quality", "r-knowledge", "implementer-fix-postmortem"]
      frequency: 4

  bottlenecks:
    - agent: "ct-cluster (step 3b)"
      reason: "First CT run returned High/Critical severities requiring designer revision and full CT re-run — added 5 additional dispatches (1 designer + 4 CT re-runs)"
      retry_count: 0
    - agent: "r-quality (step 7)"
      reason: "Returned NEEDS_REVISION with 2 Major findings, triggering 2 fix implementer dispatches at step 7.4"
      retry_count: 0

  most_common_missing_information:
    - item: "AC-5 text does not reflect the CT-approved design revision (retaining read tools)"
      reported_by: ["v-feature"]
      source_artifact: "feature.md"
    - item: "Evaluation step template does not specify source artifact list formatting (header style, bold vs plain) — led to 5 template variants across implementers"
      reported_by: ["r-quality"]
      source_artifact: "design.md"
    - item: "No cross-reference check between template error handling text and evaluation-schema.md Rule 4"
      reported_by: ["r-quality"]
      source_artifact: "design.md"
    - item: "Task 02 did not mention expected code fence format (bare --- vs ```chatagent wrapper)"
      reported_by: ["v-tasks"]
      source_artifact: "tasks/02-create-postmortem-agent.md"
    - item: "No explicit rationale for why designer evaluates only feature.md but not the 4 research files it also consumes"
      reported_by: ["r-testing"]
      source_artifact: "design.md"
    - item: "r-testing evaluation target listed only feature.md in task file but both design.md and feature.md were intended"
      reported_by: ["implementer-09"]
      source_artifact: "tasks/09-agent-eval-r-cluster.md"
    - item: "No guidance on what 'sufficient structural validation' means for a Markdown-only repo"
      reported_by: ["r-testing"]
      source_artifact: "feature.md"
    - item: "No content-level verification of evaluation step wording consistency across the 14 agents"
      reported_by: ["v-feature"]
      source_artifact: "verification/v-build.md"
    - item: "No line numbers for evaluation-schema.md structural sections in v-build report"
      reported_by: ["v-tests"]
      source_artifact: "verification/v-build.md"

  agent_accuracy_scores:
    - agent: "planner"
      average_usefulness: 8.67
      average_clarity: 8.78
      total_inaccuracies_reported: 0
      evaluations_count: 9
      source_artifacts_evaluated:
        [
          "tasks/01",
          "tasks/02",
          "tasks/03",
          "tasks/09",
          "tasks/10",
          "tasks/11",
          "plan.md",
        ]
    - agent: "designer"
      average_usefulness: 8.20
      average_clarity: 8.20
      total_inaccuracies_reported: 1
      evaluations_count: 5
      source_artifacts_evaluated: ["design.md"]
    - agent: "spec"
      average_usefulness: 7.40
      average_clarity: 8.40
      total_inaccuracies_reported: 2
      evaluations_count: 5
      source_artifacts_evaluated: ["feature.md"]
    - agent: "v-build"
      average_usefulness: 9.00
      average_clarity: 9.00
      total_inaccuracies_reported: 0
      evaluations_count: 3
      source_artifacts_evaluated: ["verification/v-build.md"]

  pipeline_summary:
    total_dispatches: 37
    unique_agents: 28
    total_errors: 0
    total_retries: 0
    revision_loops: 2
    revision_loop_details:
      - loop: "CT cluster (step 3b)"
        trigger: "High/Critical severities in first CT run"
        additional_dispatches: 5
        resolution: "Designer revision reduced all severities to Medium"
      - loop: "R-Quality (step 7)"
        trigger: "2 Major findings (code fence format, design-schema error handling conflict)"
        additional_dispatches: 2
        resolution: "implementer-fix-postmortem and implementer-fix-eval-errors resolved both issues"
    evaluation_files_processed: 8
    total_evaluations_extracted: 22
    human_interventions: 0
````
