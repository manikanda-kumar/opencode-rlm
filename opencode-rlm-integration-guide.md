# OpenCode RLM Integration Guide

This guide covers two RLM implementations for OpenCode and how to integrate them into your OpenCode installation.

---

## Overview: Two Repos, Two Approaches

| | **opencode_RLM** (SRE Edition) | **ralph-rlm** |
|---|---|---|
| **Repo** | `.opencode_rlm/` | `.opencode-ralph-rlm/` |
| **Source** | Adapted from [claude_code_RLM](https://github.com/Brainqub3/claude_code_RLM) | [github.com/doeixd/opencode-ralph-rlm](https://github.com/doeixd/opencode-ralph-rlm) |
| **Type** | Config-based (agents, skills, commands) | TypeScript plugin (`@opencode-ai/plugin`) |
| **RLM mechanism** | Python REPL + subagent for chunk analysis | File-first worker sessions with automatic retry loop |
| **Use case** | Analyzing large files (logs, K8s manifests, Terraform state) | Iterative coding loop — describe a goal, walk away, come back to working code |
| **Automation** | Manual `/rlm` command triggers chunk workflow | Fully automated: verify → fail → retry loop until tests pass |
| **Dependencies** | Python 3 | Bun, `effect` npm package |
| **Complexity** | Low (copy files) | Medium (plugin install + config) |

### When to use which

- **opencode_RLM**: You have a large file (logs, configs, manifests) that exceeds context limits and need to analyze it chunk by chunk.
- **ralph-rlm**: You have a coding task and want the agent to iterate autonomously until `verify.command` passes, with fresh context windows per attempt.
- **Both together**: Use opencode_RLM's skills/agents for analysis tasks, and ralph-rlm's plugin for autonomous coding loops.

---

## Option A: Install opencode_RLM (Config-Based RLM)

This approach uses OpenCode's built-in extensibility (agents, skills, commands) — no plugin code required.

### Prerequisites

- OpenCode installed
- Python 3 available on `PATH`
- API keys configured for your model

### Step-by-step

#### 1. Copy the `.opencode/` directory into your project

```bash
# From the repo root
cp -r .opencode_rlm/.opencode/ /path/to/your-project/.opencode/
```

This installs:
- `.opencode/agents/rlm-subcall.md` — subagent definition for chunk analysis
- `.opencode/skills/rlm/SKILL.md` — the RLM skill instructions
- `.opencode/skills/rlm/scripts/rlm_repl.py` — persistent Python REPL
- `.opencode/commands/rlm.md` — the `/rlm` slash command

#### 2. Merge agent definitions into your `opencode.json`

Add the agent configs from `.opencode_rlm/opencode.json` into your project's `opencode.json`. At minimum, add the `rlm-subcall` subagent:

```json
{
  "agent": {
    "rlm-subcall": {
      "description": "RLM sub-LLM for chunk-level analysis",
      "mode": "subagent",
      "temperature": 0.1,
      "tools": {
        "read": true,
        "write": false,
        "edit": false,
        "bash": false,
        "webfetch": false
      }
    }
  }
}
```

Optionally add the SRE-specific agents (`sre-build`, `sre-plan`, `log-analyzer`, `k8s-expert`, `terraform-expert`, `podman-expert`) and commands (`/incident`, `/k8s-debug`, `/tf-plan`, etc.).

#### 3. (Optional) Copy prompts and knowledge bases

```bash
cp -r .opencode_rlm/prompts/ /path/to/your-project/prompts/
cp -r .opencode_rlm/knowledge/ /path/to/your-project/knowledge/
```

If you do, also add this to your `opencode.json`:

```json
{
  "instructions": [
    "AGENTS.md",
    "./knowledge/*.md"
  ]
}
```

#### 4. Create a context directory

```bash
mkdir -p /path/to/your-project/context/
```

Place large files here for RLM analysis.

#### 5. Usage

```bash
cd /path/to/your-project
opencode

# In the OpenCode TUI:
/rlm context=./context/large-logfile.log query="Find all errors and root causes"
```

The workflow:
1. Python REPL loads the file and chunks it
2. Subagent `@rlm-subcall` analyzes each chunk, returning structured JSON
3. Root LLM synthesizes chunk results into a final answer

---

## Option B: Install ralph-rlm (Plugin-Based Autonomous Loop)

This is a full OpenCode plugin that implements the Ralph outer loop (supervisor) + RLM inner loop (file-first workers).

### Prerequisites

- OpenCode installed
- [Bun](https://bun.sh) runtime available
- API keys configured for your model

### Step-by-step

#### 1. Copy the plugin file

```bash
# Project-level install (recommended)
mkdir -p /path/to/your-project/.opencode/plugins/
cp .opencode-ralph-rlm/.opencode/plugins/ralph-rlm.ts \
   /path/to/your-project/.opencode/plugins/ralph-rlm.ts
```

Or install globally:

```bash
cp .opencode-ralph-rlm/.opencode/plugins/ralph-rlm.ts \
   ~/.config/opencode/plugins/ralph-rlm.ts
```

#### 2. Add the `effect` dependency

Create (or update) `.opencode/package.json`:

```json
{
  "type": "module",
  "dependencies": {
    "effect": "^3.13.0",
    "@opencode-ai/plugin": "1.1.57"
  }
}
```

OpenCode runs `bun install` at startup automatically.

#### 3. Create `.opencode/ralph.json`

```json
{
  "enabled": true,
  "autoStartOnMainIdle": false,
  "statusVerbosity": "normal",
  "maxAttempts": 25,
  "heartbeatMinutes": 15,
  "verifyTimeoutMinutes": 15,
  "verify": {
    "command": ["bun", "run", "verify"],
    "cwd": "."
  },
  "gateDestructiveToolsUntilContextLoaded": true,
  "maxRlmSliceLines": 200,
  "requireGrepBeforeLargeSlice": true,
  "grepRequiredThresholdLines": 120,
  "subAgentEnabled": true,
  "maxSubAgents": 5,
  "maxConversationLines": 1200,
  "conversationArchiveCount": 3,
  "reviewerEnabled": false,
  "reviewerRequireExplicitReady": true,
  "reviewerMaxRunsPerAttempt": 1,
  "reviewerOutputDir": ".opencode/reviews",
  "reviewerPostToConversation": true,
  "agentMdPath": "AGENT.md"
}
```

**Important:** Update `verify.command` to match your project's actual test/build command:

| Project type | verify.command |
|---|---|
| Bun/TypeScript | `["bun", "run", "verify"]` or `["bun", "test"]` |
| npm | `["npm", "test"]` |
| Cargo/Rust | `["cargo", "test"]` |
| Python | `["python", "-m", "pytest"]` |
| Make | `["make", "test"]` |

#### 4. (Optional) Copy agent profiles

```bash
cp -r .opencode-ralph-rlm/.opencode/agents/ \
   /path/to/your-project/.opencode/agents/
```

This provides:
- `supervisor` — primary agent for safe loop orchestration
- `ralph-reviewer` — read-only quality review subagent
- `docs-writer` — documentation subagent
- `security-auditor` — read-only security review subagent

#### 5. Create your AGENT.md

Create a project-level `AGENT.md` at the repo root with project-specific rules. Ralph loads this via `ralph_load_context()` for every worker.

#### 6. Usage

```bash
cd /path/to/your-project
opencode
```

In the OpenCode TUI:

```
# 1. Check setup
ralph_doctor(autofix=true)

# 2. Create a plan (interactive wizard)
ralph_quickstart_wizard(
  goal="Implement user authentication",
  requirements="JWT-based, refresh tokens",
  stopping_conditions="All tests pass, types check",
  features="login, logout, token refresh",
  steps="...",
  todos="..."
)

# 3. Start the loop
ralph_create_supervisor_session(start_loop=true)

# 4. Monitor progress
ralph_supervision_status()
```

The loop runs automatically:
1. Worker loads context from protocol files (fresh context window)
2. Worker implements changes
3. Worker calls `ralph_verify()` → runs your `verify.command`
4. If fail → state rolls over, strategist (you) adjusts plan, new worker spawns
5. If pass → done

### Key tools available in ralph-rlm

| Tool | Purpose |
|---|---|
| `ralph_load_context()` | Load all protocol files (required first step for workers) |
| `ralph_spawn_worker()` | Spawn a fresh RLM worker session |
| `ralph_verify()` | Run verify.command and report result |
| `ralph_doctor(autofix?)` | Check setup readiness |
| `ralph_quickstart_wizard(...)` | Generate PLAN.md + TODOS interactively |
| `ralph_bootstrap_plan(...)` | Bootstrap PLAN.md from structured input |
| `ralph_create_supervisor_session(...)` | Bind supervisor and start loop |
| `ralph_pause_supervision()` | Pause the loop |
| `ralph_resume_supervision()` | Resume the loop |
| `ralph_end_supervision()` | Stop the loop |
| `ralph_report(message)` | Fire-and-forget progress update |
| `ralph_ask(question)` | Block until human answers |
| `ralph_respond(id, answer)` | Answer a pending question |
| `rlm_grep(pattern, file?)` | Grep large reference files |
| `rlm_slice(file, start, end)` | Read a line range from a file |
| `subagent_spawn(name, goal)` | Spawn a child session for parallel work |
| `subagent_await(name)` | Wait for sub-agent completion |

---

## Option C: Use Both Together

You can combine both approaches in the same project for maximum coverage.

### Directory structure

```
your-project/
├── AGENT.md                              # Project rules (for ralph-rlm)
├── opencode.json                         # Merged config
├── context/                              # Large files for RLM analysis
├── .opencode/
│   ├── package.json                      # effect dependency (for ralph-rlm)
│   ├── ralph.json                        # Ralph loop config
│   ├── plugins/
│   │   └── ralph-rlm.ts                  # Ralph plugin
│   ├── agents/
│   │   ├── rlm-subcall.md                # Chunk analyzer (from opencode_RLM)
│   │   ├── supervisor.md                 # Loop supervisor (from ralph-rlm)
│   │   ├── ralph-reviewer.md             # Code reviewer (from ralph-rlm)
│   │   ├── docs-writer.md               # Docs subagent (from ralph-rlm)
│   │   └── security-auditor.md          # Security subagent (from ralph-rlm)
│   ├── skills/
│   │   └── rlm/
│   │       ├── SKILL.md                  # RLM skill (from opencode_RLM)
│   │       └── scripts/
│   │           └── rlm_repl.py           # Python REPL (from opencode_RLM)
│   ├── commands/
│   │   └── rlm.md                        # /rlm slash command (from opencode_RLM)
│   └── reviews/                          # Ralph reviewer output
├── prompts/                              # Agent prompts (from opencode_RLM)
└── knowledge/                            # Knowledge bases (from opencode_RLM)
```

### Merged `opencode.json` example

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "sre-build": {
      "description": "Full SRE/DevOps agent with all tools enabled",
      "mode": "primary",
      "prompt": "{file:./prompts/sre-build.md}",
      "tools": {
        "write": true, "edit": true, "bash": true,
        "read": true, "grep": true, "glob": true,
        "webfetch": true, "skill": true
      }
    },
    "sre-plan": {
      "description": "Read-only analysis and planning agent",
      "mode": "primary",
      "prompt": "{file:./prompts/sre-plan.md}",
      "temperature": 0.1,
      "tools": {
        "write": false, "edit": false, "bash": true,
        "read": true, "grep": true, "glob": true,
        "webfetch": true, "skill": true
      }
    },
    "rlm-subcall": {
      "description": "RLM sub-LLM for chunk-level analysis",
      "mode": "subagent",
      "temperature": 0.1,
      "tools": {
        "read": true, "write": false, "edit": false,
        "bash": false, "webfetch": false
      }
    }
  },
  "command": {
    "rlm": {
      "template": "Load the rlm skill to process a large context file. $ARGUMENTS",
      "description": "Run RLM workflow for large context processing",
      "agent": "sre-build"
    }
  },
  "instructions": ["AGENTS.md", "./knowledge/*.md"],
  "permission": {
    "edit": "allow",
    "bash": "allow",
    "skill": { "*": "allow" }
  }
}
```

### Workflow split

| Task | Use |
|---|---|
| Analyze a 500MB log file | `/rlm` command (opencode_RLM skill + Python REPL + rlm-subcall subagent) |
| Implement a feature until tests pass | Ralph loop (`ralph_create_supervisor_session(start_loop=true)`) |
| Review Terraform plan output | `/rlm` or `@terraform-expert` subagent |
| Overnight autonomous coding | Ralph loop with `maxAttempts: 50` |
| Incident investigation | `/incident` command (opencode_RLM) |

---

## Troubleshooting

| Problem | Solution |
|---|---|
| Plugin not loading | Ensure `ralph-rlm.ts` is in `.opencode/plugins/` and `.opencode/package.json` has `effect` dependency |
| `bun install` fails | Check Bun is installed: `bun --version` |
| Python REPL errors | Check Python 3 is available: `python3 --version` |
| Ralph loop not starting | Run `ralph_doctor(autofix=true)`, ensure `verify.command` is set |
| verify always failing | Test your verify command manually: e.g., `bun run verify` |
| RLM chunks too large/small | Adjust `chunk_chars` in `/rlm` command or `maxRlmSliceLines` in `ralph.json` |
| Workers not spawning | Ensure `ralph_create_supervisor_session()` was called first |
| Prompts not loading | Check `{file:./prompts/...}` paths are relative to `opencode.json` location |

---

## Emerging Alternatives

The RLM ecosystem is evolving fast. Consider these newer approaches that may simplify your setup:

| Approach | Why Consider It | Link |
|----------|----------------|------|
| **Official `pip install rlms`** | Paper authors' canonical lib with pluggable environments (local, docker, modal, e2b) and multi-provider support. | [alexzhang13/rlm](https://github.com/alexzhang13/rlm) |
| **RLM as MCP Server** | Cleaner integration — exposes RLM tools via Model Context Protocol. Free local sub-queries via Ollama. | [richardwhiteii/rlm](https://github.com/richardwhiteii/rlm) |
| **Bash-first RLM (no Python)** | Eliminates Python dependency entirely. True depth-N recursion via `claude -p`. ~250 lines of bash. | [Tenobrus/claude-rlm](https://github.com/Tenobrus/claude-rlm) |
| **Programmatic `sub_llm()` (upcoming)** | [OpenCode Issue #8554](https://github.com/anomalyco/opencode/issues/8554) — built-in tool for `sub_llm()` in loops. 3 lines instead of 10,000 tool calls. | Pending in OpenCode core |
| **Trained RLM via RL** | Prime Intellect is training models to manage context autonomously using `RLMEnv`. | [PrimeIntellect-ai/verifiers](https://github.com/PrimeIntellect-ai/verifiers/) |

**Key finding**: RLM overhead only pays off above ~50KB of context. Below that, direct processing wins. ([Source](https://medium.com/@constantine124/exploring-rlm-part-2-context-engineering-for-coding-agents-b05befc3851d))

---

## References

- [RLM Paper (arXiv:2512.24601)](https://arxiv.org/abs/2512.24601) — Zhang, Kraska, Khattab (MIT CSAIL)
- [RLM Official Implementation](https://github.com/alexzhang13/rlm) — Canonical Python lib (`pip install rlms`)
- [RLM Minimal](https://github.com/alexzhang13/rlm-minimal) — 2-file gist to understand the core loop
- [claude_code_RLM](https://github.com/Brainqub3/claude_code_RLM) — Original Claude Code RLM implementation
- [ralph-rlm](https://github.com/doeixd/opencode-ralph-rlm) — Ralph + RLM OpenCode plugin
- [Prime Intellect RLM Blog](https://www.primeintellect.ai/blog/rlm) — "Recursive Language Models: the paradigm of 2026"
- [RLM Context Engineering (Cui Xiao)](https://medium.com/@constantine124/exploring-rlm-part-2-context-engineering-for-coding-agents-b05befc3851d) — Practical RLM patterns and crossover analysis
- [OpenCode Docs](https://opencode.ai/docs/) — Official OpenCode documentation
