---
name: agent-sop-author
description: Create, update, and validate Agent SOPs (Standard Operating Procedures), also called Agent Scripts - markdown workflows that guide AI agents through complex, multi-step tasks with RFC 2119 constraints. Covers SOP format and design - enforcement language, validation, subagent dispatch (parallel and isolated), and failure handling. Use when authoring or editing an SOP, structuring a multi-step agent workflow, deciding how to split work between scripts/subagents/main agent, converting a workflow into a reusable template, or fixing an SOP that produces wrong or inconsistent output, skips or adds steps, uses the wrong tools, reformats output instead of following instructions, or slows down, times out, or loses data when processing many items.
version: 1.0.0
tags: [skill, agent-sop, agent-script, workflow, automation, sop, rfc2119]
---

# Agent SOP Author

## Overview

This skill is split into reference files covering SOP format, design principles, and delegation. See the Reference Files table below.

## Terminology

- **SOP** — a `.sop.md` file containing a sequence of steps that guide an agent through a procedure.
- **Workflow** — synonym for SOP.
- **Step** — a numbered instruction within an SOP (e.g., `### 1. Fetch data`). Each step achieves one outcome.
- **Orchestrator** — the agent executing the SOP. It follows steps, dispatches subagents, and handles failures.
- **Subagent** — a separate agent spawned by the orchestrator to handle a unit of work in isolation.
- **Digest** — a compact summary or extracted subset of a payload that a subagent returns in place of the full data.

## Usage

Use this skill when:
- Creating a new SOP or workflow
- Updating an existing SOP
- Structuring multi-step agent workflows with RFC 2119 constraints
- An SOP produces wrong or inconsistent output, skips steps, or uses the wrong tools
- An SOP slows down, times out, or loses data when processing many items
- Agents add extra steps, skip steps, or reformat output instead of following the SOP
- Deciding how to split work between scripts, subagents, and the main agent
- Converting workflows into reusable templates

## Core Concepts

**SOPs are step sequences the agent follows.** Unlike skills (which provide guidance and tools for a domain), SOPs define ordered steps the agent executes to completion.

**RFC 2119 constraints control agent behavior.** Use MUST for absolute requirements, SHOULD for strong recommendations, MAY for optional behavior. Always provide context for negative constraints ("You MUST NOT X because Y").

**Delegation runs a step's work in separate subagent contexts.** Use it when a step's work would overwhelm the orchestrator's context (many items, large payloads) or should run in isolation from it (sandboxed, sensitive, or independently verified work). See delegation.md for the model and the available patterns.

## Reference Files

| File | Content |
|---|---|
| [sop-format.md](references/sop-format.md) | File structure, naming, parameters, steps, validation |
| [design-principles.md](references/design-principles.md) | Core principles, SOP architecture, validation, failure handling, enforcement language |
| [delegation.md](references/delegation.md) | Delegation model, roles, communication channels, constraints, subagent design rules |
| [delegation/fan-out-aggregate.md](references/delegation/fan-out-aggregate.md) | Pattern: N items → parallel subagents → aggregate results |
| [delegation/fetch-and-isolate.md](references/delegation/fetch-and-isolate.md) | Pattern: verbose fetch isolated from orchestrator → digest |

### sop-format.md
Read when creating or updating a `.sop.md` file. Covers file naming, required sections, parameter format, step structure, RFC 2119 keywords, negative constraints, and validation.

### design-principles.md
Read when deciding how an SOP should work — splitting work between scripts, subagents, and the main agent; enforcing tool choices; validating step outcomes; handling failures; or scaling to many items.

### delegation.md
Read when a step processes many items, produces verbose intermediate output, or would exhaust the main context. Covers the shared delegation model, orchestrator/subagent roles, communication channels, constraints, and subagent design rules.

### delegation/fan-out-aggregate.md
Read `delegation.md` first for shared rules. Then read this when a step processes a list of independent items that each require tool calls. Covers batch splitting, subagent prompt templates, partial-failure aggregation, and concurrency limits.

### delegation/fetch-and-isolate.md
Read `delegation.md` first for shared rules. Then read this when a step needs data from a source that returns large payloads but the orchestrator only needs a summary or extracted fields.

## Quick Reference

| Topic | Rule |
|---|---|
| File extension | `.sop.md` |
| File naming | kebab-case (e.g., `code-assist.sop.md`) |
| Required sections | Overview, Parameters, Steps (with Constraints) |
| Constraint keywords | MUST, SHOULD, MAY (RFC 2119) |
| Negative constraints | Always include "because Y" context |
| Deterministic work | Scripts, not LLMs |
| Data passing | Structured data passes through files, not context |
| Subagents | Use when work would degrade the main context |
| Steps | One outcome per step (including validation) |
| Validation | Each step validates its own outcome |
| Language | "You MUST", not "you should" or "consider" |
| Tool naming | Name tools explicitly when alternatives exist |
| Script output | Mark user-facing script output as self-displaying |
| Validate after changes | Run `validate-sop.sh` after EVERY change |
