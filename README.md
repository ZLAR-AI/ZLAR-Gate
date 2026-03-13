# ZLAR Gate

**One gate. Your rules. Every agent framework.**

ZLAR Gate is a policy enforcement layer for AI coding agents. It intercepts every tool call — shell commands, file writes, network requests — and evaluates it against your signed policy before it executes. Actions are allowed instantly, denied instantly, or held for your approval via Telegram.

Works with **Claude Code**, **Cursor**, and **Windsurf**. Same policy, same audit trail, same human authority across all three.

## Why

AI coding agents run with your permissions. They can delete files, push code, install packages, and make network requests. Most people turn off the safety prompts because they're annoying. Then something goes wrong.

ZLAR Gate puts you back in control without slowing you down. Safe actions flow through. Dangerous actions stop and ask. Everything is logged.

## How It Works

```
Agent (Claude Code / Cursor / Windsurf)
    │
    │ Framework hook — synchronous, fail-closed
    ▼
adapters/{framework}/hook.sh — thin translator
    │
    │ Normalized tool call
    ▼
bin/zlar-gate — core engine (857 lines, bash + jq)
    ├── Classifies tool call into domain (bash, write, edit, read, etc.)
    ├── Evaluates against Ed25519-signed JSON policy
    ├── Routes: allow | deny | ask (Telegram) | log
    └── Writes JSONL audit trail with hash chain
    │
    │ Decision
    ▼
adapters/{framework}/hook.sh — translates back to framework format
    │
    ▼
Agent proceeds (or doesn't)
```

The core gate engine is the same regardless of framework. Each adapter is a ~80-line translator between the framework's hook format and the gate's native format.

## Quick Start

```bash
# Clone
git clone https://github.com/ZLAR-AI/ZLAR-Gate.git
cd ZLAR-Gate

# Run setup — picks your framework, checks prerequisites, generates keys, signs policy
./scripts/zlar-setup.sh

# Edit your Telegram credentials
vim etc/gate.json    # set telegram.chat_id
vim .env             # set TELEGRAM_BOT_TOKEN

# Re-sign policy after any changes
bin/zlar-policy sign --input etc/policies/active.policy.json --key ~/.zlar-signing.key

# Done. Open your editor. ZLAR is gating every tool call.
```

The setup script handles everything: prerequisite checks (jq, openssl with Ed25519, curl, bash 4+), config file templating, Ed25519 key generation, policy signing, and hook configuration for your chosen framework.

## Supported Frameworks

### Claude Code

Hook: `PreToolUse` — intercepts all 10 built-in tools before execution.

The Claude Code adapter is a pass-through — the gate's native format matches Claude Code's hook contract. Decision is returned as JSON on stdout.

### Cursor

Hooks: `beforeShellExecution`, `beforeReadFile`, `beforeMCPExecution` — intercepts shell commands, file reads, and MCP tool calls.

The Cursor adapter translates Cursor's event format to the gate's format and returns `{"permission":"allow"|"deny"}` as Cursor expects.

### Windsurf

Hooks: `pre_run_command`, `pre_write_code`, `pre_read_code`, `pre_mcp_tool_use` — intercepts commands, file writes, file reads, and MCP tool calls.

The Windsurf adapter translates Cascade's event format and returns decisions via exit codes (0 = allow, 2 = block with reason on stderr).

## Architecture

### One Policy, Many Frameworks

The core insight: Claude Code, Cursor, and Windsurf all have hook systems that intercept agent tool calls before execution. The hook protocols differ — different JSON schemas, different response formats — but the *decision* is the same: should this action proceed?

ZLAR Gate separates the decision engine from the framework integration:

- **Core engine** (`bin/zlar-gate`): reads a normalized tool call, evaluates policy, asks Telegram if needed, writes audit log, returns allow/deny. Framework-agnostic.
- **Adapters** (`adapters/{framework}/hook.sh`): translate between the framework's hook protocol and the gate's format. ~80 lines each. No policy logic.

This means:
- One policy file governs all frameworks
- One audit trail captures all actions
- One Telegram bot serves all approval requests
- Adding a new framework = writing one adapter script

### Policy

JSON format with Ed25519 signature. Rules match by domain (bash, write, edit, read, glob, grep, webfetch, websearch, notebook, agent) and detail fields (regex, contains, prefix, eq, not_regex). Actions: allow, deny, ask, log.

```json
{
  "version": "1.0",
  "author": "you",
  "default_action": "deny",
  "rules": [
    {
      "id": "bash-safe",
      "domain": "bash",
      "match": {"detail": {"command": {"regex": "^(ls|pwd|echo|cat|head|wc|date|whoami)"}}},
      "action": "allow",
      "severity": "info"
    },
    {
      "id": "bash-delete",
      "domain": "bash",
      "match": {"detail": {"command": {"regex": "rm\\s+-rf|rm\\s+-r|rmdir"}}},
      "action": "deny",
      "severity": "critical"
    }
  ],
  "signature": {"algorithm": "ed25519", "value": "..."}
}
```

The policy is a human artifact. It is signed with your private key. The gate verifies the signature on every load. AI cannot modify the rules that govern it.

### Telegram Ask

For "ask" actions, the gate sends an inline-keyboard message to your Telegram chat with Approve/Deny buttons. The agent blocks until you respond or the timeout expires (default: 5 minutes). Timeout = deny.

