# OpenCode Plugin & Extension System

OpenCode has a comprehensive plugin/extension ecosystem with multiple integration mechanisms.

---

## 1. Plugins (`@opencode-ai/plugin`)

Plugins are JavaScript/TypeScript modules that export plugin functions receiving a context object and returning a hooks object.

### Loading Plugins

| Method | Location |
|--------|----------|
| Local (project) | `.opencode/plugins/` |
| Local (global) | `~/.config/opencode/plugins/` |
| npm | Specified in `opencode.json` via `"plugin": ["package-name"]` |

Plugins are loaded in order: global config → project config → global plugins directory → project plugins directory.

### Basic Plugin Structure

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async ({
  project,
  client,        // OpenCode SDK client
  directory,     // Current working directory
  worktree,      // Git worktree path
  serverUrl,     // Server URL
  $              // Bun shell API
}) => {
  return {
    // Hook implementations go here
  }
}
```

### Available Hooks

| Category | Hook | Description |
|----------|------|-------------|
| **Chat** | `chat.message` | Called when a new message is received |
| | `chat.params` | Modify parameters sent to LLM |
| | `chat.headers` | Modify headers sent to LLM |
| | `experimental.chat.messages.transform` | Transform message history |
| | `experimental.chat.system.transform` | Modify system prompt |
| **Tool** | `tool.execute.before` | Intercept tool execution before running |
| | `tool.execute.after` | Process tool output after execution |
| | `tool.definition` | Modify tool definitions sent to LLM |
| **Permission** | `permission.ask` | Control permission requests |
| **Auth** | `auth` | Authentication hooks (OAuth, API key) |
| **Session** | `event` | Subscribe to all system events |
| | `experimental.session.compacting` | Customize context for session compaction |
| **Shell** | `shell.env` | Inject environment variables |
| | `command.execute.before` | Intercept command execution |

---

## 2. Custom Tools

Define tools that the LLM can call during conversations. Place them in:

- `.opencode/tools/` (project-level)
- `~/.config/opencode/tools/` (global)

The filename becomes the tool name (e.g., `database.ts` → `database` tool).

### Single Tool Per File

```typescript
// .opencode/tools/database.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Query the project database",
  args: {
    query: tool.schema.string().describe("SQL query to execute"),
  },
  async execute(args, context) {
    return `Executed query: ${args.query}`
  },
})
```

### Multiple Tools Per File

Use named exports — they become `<filename>_<exportname>`:

```typescript
// .opencode/tools/math.ts
export const add = tool({ /* ... */ })      // Creates tool "math_add"
export const multiply = tool({ /* ... */ })  // Creates tool "math_multiply"
```

### Tool Context

Tools receive context with: `sessionID`, `messageID`, `agent`, `directory`, `worktree`, `abort` signal, and `metadata()`/`ask()` methods.

---

## 3. MCP (Model Context Protocol) Servers

OpenCode natively supports both local and remote MCP servers for external tool integration.

### Local MCP Server

```json
{
  "mcp": {
    "my-local-mcp": {
      "type": "local",
      "command": ["npx", "-y", "my-mcp-command"],
      "environment": {
        "MY_VAR": "value"
      },
      "enabled": true,
      "timeout": 5000
    }
  }
}
```

### Remote MCP Server

```json
{
  "mcp": {
    "my-remote-mcp": {
      "type": "remote",
      "url": "https://my-mcp-server.com/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer MY_API_KEY"
      }
    }
  }
}
```

### OAuth Support

OpenCode automatically handles OAuth for remote MCP servers via Dynamic Client Registration (RFC 7591):

```json
{
  "oauth": {
    "clientId": "{env:CLIENT_ID}",
    "clientSecret": "{env:CLIENT_SECRET}",
    "scope": "tools:read tools:execute"
  }
}
```

### Tool Management

MCP tools are managed per-agent with glob patterns:

```json
{
  "tools": {
    "my-mcp*": false,
    "my-mcp_search": true
  },
  "agent": {
    "my-agent": {
      "tools": { "my-mcp*": true }
    }
  }
}
```

---

## 4. Skills

Reusable instruction sets discovered on-demand via the `skill` tool. Placed in:

- `.opencode/skills/<name>/SKILL.md` (project-level)
- `~/.config/opencode/skills/<name>/SKILL.md` (global)

### Example Skill

```markdown
---
name: git-release
description: Create consistent releases and changelogs
license: MIT
---

