# Fetch and Isolate

The orchestrator keeps its own context clean and stays in control of the sequence. It dispatches one narrow, stateless task to a subagent that fetches, processes, and returns. Sequential, not parallel.

## When This Pattern Fits

- The orchestrator needs data that requires MCP tools or LLM interpretation to obtain
- The work should be isolated from the orchestrator's context (large payloads, sensitive operations, sandboxed execution)
- The orchestrator must drive the reasoning and decide each next step — the subagent only fetches and returns

See [design-principles.md](../design-principles.md) → Core Principles for whether to use a script, the orchestrator directly, or a subagent, and [delegation.md](../delegation.md) → Choosing a Pattern for which delegation pattern fits.

## Dispatch Structure

1. **Dispatch** — orchestrator spawns a subagent with a narrow task: what to fetch, how to reduce it to a digest, and where to write the raw response and the digest
2. **Fetch** — subagent fetches the raw response (MCP, interpretation, or sandboxed computation) and writes it to the raw file
3. **Extract** — a script reads the raw file and writes the digest to the output file; when the digest needs interpretation, the subagent writes it and a script validates it
4. **Return** — subagent returns a short status — the data stays in the files
5. **Continue** — orchestrator reads the digest and continues its own reasoning

The orchestrator drives the sequence.

## Subagent Prompt Template

This is an example, not a fixed form — expect to adapt it to your use-case (tools, steps, output shape). The conventions it follows carry over (see [delegation.md](../delegation.md)); the exact wording doesn't.

```
You have `ReadInternalWebsites` and `SkillsTool` — use them.
You MUST fetch {{resource}} using `ReadInternalWebsites` and write the raw response to {{raw_path}}.
Run {{extract_script}} to write {{fields_to_extract}} from {{raw_path}} to {{output_path}} as JSON.
Write only to the paths given above. Return a short status confirming the write (or the error) — not the data itself.
Do NOT use `web_fetch` as an alternative.
```

Subagent prompts are narrow and specific — the orchestrator has already decided what to do; the subagent only executes.

If the digest needs LLM interpretation rather than deterministic extraction, the subagent writes it and a script validates — see the Output Contract in [delegation.md](../delegation.md).

## Output

The subagent writes the digest — only the fields or summary the orchestrator needs for its next decision — to the specified file.

**Pass/fail:** The subagent summary tells the orchestrator whether the task succeeded:
- Success: "Fetched and wrote 12 fields to /path/output.json"
- Failure: "Fetch failed: 404 Not Found for {{resource}}"

The orchestrator handles failure per the step's failure policy.
