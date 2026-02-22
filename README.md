# OpenCode RLM

Recursive Language Model (RLM) implementations and integration guides for [OpenCode](https://opencode.ai) — process documents, codebases, and contexts that exceed typical context window limits.

Based on [Recursive Language Models (arXiv:2512.24601)](https://arxiv.org/abs/2512.24601) by Zhang, Kraska, Khattab (MIT CSAIL).

## What's in this repo

```
opencode-rlm/
├── opencode_rlm_generic/              # ← Ready-to-use generic RLM setup
├── opencode-plugins-extensions.md     # OpenCode plugin system reference
├── opencode-rlm-integration-guide.md  # Integration guide for all RLM approaches
└── README.md                          # This file
```

### `opencode_rlm_generic/` — Generic RLM Setup

A drop-in RLM implementation that works with **any project type** — code, logs, data, documents, configs.

**Features:**
- **Dual-mode routing** — Native Mode (grep/find) for quick searches, Strict Mode (Python REPL + subagents) for dense analysis
- **`init-dir` command** — Load entire directory trees with auto-exclusion of `.git`, `node_modules`, `__pycache__`, `venv`, `build`, `dist`, and more
- **Python REPL** with `peek`, `grep` (with line numbers + context window), `find_lines`, `chunk_indices`, `write_chunks`, `extract_json_objects`, `extract_yaml_documents`, `time_range`, `stats`
- **Structured subagent** (`rlm-subcall`) returns typed JSON for each chunk
- **Two primary agents**: `main` (full access worker) and `planner` (read-only analysis)
- **Slash commands**: `/rlm`, `/analyze`, `/summarize`
- **Recovery fallback** — documented degradation path when subagents aren't available

**Quick start:**
```bash
cp -r opencode_rlm_generic/.opencode /path/to/your-project/.opencode
cp opencode_rlm_generic/opencode.json /path/to/your-project/
cp -r opencode_rlm_generic/prompts /path/to/your-project/

cd /path/to/your-project && opencode

# In OpenCode:
/rlm context=./big-file.log query="Find all errors and root causes"
```

See [`opencode_rlm_generic/README.md`](opencode_rlm_generic/README.md) for full documentation.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Root LLM (main)                     │
│                   Main Conversation                      │
│                  Orchestrator / Worker                   │
└──────────────────────┬──────────────────────────────────┘
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  Python REPL    │         │  rlm-subcall    │
│  (Environment)  │         │  (Subagent)     │
│                 │         │                 │
│ - Load context  │         │ - Analyze chunk │
│ - Chunk data    │         │ - Extract info  │
│ - Search/grep   │         │ - Return JSON   │
│ - Store results │         │                 │
└─────────────────┘         └─────────────────┘
```

The core principle: **Context is an external resource, not a local variable.** Don't try to fit large content into the chat context — treat the filesystem as a database and query it surgically via the REPL and subagents.

## RLM Approaches Compared

This repo documents three approaches to RLM in OpenCode:

| Approach | Type | Best for | Automation |
|----------|------|----------|------------|
| **[opencode_rlm_generic](opencode_rlm_generic/)** | Config-based (agents, skills, commands) | Analyzing large files of any kind | Manual `/rlm` command |
| **opencode_RLM** (SRE Edition) | Config-based, SRE-specialized | SRE/DevOps: logs, K8s, Terraform | Manual `/rlm` + domain commands |
| **[ralph-rlm](https://github.com/doeixd/opencode-ralph-rlm)** | TypeScript plugin | Autonomous coding loops | Fully automated verify→retry loop |

See [`opencode-rlm-integration-guide.md`](opencode-rlm-integration-guide.md) for detailed installation and integration instructions for all three.

## Recent Developments & Simplified Approaches

The RLM ecosystem has evolved rapidly since the original paper. Several newer approaches simplify or extend the pattern:

### Simpler Alternatives

| Approach | What's New | Link |
|----------|-----------|------|
| **Bash-as-REPL** | No Python needed — filesystem IS the variable store. True depth-N recursion via `claude -p` sub-agents. ~250 lines of pure bash. | [Tenobrus/claude-rlm](https://github.com/Tenobrus/claude-rlm) |
| **87-line RLM skill** | Minimal dual-mode (Native + Strict) with `curl \| bash` install. No pickle, no state machine. | [BowTiedSwan/rlm-skill](https://github.com/BowTiedSwan/rlm-skill) |
| **`pip install rlms`** | Official lib now pip-installable with pluggable REPL environments (local, docker, modal, e2b) and multi-provider support. | [alexzhang13/rlm](https://github.com/alexzhang13/rlm) |
| **RLM as MCP Server** | Exposes RLM tools via MCP protocol. Ollama support for free local sub-queries. | [richardwhiteii/rlm](https://github.com/richardwhiteii/rlm) |
| **Recursive Decomposition Skill** | Claude Code Marketplace plugin — auto-activates on large-scale tasks, no manual `/rlm` needed. | [massimodeluisa/recursive-decomposition-skill](https://github.com/massimodeluisa/recursive-decomposition-skill) |
| **Pydantic AI RLM** | Provider-agnostic library with grounded citations (`[N]` markers traceable to source quotes). | [vstorm-co/pydantic-ai-rlm](https://github.com/vstorm-co/pydantic-ai-rlm) |

### Key Insights from the Community

- **Crossover point**: RLM overhead only pays off above ~50KB of context. Below that, direct processing wins. ([Cui Xiao's analysis](https://medium.com/@constantine124/exploring-rlm-part-2-context-engineering-for-coding-agents-b05befc3851d))
- **Programmatic sub-LLM calls**: [OpenCode Issue #8554](https://github.com/anomalyco/opencode/issues/8554) proposes a built-in `rlm_repl` tool with `sub_llm()` in loops — 3 lines instead of 10,000 tool calls. This is the direction OpenCode is heading.
- **Trained RLM**: [Prime Intellect](https://www.primeintellect.ai/blog/rlm) is training models to manage their own context via RL using their [verifiers](https://github.com/PrimeIntellect-ai/verifiers/) framework — making the scaffolding learnable, not hand-coded.
- **Context folding**: The RLM paper authors now have a [minimal 2-file implementation](https://github.com/alexzhang13/rlm-minimal) for understanding the core loop without noise.

## Reference Repos

### Foundational

| Repo | What it provides |
|------|-----------------|
| [alexzhang13/rlm](https://github.com/alexzhang13/rlm) ⭐2655 | **Official** Python lib by paper authors. `pip install rlms`. Pluggable REPL environments, multi-provider, trajectory visualizer. |
| [alexzhang13/rlm-minimal](https://github.com/alexzhang13/rlm-minimal) ⭐687 | Gist-style 2-file reimplementation — best way to understand the core loop. |
| [claude_code_RLM](https://github.com/Brainqub3/claude_code_RLM) ⭐345 | Original Claude Code RLM skill (inspiration for all OpenCode ports). Pickle-backed REPL + Haiku subagent. |

### Claude Code / OpenCode Skills

| Repo | What it provides |
|------|-----------------|
| [BowTiedSwan/rlm-skill](https://github.com/BowTiedSwan/rlm-skill) ⭐157 | Lightweight 87-line RLM skill — dual-mode routing (Native + Strict). Simplest to understand. |
| [Tenobrus/claude-rlm](https://github.com/Tenobrus/claude-rlm) ⭐67 | Bash-first RLM with true depth-N recursion. No Python needed. tmux observability, atomic concurrency limiting. |
| [massimodeluisa/recursive-decomposition-skill](https://github.com/massimodeluisa/recursive-decomposition-skill) ⭐16 | Claude Code Marketplace plugin — auto-activates on large-scale tasks. Filter → Chunk → Recurse → Verify → Synthesize. |
| [unravel-team/rlm-skills](https://github.com/unravel-team/rlm-skills) | RLM skills for both Claude Code and OpenCode. |

### Frameworks & Integrations

| Repo | What it provides |
|------|-----------------|
| [richardwhiteii/rlm](https://github.com/richardwhiteii/rlm) ⭐37 | RLM as MCP server — exposes tools via Model Context Protocol. Ollama for free local sub-queries. |
| [vstorm-co/pydantic-ai-rlm](https://github.com/vstorm-co/pydantic-ai-rlm) ⭐35 | Pydantic AI integration — provider-agnostic, grounded citations with source quote traceability. |
| [rawwerks/rlm-cli](https://github.com/rawwerks/rlm-cli) ⭐49 | CLI wrapper — `rlm ask . -q "..."`, budget propagation, mid-run injection, dir/URL/stdin as context. |
| [rand/rlm-claude-code](https://github.com/rand/rlm-claude-code) ⭐62 | Claude Code plugin with Rust core (PyO3), SQLite memory, Go hooks. Auto-activation at >80K tokens. |
| [joshua-mo-143/rig-rlm](https://github.com/joshua-mo-143/rig-rlm) ⭐61 | RLM in Rust via the `rig` framework with PyO3 REPL. |
| [PrimeIntellect-ai/verifiers](https://github.com/PrimeIntellect-ai/verifiers/) | RLM training via RL — `RLMEnv` for plug-and-play usage in any verifiers environment. |

### OpenCode-Specific

| Repo | What it provides |
|------|-----------------|
| [opencode_RLM](https://github.com/) (SRE Edition) | Config-based RLM with SRE/DevOps specialization — agents, skills, commands, knowledge bases |
| [ralph-rlm](https://github.com/doeixd/opencode-ralph-rlm) | TypeScript plugin: Ralph outer loop (supervisor) + RLM inner loop (file-first workers) with automatic retry |
| [XiaoConstantine/rlm-go](https://github.com/XiaoConstantine/rlm-go) | Go implementation that integrates as a Claude Code/OpenCode skill. Keeps large context outside the context window entirely. |

## Documentation

| Document | Description |
|----------|-------------|
| [`opencode-plugins-extensions.md`](opencode-plugins-extensions.md) | Complete reference for OpenCode's plugin/extension system (plugins, tools, MCP, skills, agents, hooks, events) |
| [`opencode-rlm-integration-guide.md`](opencode-rlm-integration-guide.md) | Step-by-step guide to integrate config-based RLM, ralph-rlm plugin, or both |
| [`opencode_rlm_generic/README.md`](opencode_rlm_generic/README.md) | Full docs for the generic RLM setup |
| [`opencode_rlm_generic/examples/`](opencode_rlm_generic/examples/) | Example workflows for code analysis, log analysis, summarization, etc. |

## Prerequisites

- **[OpenCode](https://opencode.ai/docs/)** — AI coding tool
- **Python 3** — For the persistent REPL
- **API Keys** — For your configured LLM provider

## License

MIT