## What I do
- Draft release notes from merged PRs
- Propose a version bump
- Provide a copy-pasteable command
```

---

## 5. Custom Agents

Configure specialized agents in `opencode.json`:

```json
{
  "agent": {
    "my-agent": {
      "prompt": "You are a security specialist...",
      "model": "anthropic/claude-sonnet-4-5",
      "tools": { "bash": false }
    }
  }
}
```

---

## 6. Authentication Hooks

Plugins can provide OAuth and API-based authentication:

```typescript
export const MyAuthPlugin: Plugin = async (ctx) => {
  return {
    auth: {
      provider: "my-provider",
      loader: async (auth, provider) => {
        return { /* auth data */ }
      },
      methods: [
        {
          type: "oauth",
          label: "Login with My Service",
          async authorize(inputs) {
            return {
              url: "https://auth.example.com/authorize",
              instructions: "Click the link above",
              method: "auto",
              callback: async () => ({
                type: "success",
                access: "token",
                refresh: "refresh_token",
                expires: 3600
              })
            }
          }
        }
      ]
    }
  }
}
```

---

## 7. Event System

Plugins can subscribe to comprehensive system events via the `event` hook:

| Category | Events |
|----------|--------|
| **File** | `file.edited`, `file.watcher.updated` |
| **Session** | `session.created`, `session.deleted`, `session.idle`, `session.error`, `session.compacted` |
| **Message** | `message.updated`, `message.removed` |
| **Permission** | `permission.asked`, `permission.replied` |
| **Tool** | `tool.execute.before`, `tool.execute.after` |
| **LSP** | `lsp.updated`, `lsp.client.diagnostics` |
| **Shell** | `shell.env` |

### Example

```typescript
export const NotificationPlugin: Plugin = async ({ $, directory }) => {
  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        await $`notify-send "Session completed"`
      }
    },
  }
}
```

---

## 8. Plugin Dependencies

Local plugins and tools can use external npm packages via a config-level `package.json`:

```json title=".opencode/package.json"
{
  "dependencies": {
    "shescape": "^2.1.0"
  }
}
```

OpenCode runs `bun install` at startup to install these dependencies.

---

## Community Plugins

| Plugin | Purpose |
|--------|---------|
| [opencode-helicone-session](https://github.com/H2Shami/opencode-helicone-session) | Inject Helicone session headers |
| [opencode-type-inject](https://github.com/nick-vi/opencode-type-inject) | Auto-inject TypeScript types |
| [opencode-dynamic-context-pruning](https://github.com/Tarquinen/opencode-dynamic-context-pruning) | Optimize token usage |
| [opencode-devcontainers](https://github.com/athal7/opencode-devcontainers) | Multi-branch isolation |
| [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) | Pre-built tools, agents, LSP |
| [opencode-supermemory](https://github.com/supermemoryai/opencode-supermemory) | Persistent memory across sessions |

Browse more at [awesome-opencode](https://github.com/awesome-opencode/awesome-opencode).

---

## Summary of Extension Points

| Mechanism | Purpose |
|-----------|---------|
| **Plugins** | Hook into lifecycle events, modify LLM behavior |
| **Custom Tools** | Define new LLM-callable capabilities |
| **MCP Servers** | Integrate external protocol-compliant tools |
| **Skills** | Reusable markdown instruction sets |
| **Custom Agents** | Specialized agent profiles |
| **Auth Hooks** | OAuth/API key for custom providers |
| **Event System** | React to system state changes |
