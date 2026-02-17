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

## Reference Repos

These repos were studied during development:

| Repo | What it provides |
|------|-----------------|
| [opencode_RLM](https://github.com/) (SRE Edition) | Config-based RLM with SRE/DevOps specialization — agents, skills, commands, knowledge bases |
| [ralph-rlm](https://github.com/doeixd/opencode-ralph-rlm) | TypeScript plugin: Ralph outer loop (supervisor) + RLM inner loop (file-first workers) with automatic retry |
| [BowTiedSwan/rlm-skill](https://github.com/BowTiedSwan/rlm-skill) | Lightweight RLM skill — dual-mode routing pattern and "context-as-resource" philosophy |
| [claude_code_RLM](https://github.com/Brainqub3/claude_code_RLM) | Original Claude Code RLM implementation (inspiration for all OpenCode ports) |

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
