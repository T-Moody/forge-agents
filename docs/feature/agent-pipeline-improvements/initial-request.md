# Initial Request: Agent Pipeline Improvements

## Raw User Prompt (Verbatim)

I did a major refractoring of my orchestration agents X:\programProjects\OrchestratorAgents\NewAgents. I made tdd and e2e a central point of the flow. I am running into some issues 1. The e2e tests only setup if it is a high risk item. This worries me becuase I would like even small features and bug fixes to be confirmed working 2. The default to auto mode language is confusing the agents becuase sometimes they wont use my selected mode, they just default to auto 3. There isn't enough web fetching in the agents. I would like agents to act like real developers. They shouldn't just assume they know everything. For example the designer might do web searches to get updated info on the stack and other agents might do that too. We would only want that in interactive mode though because it requires the user to hit approve 4. The implementers and validatation agents constantly run commmands that I have to approve in github copilot. This is mostly sending build and test info to files from terminal. This clutters the repo and causes me to hit allow constantly 5. The spec agents pushes back with the number of issues found but isn't specific enough. It should have a multple multiple choice sections using the ask tool in vs code so I can answer each concern in interactive mode 6. The e2e testing doesn't have to be required, but it is very important for verifying things actually work, this means looking up github copilot skills on the web and understanding how they work, then actually building as skill so the copilot can use it to interact with the app. 7. Our sqlite database frequently becomes out of date with all the changes that I'm making. I do not want migration plans. When starting a new agent pipeline, we should check if the current sqlite db is out of date and if so rename it to archive it. 8. We need a prompt to trigger a shorter version of the orchestrator. This would start at planning. That way we don't always need to do the full flow. This prompt should tell the planner to use the ask question tool to ask the user clarifying questions. We should keep the normal prompt as well. I don't want to remove my current steps. I just want a way to do a shorter flow. Evaluate if this would be better as a prompt or if the orchestrator should dynamically change based on scope of task

You are allowed to refactor anything in X:\programProjects\OrchestratorAgents\NewAgents. Everything is in scope. I want an well designed multiagent system for github copilot. Don't assume that I am right either. If you have any good ideas feel free to help. Do online research too to get the latest information on well designed multiagent systems

## APPROVAL_MODE

autonomous

## Issues Summary

1. **E2E only for high-risk** — E2E testing should be available for all risk levels, not gated behind 🔴 risk
2. **Approval mode confusion** — "Default to autonomous" language causes agents to ignore user's selected mode
3. **No web fetching** — Agents should research online like real developers (interactive mode only, requires user approval)
4. **Terminal output to files** — Implementers/verifiers redirect terminal output to files, cluttering repo and requiring constant approval
5. **Spec pushback not specific** — Should use multiple-choice sections per concern via ask_questions (interactive mode)
6. **E2E via Copilot Skills** — Research how GitHub Copilot skills work, build skills for app interaction
7. **SQLite DB staleness** — Archive outdated DB on new pipeline run instead of migration plans
8. **Shorter pipeline prompt** — Create a fast-track prompt starting at planning, with planner asking clarifying questions

## Scope

All files in `NewAgents/.github/` are in scope for modifications. User explicitly grants permission to refactor anything. User also requests online research for latest multi-agent system best practices.