### Audit Trail

Every decision is written to `var/log/audit.jsonl` as a hash-chained JSONL record. Each entry includes: timestamp, tool, domain, detail, matched rule, decision, risk score, session ID, and the SHA-256 hash of the previous entry. Tamper-evident by construction.

### Fail-Closed

Everything fails closed:
- Gate script missing → deny
- Gate crashes → deny
- Policy file missing → deny
- Policy signature invalid → deny
- Telegram unreachable → deny (for ask actions)
- Telegram timeout → deny
- Unknown tool → deny
- No matching rule → deny (default_action)

## Repo Structure

```
ZLAR-Gate/
├── bin/
│   ├── zlar-gate              # Core gate engine
│   └── zlar-policy            # Policy CLI (keygen, sign, verify, deploy)
├── adapters/
│   ├── claude-code/hook.sh    # Claude Code adapter (pass-through)
│   ├── cursor/hook.sh         # Cursor adapter (JSON translator)
│   └── windsurf/hook.sh       # Windsurf adapter (exit-code translator)
├── etc/
│   ├── gate.example.json      # Gate config template
│   ├── keys/                  # Public keys (private key goes to ~/.zlar-signing.key)
│   └── policies/
│       └── default.policy.example.json  # 19-rule starter policy
├── scripts/
│   └── zlar-setup.sh          # One-command setup
├── var/
│   └── log/                   # Audit trail, gate logs, sessions
├── .env.example               # Telegram token template
├── CLAUDE.md                  # Project guide for AI agents
└── README.md
```

## Requirements

- **bash** 4+ (macOS users: `brew install bash`)
- **jq** (`brew install jq` or `apt install jq`)
- **openssl** with Ed25519 support (macOS users may need: `brew install openssl`)
- **curl** (for Telegram)
- **Telegram bot** (optional — without it, "ask" actions deny on timeout)

## Adding a New Framework

When a new agent framework ships pre-execution hooks (looking at you, Codex CLI), adding support is:

1. Create `adapters/{framework}/hook.sh`
2. Read the framework's stdin JSON
3. Translate to: `{"tool_name":"...","tool_input":{...},"session_id":"..."}`
4. Pipe to `bin/zlar-gate`
5. Translate the gate's response to the framework's expected format
6. Add a `configure_{framework}` function to `scripts/zlar-setup.sh`

That's it. The gate, policy, Telegram, and audit trail are shared.

## Policy Management

```bash
# Generate signing keys (first time only)
bin/zlar-policy keygen

# Edit policy
vim etc/policies/active.policy.json

# Sign after changes
bin/zlar-policy sign --input etc/policies/active.policy.json --key ~/.zlar-signing.key

# Verify
bin/zlar-policy verify --input etc/policies/active.policy.json
```

## FAQ

**Does this slow down my agent?**
Allow/deny decisions take <50ms. Telegram ask actions take however long you take to tap Approve/Deny.

**What if the gate crashes?**
All actions denied. Fail-closed, always.

**Can the AI modify its own policy?**
No. The policy is Ed25519-signed with your private key. The gate verifies the signature. An agent could write to the policy file, but the gate would reject the invalid signature and deny everything.

**Why bash and not Python/Node/Rust?**
Zero dependencies beyond jq and openssl. No build step. No runtime. No package manager. Runs on any Unix system. The gate is 857 lines — you can read the whole thing in 20 minutes.

**Why not just use the built-in permission prompts?**
Because you'll turn them off. Everyone does. ZLAR Gate gives you the control without the friction — safe actions flow through, only the dangerous ones stop.

**Are the Cursor and Windsurf adapters tested?**
The Claude Code adapter is verified — it uses the same hook format as ZLAR-CC, which has been tested in production. The Cursor and Windsurf adapters are built from framework hook documentation and have not yet been tested against live hook payloads. If you use ZLAR Gate with Cursor or Windsurf and encounter issues with the adapter, please open an issue — your real-world payloads will help us verify and fix.

## The ZLAR Family

| Product | Platform | What it does |
|---------|----------|-------------|
| **[ZLAR-OC](https://github.com/ZLAR-AI/ZLAR-OC)** | OpenClaw | OS-level containment — user isolation, kernel sandbox, pf firewall, gate daemon, signed policy, audit trail |
| **[ZLAR-CC](https://github.com/ZLAR-AI/ClaudeCode_ZLAR-CC)** | Claude Code | Hook-based gate — tool-call interception, risk classification, signed policy, Telegram approval |
| **ZLAR Gate** (this repo) | Claude Code + Cursor + Windsurf | Universal gate — one policy across multiple editors, framework-specific adapters |
| **[ZLAR-LT](https://github.com/ZLAR-AI/ZLAR-LT)** | Claude Code + Cursor + Windsurf | Zero-config governance — one command, instant protection, deny-heavy defaults |

---

## License

[Apache License 2.0](LICENSE)

## Links

- [ZLAR Gate Product Page](https://zlar.ai/zlar-gate.html)
- [ZLAR-CC (Claude Code specific)](https://github.com/ZLAR-AI/ClaudeCode_ZLAR-CC)
- [ZLAR-OC (OS-level containment)](https://github.com/ZLAR-AI/ZLAR-OC)
- [ZLAR Website](https://zlar.ai)
