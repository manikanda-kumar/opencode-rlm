# Example Workflows

## 1. Codebase Analysis

```bash
# Dump an entire codebase to a single file
find src/ -name "*.ts" -exec cat {} + > context/codebase.txt

# Analyze with RLM
opencode
/rlm context=./context/codebase.txt query="Map the architecture, identify the main modules, and find code quality issues"
```

## 2. Large Log File Analysis

```bash
# Copy a large log file
cp /var/log/application.log context/app.log

# Analyze with RLM
opencode
/rlm context=./context/app.log query="Find all errors in the last hour and identify the root cause"
```

## 3. Document Summarization

```bash
# Concatenate multiple documents
cat docs/*.md > context/all-docs.txt

# Summarize
opencode
/rlm context=./context/all-docs.txt query="Summarize each document and identify action items"
```

## 4. Data File Analysis

```bash
# Analyze a large CSV or JSON dataset
opencode
/rlm context=./context/dataset.csv query="Describe the schema, find anomalies, and compute basic statistics"
```

## 5. Configuration Review

```bash
# Dump all config files
find . -name "*.yaml" -o -name "*.yml" -o -name "*.toml" -o -name "*.json" | \
  xargs cat > context/all-configs.txt

# Review
opencode
/rlm context=./context/all-configs.txt query="Identify misconfigurations, security issues, and inconsistencies"
```

## 6. API Documentation Review

```bash
# Save API docs or OpenAPI spec
curl -o context/openapi.json https://api.example.com/docs/openapi.json

# Review
opencode
/rlm context=./context/openapi.json query="List all endpoints, identify missing validation, and suggest improvements"
```

## 7. Safe Investigation with Planner

```bash
opencode
# Press Tab to switch to the planner agent
# Then investigate without risk of making changes
/analyze src/auth/
```
