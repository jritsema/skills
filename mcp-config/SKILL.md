---
name: mcp-config
description: Configure MCP servers for an AI tool with the mcp-cli tool by asking the user which servers they want and applying the selection. Load this skill when the user says something like "update my MCP configuration", "set up my MCP servers", "enable/switch/clear my MCP servers", or otherwise wants to change which MCP servers are configured for a tool like Kiro, Cursor, or Claude Desktop.
displayName: MCP Config
keywords: ["mcp", "config", "mcp-cli", "servers", "profiles", "kiro", "cursor", "claude-desktop"]
author: John Ritsema
license: MIT-0
metadata:
  author: John Ritsema
  version: "0.1"
---

# Configure MCP Servers

This skill teaches an agent how to configure MCP servers for an AI tool using the
[mcp-cli](https://github.com/jritsema/mcp-cli) CLI. The flow is a short conversation:

1. Show the user the MCP servers available in their registry.
2. Confirm the target tool (default to the configured one; the user can override).
3. Ask which servers (or profile) they want.
4. Apply the selection with `mcp-cli set` and verify.

`mcp-cli` reads a `mcp-compose.yml` registry (a Docker-Compose–style file where each
service is an MCP server) and writes a tool-specific `mcp.json` config from it.

## Prerequisites

- **mcp-cli** must be installed and on `PATH`. Install the Go binary from the
  [releases page](https://github.com/jritsema/mcp-cli/releases). Verify with:
  ```bash
  mcp-cli --help
  ```
- A registry file must exist. `mcp-cli` looks for `./mcp-compose.yml` in the current
  directory first, then `$HOME/.config/mcp/mcp-compose.yml`. If neither exists, tell the
  user — there is nothing to configure from.

## Step 1 — Discover available servers

List every server with its profiles and description so you can present real options to
the user:

```bash
mcp-cli ls -ad
```

- `-a` lists all servers, `-d` shows descriptions.
- Output columns are `NAME`, `PROFILES`, `DESCRIPTION`.

Do NOT use the `-c` (or `-l`) flags when showing this to the user — those expand
environment variables inline and can leak API keys and secrets into the transcript.

To see what is currently configured for a tool:

```bash
mcp-cli ls -as -t kiro   # status for one tool (kiro, cursor, claude-desktop)
mcp-cli ls -as            # status across all tools
```

The `-a` flag is required here — `mcp-cli ls -s` on its own only considers default-profile
servers and will report "No servers found" when every server belongs to a profile.

## Step 2 — Determine the target tool

`mcp-cli` has a configured **default tool** — the config file it writes to when you don't
pass `-t` or `-c`. Detect it and tell the user, then default to it unless they ask
otherwise. There is no `config get` subcommand; read the default from
`~/.config/mcp/config.json`, whose `tool` key holds the destination path:

```bash
cat ~/.config/mcp/config.json 2>/dev/null
# e.g. {"tool": "/Users/you/.kiro/settings/mcp.json"}
```

Map that path back to a tool for the user (e.g. `.kiro/settings/mcp.json` → Kiro) and say
something like: "Your default tool is Kiro (`~/.kiro/settings/mcp.json`); I'll configure
that unless you'd like a different one." When you use the default, omit `-t` from the
commands below and `mcp-cli` writes to that path automatically.

If `~/.config/mcp/config.json` is missing or has no `tool` key, there is no default —
ask the user which tool to configure. Supported tool shortcuts and where they write:

- `kiro` — Kiro → `~/.kiro/settings/mcp.json`
- `cursor` — Cursor → `~/.cursor/mcp.json`
- `claude-desktop` — Claude Desktop → `~/Library/Application Support/Claude/claude_desktop_config.json`

The user can always target a different tool for a single command by adding `-t <shortcut>`,
or change their default with `mcp-cli config set tool <path>`.

## Step 3 — Ask which servers

Present the available servers (name + short description) and ask which they want. They
can pick:

- one or more specific servers by name (e.g. `time`, `github`), or
- a profile that groups several servers (e.g. `development`, `productivity`) — the
  `PROFILES` column from Step 1 shows what is available.

Do not guess destructively. If the request is ambiguous (e.g. "set up my MCP servers"),
confirm the selection before writing anything.

## Step 4 — Apply the configuration

Use `mcp-cli set` to write the selection. `set` is **declarative**: it overwrites the
target `mcp.json` with exactly the servers you select, so include everything the user
wants in a single command. The examples below omit `-t` to use the default tool from
Step 2 — add `-t <shortcut>` only when the user wants a different tool.

**A single server:**
```bash
mcp-cli set -s time
```

**Multiple specific servers** — repeat `-s` for each server:
```bash
mcp-cli set -s aws-remote -s terraform
```

**All servers in a profile:**
```bash
mcp-cli set development
```
A profile selection also includes any default servers (those labeled `default` or with no
profile label).

**Target a different tool** than the default:
```bash
mcp-cli set development -t cursor
```

**Write to a custom path** instead of a tool shortcut or the default:
```bash
mcp-cli set development -c /path/to/mcp.json
```

`set` reports the file it wrote, e.g. `Wrote /Users/you/.kiro/settings/mcp.json`.

### Choosing between `-s` and a profile

- **Ad hoc selection** — pass each chosen server with its own `-s` flag in a single
  `set` command: `mcp-cli set -s <a> -s <b> -s <c>`. All named servers are resolved by
  name across the registry regardless of profile.
- **A predefined group** — if the user's picks line up with an existing profile, use the
  profile instead: `mcp-cli set <profile>`. If they'll reuse the same ad hoc set often,
  consider adding a shared `mcp.profile: <name>` label to those servers in
  `mcp-compose.yml` (confirm before editing their registry).
- Remember `set` is declarative: put every server the user wants in one command, because
  each run overwrites the target config rather than adding to it.

### Clearing

To remove all MCP servers from the default tool's config (add `-t <shortcut>` for a
different tool):
```bash
mcp-cli clear
```

## Step 5 — Verify

Confirm the result and report it back to the user. Scope the status to the tool you
configured (`-t <shortcut>`), or omit `-t` to see all tools:

```bash
mcp-cli ls -as -t kiro
```

Status indicators: `✓` configured and matching, `✗` not configured, `~` configured but
differs from the registry, `?` unable to read the tool config.

Most tools read `mcp.json` at startup, so remind the user they may need to restart the
tool (or reload its MCP servers) for changes to take effect.

## Notes

- `set` overwrites, it does not merge — every apply is the full desired state.
- Avoid `-c`/`-l` on `ls` in user-facing output; they can expose secrets.
- `-s <server>` resolves by name across all servers regardless of profile, and can be
  repeated to include several servers in one `set` command.
- Point `-f <file>` at a specific registry if the user is not using the default locations.
