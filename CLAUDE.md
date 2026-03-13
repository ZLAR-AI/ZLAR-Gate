# ZLAR Gate — Project Guide

## What this is

ZLAR Gate is a universal policy enforcement layer for AI coding agents. One gate engine, multiple framework adapters. Same signed policy, same audit trail, same Telegram approval channel across Claude Code, Cursor, and Windsurf.

## Architecture

```
Framework hook event
    ↓
adapters/{framework}/hook.sh  — translates to gate's native format
    ↓
bin/zlar-gate                  — core engine (unchanged from ZLAR-CC)
    ↓
adapters/{framework}/hook.sh  — translates response back
    ↓
Framework receives allow/deny
```

### Adapters

Each adapter is a thin translator (~80 lines) between a framework's hook protocol and the gate's native format (Claude Code's `tool_name`/`tool_input` JSON).

- **Claude Code** (`adapters/claude-code/hook.sh`): Pass-through. CC format IS the gate's native format.
- **Cursor** (`adapters/cursor/hook.sh`): Translates Cursor events (`beforeShellExecution`, `beforeReadFile`, `beforeMCPExecution`) to CC tool calls. Returns `{"permission":"allow"|"deny"}`.
- **Windsurf** (`adapters/windsurf/hook.sh`): Translates Cascade events (`pre_run_command`, `pre_write_code`, `pre_read_code`, `pre_mcp_tool_use`) to CC tool calls. Returns exit code 0 (allow) or 2 (block, reason on stderr).

### Core Gate (`bin/zlar-gate`)

~1000-line bash script. Reads JSON on stdin, classifies tool call into domain, evaluates against Ed25519-signed JSON policy, routes action (allow/deny/ask/log), writes hash-chained JSONL audit trail. Uses jq throughout.

### Policy CLI (`bin/zlar-policy`)

Handles Ed25519 key management: keygen, sign, verify, deploy, inspect, diff, prune.

## Core principles

1. **Fail closed.** Unknown tools denied. Unmatched rules denied. Gate down = all denied.
2. **No intelligence in the gate.** Classify, halt, ask. No LLM, no ML, no heuristics.
3. **The policy is a human artifact.** Ed25519 signed. AI cannot modify the rules that govern it.
4. **Deterministic.** Same input, same output, always.
5. **Framework-agnostic core.** The gate doesn't know or care which editor sent the tool call.

## Key files

- `bin/zlar-gate` — core gate engine
- `bin/zlar-policy` — policy CLI
- `adapters/claude-code/hook.sh` — Claude Code adapter
- `adapters/cursor/hook.sh` — Cursor adapter
- `adapters/windsurf/hook.sh` — Windsurf adapter
- `etc/policies/default.policy.example.json` — starter policy (19 rules)
- `etc/gate.example.json` — gate configuration template
- `scripts/zlar-setup.sh` — unified setup for all frameworks

## Setup

```bash
./scripts/zlar-setup.sh    # picks framework, checks prereqs, generates keys, configures hooks
```

## Policy changes

```bash
# Edit policy, then re-sign:
bin/zlar-policy sign --input etc/policies/active.policy.json --key ~/.zlar-signing.key
bin/zlar-policy verify --input etc/policies/active.policy.json
```

## Adding a new framework adapter

1. Create `adapters/{framework}/hook.sh`
2. Read framework's stdin JSON
3. Translate to `{"tool_name":"...","tool_input":{...},"session_id":"..."}`
4. Pipe to `bin/zlar-gate`
5. Translate gate's `permissionDecision` to framework's response format
6. Add `configure_{framework}` to `scripts/zlar-setup.sh`
