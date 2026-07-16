# SOP Design Principles

## Core Principles

1. **Use scripts for deterministic work.** Counting, formatting, classification, validation, report generation — anything with a correct answer belongs in a script, not LLM output. LLMs hallucinate counts, invent categories, and change formatting between runs.
2. **Pass data through files, not conversation context.** When steps produce structured data for subsequent steps, write to files. The step defines the output file path, passes it to the script as an argument, and the next step receives the file path (not the data). Conversation context fills up, gets compacted, and degrades reasoning. Files are persistent, validatable, and don't consume tokens. Other storage (databases, caches) works when the use case requires it — files are the default.
3. **Dispatch to subagents when work would degrade the main context, or must run in isolation from it.** Use subagents when processing many items in a loop, handling large payloads (API responses, file contents), or performing complex reasoning that needs its own context window — and when work must stay isolated from the orchestrator (sandboxed, sensitive, or independently verified). Subagents keep the main agent's context clean for orchestration.
4. **Each step achieves one outcome.** A step executes, validates its result, and handles failure — all as part of achieving that outcome. Separate steps are for separate outcomes, not for separating execution from validation.
5. **Scripts don't share the agent's MCP access.** A script the agent runs executes in a plain shell and can't call the MCP tools wired into the agent's session. If a script needs MCP-sourced data, either call the underlying service or API directly from the script, or have a subagent (which has MCP access) fetch it via MCP and write it to a file for the script to process. The agent can also call MCP directly for small responses.
6. **Agents pass file references, not file contents.** No agent (orchestrator or subagent) should read a file into context unless it needs the data for LLM reasoning. If a script can process the file, pass the path and let the script read it (see Principles 1 and 2).

## SOP Architecture

- The agent is the orchestrator — it follows steps, dispatches subagents, and handles failures.
- Each step runs the flow: execute → validate → next step. Validation produces a pass or fail — a pass advances; a fail, or an execution error, triggers failure handling.

**Failure handling.** A failing step takes one of:
- **Retry** — re-run the step (for transient failures like API timeouts). When retrying via subagent, pass the validation error so the subagent knows what to fix.
- **Continue** — log the failure and proceed (for non-critical steps where partial results are acceptable)
- **Branch** — take an alternate path (for expected failure cases)
- **Hard stop** — halt and surface the error (for data corruption or unexpected state)

Each step SHOULD specify which option applies.

## Validation

Validation has two independent choices: how to validate, and where to validate.

### How to validate

**Deterministic checks → script validation:**
- Check: structure (valid JSON, required fields) and computable content — values within allowed ranges, no duplicates, correct ordering, recomputed totals/scores match, rankings valid and complete
- Best for: file/JSON/cache output, and computed results like classifications, scores, and rankings

**Judgment checks → AI validation:**
- The agent performs the validation directly
- Best for: natural-language quality, semantic completeness or coherence — correctness no rule captures
- A script can't make judgment calls — use AI where the check needs interpretation

**Deterministic + judgment → script + AI validation:**
- Script for the deterministic checks (structure + computable content)
- AI for the judgment checks (does the content make sense, is it complete)
- Best for: output that's both computable and meaningful — a report whose numbers must be correct and whose narrative must read coherently

### Where to validate

Run each check where it's best positioned rather than re-checking everything in the orchestrator.

**Validate at the layer with the right context:**
- Producer scripts validate their own output
- Aggregator scripts validate cross-output consistency (no conflicts, referential integrity)

**Verify in an isolated subagent when the check needs independence:**
- Dispatch a separate subagent to verify output when the check needs its own context, skills, or adversarial separation from the generator
- Best for: security review, compliance checks, fact-checking generated output against a source

**Compose verification with any delegation pattern:**
- After fan-out (check each batch)
- After fetch-and-isolate (check the extracted data)
- Inline (no delegation at all)

## Enforcement Language

Agents treat SOPs as suggestions unless the language is imperative.

- **Use "You MUST" on every step.** Not "you should" or "run this command" — "You MUST run this command."
- **Mark mandatory output steps.** If a step produces user-visible output: `⚠️ MANDATORY OUTPUT STEP`. Add: "The script output IS the report. Do NOT reformat, re-render, or add commentary."
- **Prevent agent-generated intermediate reports.** Agents summarize data between steps, building their own tables and narratives that bloat context and may contain hallucinated data. Brief status updates are fine ("Step 2 complete, 15 items collected"). Agent-generated tables and summaries are not.
- **Lock down menus.** "Show EXACTLY this menu — do not add, remove, or modify options."
- **Name tools explicitly.** If the agent has access to similar tools, specify exactly which one and forbid alternatives. Example: "You MUST use `@builder-mcp` `InternalCodeSearch`. Do NOT use `grep` or `WorkspaceSearch`." Without this, agents pick whichever tool seems easiest.
- **Forbid shortcuts explicitly.** If there's an obvious shortcut that gives wrong results, call it out. Example: "Do NOT use `GetPipelineHealth` to check deployment status — health shows if the pipeline is working, not whether a specific commit deployed. You MUST use `GetPipelineDetails`." Without the prohibition, agents take the shortcut.
- **Mark script output as self-displaying.** Agents see script stdout and echo it back. For scripts whose stdout IS the report: "The script output IS the report — the user already sees it. Do NOT repeat, echo, reformat, or re-render it." This must appear on EVERY path that calls such a script — branching paths are where it gets missed.
