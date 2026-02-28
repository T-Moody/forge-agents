# Context7 MCP Integration (Conditional)

> Tier 2 shared reference. Referenced by implementer and researcher agent definitions.

## §1 Availability Check

Before using Context7, attempt to call `context7-resolve-library-id` with a known library name.
If the tool is not available (MCP server not configured), skip **all** Context7 usage and proceed normally.

Log: `"Context7 unavailable — proceeding without documentation lookup."`

Do **not** treat unavailability as an error. Do not retry. Continue your workflow without Context7.

## §2 Two-Step Usage Pattern

1. **`context7-resolve-library-id`** — resolve the library/framework name to a Context7 library ID
2. **`context7-query-docs`** — query documentation using the resolved ID

Always resolve the ID first. Never pass a raw library name to `context7-query-docs`.

## §3 When to Use

- **Before guessing** at library/framework API usage (imports, method signatures, configuration)
- When implementing against an unfamiliar or recently updated API
- When research requires current library documentation

Use Context7 docs to improve prompt quality and reduce hallucinated API calls.

## §4 Graceful Degradation

| Condition                                        | Action                                              |
| ------------------------------------------------ | --------------------------------------------------- |
| MCP server not configured                        | Log "Context7 unavailable"; skip all Context7 calls |
| `context7-resolve-library-id` returns no results | Proceed without docs; do not error                  |
| `context7-query-docs` returns empty              | Use best-effort knowledge; note gap                 |

Never set agent status to ERROR due to Context7 unavailability.

## §5 Applicable Agents

| Agent           | Usage                                                     | Priority  |
| --------------- | --------------------------------------------------------- | --------- |
| **Implementer** | Use during Step 5 when writing code against external APIs | Primary   |
| **Researcher**  | Use during Step 1 for library-specific research topics    | Secondary |
| All others      | Skip Context7 entirely                                    | N/A       |
