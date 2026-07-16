# Fan-Out Aggregate

Dispatch N items to parallel subagents, then aggregate results.

## When This Pattern Fits

- A step processes a list of independent items
- Each item requires tool calls (API fetches, code search, file reads)
- Items can be processed in any order
- Results need to be combined into a single output

Examples: analyze 50 API responses, check 20 service endpoints, process a batch of documents.

## Dispatch Structure

1. **Prepare inputs** — determine what each subagent needs (items to process, file paths, skill names). For large batches, a script splits items into batch files.
2. **Dispatch** — orchestrator spawns one subagent per batch with a prompt containing the task description and any references (files, skills, inline data)
3. **Collect** — orchestrator waits for all subagents to complete
4. **Aggregate** — script merges output files into a single result

```
Items (N)
    → prepare: split into batches (inline or files)
    → dispatch: subagent per batch (prompt + references)
    → collect: wait for completion
    → aggregate: merge results → final-output.json
```

## Subagent Prompt Template

This is an example, not a fixed form — expect to adapt it to your use-case (tools, steps, output shape). The conventions it follows carry over (see [delegation.md](../delegation.md)); the exact wording doesn't.

Each subagent receives:
- Input file path (what to process)
- Output file path (where to write results)
- Task description (what to do with each item)
- Tool and skill instructions (which tools to use, which skills to load)

```
You have `ReadInternalWebsites` and `SkillsTool` — use them.
Load skill 'X' via `SkillsTool` for detailed instructions.
You MUST process the items in {{input_path}}.
For each item, {{per_item_instructions}}.
Write results to {{output_path}} as a JSON array, then run {{validate_script}} to confirm it is valid JSON with the required fields.
Do NOT use `web_fetch` or `grep` as alternatives.
```

## Aggregation

After all subagents complete, a script merges the output files.

**Partial failure handling:** Some subagents may fail while others succeed. The aggregation script SHOULD:
- Merge all successful outputs
- Report which batches failed (batch number, error summary)
- Write the merged result even if some batches failed (partial results are often useful)

The orchestrator then decides per the step's failure policy whether partial results are acceptable or whether failed batches need retry.

## Batch Sizing

- **Assign explicit batch numbers** — pass each subagent a unique N in the prompt. Do NOT ask subagents to find the next available number — parallel subagents will collide and overwrite each other's output.
- **Concurrency limits** — some APIs or services rate-limit. Specify max concurrent subagents in the SOP step (e.g., "max 4 concurrent").
- **Uniform batches** — split items evenly. Uneven batches cause one subagent to run much longer than others, delaying aggregation.
