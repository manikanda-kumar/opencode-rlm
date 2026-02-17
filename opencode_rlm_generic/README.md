# OpenCode RLM — Generic Edition

A Recursive Language Model (RLM) implementation for OpenCode that works with any project type. Process documents, codebases, and contexts that exceed typical context window limits.

Based on [arXiv:2512.24601](https://arxiv.org/abs/2512.24601) (Zhang, Kraska, Khattab — MIT CSAIL), adapted from [claude_code_RLM](https://github.com/Brainqub3/claude_code_RLM) for OpenCode.

## How it works

A root language model orchestrates sub-LLM calls over chunks of a large document. Two primary agents collaborate:

| Agent | Mode | Purpose |
|-------|------|---------|
| `main` | Primary (full access) | Orchestrates work, makes changes, synthesizes results |
| `planner` | Primary (read-only) | Safe analysis, investigation, planning — no file edits |
| `rlm-subcall` | Subagent | Chunk-level analysis, returns structured JSON |

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

**Works with any model OpenCode supports.**

## Prerequisites

- **OpenCode** — [Install OpenCode](https://opencode.ai/docs/)
- **Python 3** — For the persistent REPL environment
- **API Keys** — API keys for your configured model in OpenCode

## Quick Start

1. **Copy this directory into your project** (or use it as your working directory):
   ```bash
   cp -r opencode_rlm_generic/ /path/to/your-project/
   # or
   cd opencode_rlm_generic && opencode
   ```

2. **Start OpenCode**:
   ```bash
   opencode
   ```

3. **Use the RLM workflow**:
   ```
   /rlm context=/path/to/large-file.txt query="Summarize the key themes and extract actionable items"
   ```

## Available Agents

### Primary Agents (Tab to switch)

| Agent | Description | Use Case |
|-------|-------------|----------|
| `main` | Full access worker | Making changes, implementing solutions, writing code |
| `planner` | Read-only analysis | Safe investigation, planning, review |

### Subagents (@mention to invoke)

| Agent | Description | Use Case |
|-------|-------------|----------|
| `rlm-subcall` | Chunk analyzer | RLM workflow chunk processing |

## Custom Commands

| Command | Description |
|---------|-------------|
| `/rlm` | Run RLM workflow for large context processing |
| `/analyze` | General-purpose analysis of any file or codebase |
| `/summarize` | Summarize a large document or dataset |

## RLM REPL Commands

The Python REPL provides these helper functions:

```python
# Basic operations
peek(start, end)              # View slice of content
grep(pattern, max_matches)    # Search with regex
grep_count(pattern)           # Count pattern occurrences
find_lines(pattern)           # Find matching lines with numbers

# Chunking
chunk_indices(size, overlap)  # Get chunk boundaries
write_chunks(out_dir, size)   # Write chunks to files

# Analysis helpers
stats()                       # Get content statistics
time_range()                  # Extract time range (if timestamps present)
extract_json_objects()        # Parse JSONL content
extract_yaml_documents()      # Split YAML documents

# State management
add_buffer(text)              # Store intermediate results
```

## Repository Structure

```
opencode_rlm_generic/
├── AGENTS.md                      # Main agent instructions (generic)
├── opencode.json                  # OpenCode configuration
├── README.md                      # This file
├── .opencode/
│   ├── agents/
│   │   └── rlm-subcall.md        # Sub-LLM agent definition
│   ├── skills/
│   │   └── rlm/
│   │       ├── SKILL.md          # RLM skill definition
│   │       └── scripts/
│   │           └── rlm_repl.py   # Persistent Python REPL
│   └── commands/                  # Custom slash commands
│       ├── rlm.md
│       ├── analyze.md
│       └── summarize.md
├── prompts/
│   ├── main.md                   # Main worker agent prompt
│   └── planner.md                # Planner agent prompt
├── context/                       # Place large files here for RLM processing
└── examples/                      # Example workflows
    └── example-workflows.md
```

## Example Use Cases

### Codebase Analysis
```
/rlm context=./context/codebase-dump.txt query="Map the architecture and identify code smells"
```

### Document Summarization
```
/rlm context=./context/research-papers.txt query="Extract key findings and methodology from each paper"
```

### Data Processing
```
/rlm context=./context/dataset.csv query="Find anomalies and statistical outliers"
```

### Log Analysis
```
/rlm context=./context/app.log query="Find all errors and identify the root cause"
```

### Configuration Review
```
/rlm context=./context/config-dump.yaml query="Identify misconfigurations and security issues"
```

## Security Notes

- The REPL executes arbitrary Python code — treat it like running your own scripts
- Sensitive data in context files stays local (not sent to cloud unless in chunks)
- Use `planner` agent for safe, read-only investigation

## License

MIT License
