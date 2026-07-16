# Delegation Model

Shared rules and constraints for all delegation patterns. Each pattern file in the `delegation/` subfolder builds on this foundation.

## Model

An **orchestrator** (the main agent) spawns one or more **subagents** to handle work in independent contexts. Subagents communicate results through files. The orchestrator coordinates — what happens after dispatch (wait, validate, aggregate, or ignore) depends on the specific pattern.

```
Orchestrator
    ├── dispatch → Subagent 1 → writes /path/batch-1.json
    ├── dispatch → Subagent 2 → writes /path/batch-2.json
    └── (pattern-specific: wait, validate, aggregate, or continue)
```

## Roles

### Orchestrator

- Resolves inputs and outputs for delegated work (file paths, task descriptions, expected results)
- Dispatches work with explicit task descriptions
- Handles post-dispatch behavior per the specific pattern (wait, validate, aggregate, or continue)
- Handles failure per the policy defined for the SOP (see [design-principles.md](design-principles.md) — SOP Architecture)

### Subagent

- Runs in its own context, separate from the orchestrator
- Reads inputs from files at paths the orchestrator specified
- Writes results to a file at a path the orchestrator specified
- Returns a short summary in the response — not the data itself

## Communication Channels

Subagents use two channels:

- **Files carry data.** The orchestrator writes inputs to a file and passes the absolute path in the subagent prompt. The subagent writes results to a file the orchestrator specified. The orchestrator owns output paths: it assigns each subagent a specific path and MUST make paths distinct when subagents run concurrently, or when an earlier result hasn't been consumed yet. Subagents write only to the path they're given and never choose their own — otherwise parallel subagents collide and overwrite each other's output.
- **Skills carry instructions.** Keep subagent prompts short — task description and file paths. The subagent loads its own skills via `SkillsTool` for the detailed instructions.

