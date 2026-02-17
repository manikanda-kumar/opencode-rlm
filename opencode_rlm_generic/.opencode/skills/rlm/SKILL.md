---
name: rlm
description: Run a Recursive Language Model-style loop for long-context tasks. Uses a persistent Python REPL and rlm-subcall subagent for chunk-level analysis. Works with any content type.
globs:
  - "**/*"
alwaysAllow:
  - "bash"
  - "read"
triggers:
  - "RLM"
  - "analyze codebase"
  - "scan all files"
  - "large repository"
  - "find usage of"
  - "process large file"
  - "context too large"
---

# rlm (Recursive Language Model workflow)

Use this Skill when:
- The user provides (or references) a very large context file that won't fit comfortably in chat context.
- You need to iteratively inspect, search, chunk, and extract information from that context.
- You need to delegate chunk-level analysis to a subagent for efficient processing.
- The user says "analyze codebase", "scan all files", "large repository", or similar.

## Core Principle

**Context is an external resource, not a local variable.**

Do not try to fit large content into the chat context. Treat the filesystem as a database.
Write programs (plans) that _query_ the data, not readers that absorb it.
The REPL and subagents are your retrieval primitives — use them surgically.

## Mental Model

- Main OpenCode conversation = the root LM (orchestrator)
- Persistent Python REPL (`rlm_repl.py`) = the external environment for state management
- Subagent `rlm-subcall` = the sub-LM used for chunk analysis (like `llm_query` in RLM paper)

## Step 0: Choose Your Mode

Before starting, decide which mode fits the task:

### Native Mode (filesystem tools — prefer when possible)

Use `grep`, `find`, `ripgrep`, or OpenCode's built-in Read/Grep tools when:
- You need to find files matching a pattern across a project
- You need a quick keyword search across many files
- The task is "find where X is defined" or "list files containing Y"
- The total relevant content is small (even if spread across many files)

```bash
# Examples of Native Mode — no REPL needed
grep -rn "TODO" src/
find . -name "*.ts" -newer src/index.ts
rg "function.*auth" --type ts
```

**If Native Mode answers the question, stop here.** Don't reach for the REPL unnecessarily.

### Strict Mode (Python REPL + subagents)

Use the full RLM workflow when:
- A single file is too large to fit in context (logs, data dumps, monorepo exports)
- You need to chunk, iterate, and synthesize across a large corpus
- The task requires extracting structured data from dense content
- Simple grep won't work because you need semantic analysis of each chunk

**Continue to Step 1 below.**

## How to Run (Strict Mode)

### Inputs

This Skill accepts these patterns:
- `context=<path>` (required): path to the file or directory containing the large context
- `query=<question>` (required): what the user wants to know
- Optional: `chunk_chars=<int>` (default ~200000) and `overlap_chars=<int>` (default 0)

If arguments weren't supplied, ask for:
1. The context file path (or directory)
2. The query/question

### Step 1: Initialize the REPL state

**For a single large file:**
```bash
python3 .opencode/skills/rlm/scripts/rlm_repl.py init <context_path>
python3 .opencode/skills/rlm/scripts/rlm_repl.py status
```

**For a directory tree** (auto-excludes .git, node_modules, __pycache__, etc.):
```bash
python3 .opencode/skills/rlm/scripts/rlm_repl.py init-dir <directory> --pattern "**/*.py"
python3 .opencode/skills/rlm/scripts/rlm_repl.py status
```

### Step 2: Scout the context quickly

```bash
# Preview beginning
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(peek(0, 3000))"

# Preview end
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(peek(len(content)-3000, len(content)))"

# Get stats (chars, lines, file count if loaded from directory)
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(stats())"
```

### Step 3: Choose a chunking strategy

- For code: chunk by file/module boundaries or by size
- For logs: chunk by time windows or by size
- For structured data (JSON/YAML): chunk by top-level keys or documents
- For prose/documents: chunk by sections or paragraphs
- Default: chunk by characters (size around chunk_chars)

### Step 4: Materialize chunks as files (so subagents can read them)

```bash
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec <<'PY'
paths = write_chunks('.opencode/rlm_state/chunks', size=200000, overlap=0)
print(f"Created {len(paths)} chunks")
print(paths[:5])
PY
```

### Step 5: Subcall loop (delegate to rlm-subcall)

- For each chunk file, invoke the `@rlm-subcall` subagent with:
  - The user query
  - The chunk file path
  - Any specific extraction instructions
- Keep subagent outputs compact and structured (JSON preferred)
- Store results using the REPL's `add_buffer()` function

### Step 6: Synthesis

- Once enough evidence is collected, synthesize the final answer
- Cite specific evidence (line numbers, positions, identifiers)
- Provide actionable recommendations
- Optional: if gaps remain, run a second pass on specific chunks

## Content-Specific Tips

### For Source Code
```bash
# Find function/class definitions
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(grep(r'(def |class |function |const |export )', max_matches=50))"

# Find imports/dependencies
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(grep(r'(import |require\(|from .+ import)', max_matches=50))"
```

### For Log Files
```bash
# Find error patterns
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(grep(r'(ERROR|FATAL|Exception|WARN)', max_matches=50))"

# Check time range
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(time_range())"
```

### For Structured Data (JSON/YAML/CSV)
```bash
# Extract JSON objects
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(extract_json_objects(max_objects=10))"

# Split YAML documents
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(extract_yaml_documents(max_docs=10))"

# Find keys/fields
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(grep(r'\"[a-zA-Z_]+\":', max_matches=30))"
```

### For Documents (Markdown, Text)
```bash
# Find headings/sections
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(find_lines(r'^#{1,3} ', max_matches=50))"

# Find action items
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec -c "print(grep(r'(TODO|FIXME|ACTION|DECISION)', max_matches=30))"
```

## Recovery Mode

If subagent delegation is unavailable or failing, fall back to iterative Python:

```bash
python3 .opencode/skills/rlm/scripts/rlm_repl.py exec <<'PY'
# Process each chunk directly in the REPL
indices = chunk_indices(size=100000)
for i, (start, end) in enumerate(indices):
    chunk = content[start:end]
    # Run targeted extraction
    errors = [l for l in chunk.splitlines() if 'ERROR' in l or 'Exception' in l]
    if errors:
        add_buffer(f"Chunk {i}: {len(errors)} errors found\n" + "\n".join(errors[:5]))
print(f"Processed {len(indices)} chunks, {len(buffers)} had findings")
PY

# Export results
python3 .opencode/skills/rlm/scripts/rlm_repl.py export-buffers .opencode/rlm_state/findings.txt
```

This is slower (no parallel agents) but works when subagent infrastructure is unavailable.
The key principle: **the REPL is always available as a fallback**.

## Guardrails

- Do not paste large raw chunks into the main chat context
- Use the REPL to locate exact excerpts; quote only what you need
- Subagents cannot spawn other subagents; orchestration stays in root
- Keep scratch/state files under `.opencode/rlm_state/`
- Clean up chunk files after analysis is complete
