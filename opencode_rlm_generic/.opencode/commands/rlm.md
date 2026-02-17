---
description: Run RLM workflow for large context processing
agent: main
---

Load the `rlm` skill to process a large context file using the Recursive Language Model workflow.

$ARGUMENTS

If no arguments were provided, ask for:
1. The path to the context file
2. The query or question about the content

Then follow the RLM skill procedure to:
1. Initialize the REPL with the context
2. Scout and understand the content structure
3. Chunk the content appropriately
4. Delegate chunk analysis to the rlm-subcall subagent
5. Synthesize findings into a clear answer with evidence