This split keeps context small (prompts don't carry instructions; skills do) and keeps data persistent (files persist regardless of context compaction).

## Constraints as Features

The subagent pattern deliberately does NOT support:

- **Inter-subagent messaging** — subagents do not communicate with each other. Coordination flows through the orchestrator.
- **Persistent subagents** — subagents do not persist between dispatches. State lives in files, not in subagent memory.
- **User steering mid-flight** — once dispatched, a subagent runs to completion. The user interacts with the orchestrator, not individual subagents.

These constraints keep SOPs repeatable. If your use case needs any of these, a different orchestration model (e.g., Claude Code's Agent Teams) may fit better.

## Subagent Patterns

The following sections apply specifically to subagent-based delegation — where the orchestrator spawns independent sub-sessions to handle work.

### When to Use Subagents

Two motivations. A single step can hit both.

#### Parallelization

Processing multiple independent items where sequential processing in one context would be too slow or would exhaust context. Examples: process 50 API responses, analyze 50 CRs, check 20 pipelines. Each item requires tool calls; sequential processing runs out of context before finishing.

#### Context Isolation

A single step produces verbose intermediate output (API payloads, code search results, large file contents) that would flood the orchestrator's context. Dispatching the work to a subagent keeps the verbose data in the subagent's context; only a short summary returns.

Isolation also serves reasons beyond context size: keeping a verification independent of the generator, or containing sensitive or sandboxed work.

Many cases need both — e.g., 50 CRs where each CR requires a verbose fetch.

#### Scale awareness

When authoring an SOP, ask: "What if the input had 10x or 100x items?" An SOP that works for 5 items often breaks at 50 — context fills up, compaction degrades earlier results, reasoning suffers.

If an SOP processes a list of items where each iteration requires tool calls, it SHOULD use subagents. The threshold is low: if each item generates more than a few tool calls, batch into subagents.

#### When NOT to use subagents

- Simple single-item operations where the dispatch overhead isn't worth it
- Steps that need to modify the main conversation state directly

### Subagent Design

#### Tool Naming

- **Name tools explicitly.** Subagents may not follow tool preferences unless the prompt names the exact tool and forbids alternatives: "Use `ReadInternalWebsites` to fetch X. Do NOT use `web_fetch`."
- **Lead with positive assertions.** Subagents may incorrectly report they lack access to tools they actually have. Start the prompt with: "You have `ReadInternalWebsites` and `SkillsTool` — use them." Backticks around tool names anchor the model to exact identifiers.
- **Skills must name tools too.** If a specific tool is required, the skill MUST name it explicitly. The skill is the subagent's only source of "how to do the work." If tool choices aren't in the skill, the orchestrator must repeat them in every prompt.

#### Skill Loading

- **Tell subagents to load skills explicitly.** Subagents may default to the `knowledge` tool for lookups instead of `SkillsTool`. Put "Load skill 'X' via `SkillsTool`" directly in the prompt.
- **For simple tasks, skip skill loading.** If the task can be fully described inline (URL pattern, extraction fields, output schema), a self-contained prompt is more reliable than one that depends on a skill load succeeding.

#### Instruction Isolation

Don't leak orchestrator concerns into subagents. A subagent performing data analysis doesn't need UI skills, workflow coordination state, or unrelated domain instructions — those pollute its context and degrade reasoning.

Load only the skills the subagent's task requires. Shared skills (e.g., a common data-parsing skill used by both orchestrator and subagents) are fine — reuse over duplication. The test: does this skill help the subagent do *its* task? If not, don't load it.

If a subagent needs only part of a skill, that's a signal to split the skill — or inline the relevant portion in the prompt if it's small.

#### Path Passing

**Skills load by name** — `SkillsTool` resolves skills from the installed package. No paths needed.

**Files need absolute paths in subagents.** Subagents don't inherit environment variables from the orchestrator — `$PKG_ROOT`, `$CACHE_DIR`, etc. expand to empty strings. The orchestrator MUST substitute literal absolute paths into the subagent prompt before dispatching.

```
# ✅ Works — literal path
prompt: "Write results to /home/user/.cache/my-project/batch-1.json"

# ❌ Fails — variable unexpanded in subagent
prompt: "Write results to $CACHE_DIR/batch-1.json"
```

This applies to: output files, input data files, script paths, and any file the subagent needs to read or write.

#### Output Contract

- Subagents write results to the file the orchestrator specified.
- **Structured output SHOULD be script-produced or script-validated** (valid JSON, required fields), not hand-written by the subagent — Principle 1 applies inside subagents. Reserve hand-written output for genuinely semantic results; fully deterministic output needs no subagent at all.
- Subagents return a short summary in the response — not the data itself. Large responses bloat context.
- Summary format **SHOULD** include enough information for the orchestrator to act on the result (e.g., "Done: 10 items processed, 0 failures" or "Error: 3 API timeouts, wrote partial results to /path/batch-1.json").

### Granularity

When deciding how many items each subagent processes:

- **One item per subagent** — maximum parallelism, highest dispatch overhead. Best when each item takes significant time or context (large API responses, deep analysis).
- **N items per subagent (batching)** — lower dispatch overhead, moderate parallelism. Best when items are small and uniform.
- **Width vs depth** — prefer wider fan-out (more subagents, fewer items each) when items are slow or context-heavy. Prefer batching when items are fast and uniform.

No universal threshold. Measure when in doubt.

### Validation

For patterns that wait for subagent completion, validation of output is part of the step that dispatched it — the step is not complete until results are validated. The validation method, location, and failure handling are defined in [design-principles.md](design-principles.md).

When a validation failure has an obvious destructive recovery path a subagent or the orchestrator might attempt (e.g., deleting cache files, rewriting inputs), the validation error message SHOULD explicitly call it out. Example: "Duplicate entry detected — DO NOT modify the cache file to fix this. Investigate the dispatch inputs."

## Choosing a Pattern

Match the step's shape to a pattern. Patterns differ on how many sources a step handles and whether they run in parallel or in sequence:

| Pattern | Use when | Execution |
|---|---|---|
| [fan-out-aggregate.md](delegation/fan-out-aggregate.md) | A step processes many independent items that each need tool calls | Parallel — many subagents, outputs aggregated |
| [fetch-and-isolate.md](delegation/fetch-and-isolate.md) | A step needs one large or sensitive payload but the orchestrator only needs a digest | Sequential — one subagent, digest written to a file |

You can combine patterns — e.g., fan out to subagents that each isolate a verbose fetch. More patterns may be added as new step shapes emerge; add a row here and a reference file in `delegation/`.

See [SKILL.md](../SKILL.md) for the full reference files list.
