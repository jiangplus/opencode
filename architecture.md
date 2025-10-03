# OpenCode Architecture Overview

This document provides a comprehensive technical overview of the opencode codebase architecture, designed for developers who need to understand the system's internal workings.

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Core Components](#core-components)
3. [Data Flow](#data-flow)
4. [State Management](#state-management)
5. [Extension Points](#extension-points)
6. [Key Subsystems](#key-subsystems)

---

## High-Level Architecture

### System Overview

opencode is an AI coding agent built on a **client-server architecture** where the server handles all AI interactions, state management, and tool execution, while clients (primarily a Go-based TUI) provide the user interface.

```
┌─────────────────────────────────────────────────────┐
│                    Clients                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │   TUI    │  │   Web    │  │  Mobile  │         │
│  │   (Go)   │  │          │  │ (Future) │         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│       │             │              │                │
└───────┼─────────────┼──────────────┼────────────────┘
        │             │              │
        └─────────────┴──────────────┘
                      │
        ┌─────────────▼─────────────────────────┐
        │      HTTP/REST API (Hono)             │
        │        (server/server.ts)             │
        └─────────────┬─────────────────────────┘
                      │
        ┌─────────────▼─────────────────────────┐
        │         Core Server Logic              │
        │  ┌───────────────────────────────┐    │
        │  │  Session Management           │    │
        │  │  Tool Execution               │    │
        │  │  Agent Orchestration          │    │
        │  │  Provider Abstraction         │    │
        │  │  Permission System            │    │
        │  └───────────────────────────────┘    │
        └───────────────────────────────────────┘
```

### Technology Stack

- **Runtime**: Bun (JavaScript/TypeScript)
- **Server Framework**: Hono (lightweight HTTP framework)
- **API Design**: OpenAPI with hono-openapi
- **Schema Validation**: Zod v4
- **LLM Orchestration**: Vercel AI SDK (`ai` package)
- **TUI Client**: Go with custom terminal rendering
- **Storage**: File-based JSON with locking
- **Logging**: Structured logging with custom Log utility

---

## Core Components

### 1. Session System (`packages/opencode/src/session/`)

The session system is the heart of opencode, managing conversations between users and AI agents.

#### Session Structure

```typescript
Session {
  id: string              // Descending ULID
  projectID: string       // Git repository root commit hash
  directory: string       // Working directory
  parentID?: string       // For child sessions (subagents)
  title: string
  version: string         // opencode version
  time: {
    created: number
    updated: number
    compacting?: number   // Last compaction time
  }
  share?: {
    url: string          // Share URL if published
  }
  revert?: {
    messageID: string
    partID?: string
    snapshot?: string    // File snapshot for undo
    diff?: string
  }
}
```

#### Message System (MessageV2)

Messages use a **versioned, part-based architecture** for flexibility:

```typescript
Message {
  id: string
  sessionID: string
  role: "user" | "assistant"
  time: { created, completed? }
  // Assistant-specific fields
  modelID?: string
  providerID?: string
  mode?: string          // Agent mode
  cost?: number
  tokens?: { input, output, reasoning, cache }
  error?: NamedError
  system?: string[]      // System prompt parts
  path?: { cwd, root }
}

Part = TextPart
     | ReasoningPart
     | FilePart
     | ToolPart
     | StepStartPart
     | StepFinishPart
     | SnapshotPart
     | PatchPart
     | AgentPart
```

**Key Design Decisions:**

- **Parts are stored separately** in storage (`storage/part/{messageID}/{partID}.json`)
- **Tool calls are tracked through ToolPart states**: pending → running → completed/error
- **Reasoning traces** from models like Claude are captured in ReasoningPart
- **File attachments** use data URLs stored in FilePart
- **Snapshots** capture file state for undo/revert functionality

#### Session Lifecycle

1. **Creation** (`Session.create`):
   - Generate descending ULID for chronological sorting
   - Associate with project (git root commit or "global")
   - Optionally auto-share based on config

2. **Message Streaming** (`SessionPrompt.prompt`):
   - Queue management prevents concurrent prompts per session
   - Builds context from history, system prompts, and tools
   - Streams LLM responses with real-time tool execution
   - Updates storage atomically with locking

3. **Compaction** (`SessionCompaction`):
   - Summarizes old messages to reduce context size
   - Marks tool outputs as compacted, clearing content
   - Preserves recent messages for continuity

4. **Revert** (`SessionRevert`):
   - Captures file snapshots before edits
   - Stores git diffs for rollback
   - Allows undo of individual tool calls or entire messages

---

### 2. Tool System (`packages/opencode/src/tool/`)

Tools are functions the AI can call to interact with the environment.

#### Tool Architecture

```typescript
Tool.Info {
  id: string
  init: () => Promise<{
    description: string
    parameters: ZodType        // Zod schema for validation
    execute: (args, ctx) => Promise<{
      title: string
      output: string
      metadata: Record<string, any>
    }>
  }>
}

Context {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  callID?: string
  metadata: (input: { title?, metadata? }) => void
}
```

#### Built-in Tools

| Tool | Purpose | Key Features |
|------|---------|-------------|
| **bash** | Execute shell commands | Timeout support, background execution |
| **read** | Read files | Offset/limit for large files, image support |
| **write** | Create files | Must read first for existing files |
| **edit** | Modify files | Exact string replacement, preserve indentation |
| **glob** | File pattern matching | Uses Bun.Glob, sorted by mtime |
| **grep** | Content search | Ripgrep wrapper, regex, context lines |
| **ls** | List directory | Formatted tree view |
| **task** | Launch subagent | Creates child session, streams tool calls |
| **todowrite** | Task tracking | Manages todo list state |
| **webfetch** | Fetch URLs | HTML→markdown, caching |
| **patch** | Apply diffs | Git-style patches |

#### Custom Tools

Tools can be added via:

1. **Config directory** (`.opencode/tool/*.{js,ts}`):
   ```typescript
   export default {
     args: { path: z.string() },
     description: "Custom tool",
     async execute(args, ctx) {
       return "result"
     }
   }
   ```

2. **Plugins** (via `plugin.tool` hook):
   ```typescript
   export default async (input: PluginInput) => ({
     tool: {
       mytool: {
         args: { ... },
         description: "...",
         execute: async (args, ctx) => { ... }
       }
     }
   })
   ```

#### Tool Execution Flow

1. **LLM requests tool** via function calling
2. **Permission check** (via `Permission.ask`)
3. **Tool registry lookup** (`ToolRegistry.tools`)
4. **Parameter validation** (Zod schema)
5. **Plugin `tool.execute.before` hooks**
6. **Tool execution** with abort signal
7. **Plugin `tool.execute.after` hooks**
8. **Result stored in ToolPart** (pending→running→completed)
9. **Result streamed back** to LLM in next step

---

### 3. Agent System (`packages/opencode/src/agent/`)

Agents are **AI configurations** with specific behaviors, permissions, and tool access.

#### Agent Structure

```typescript
Agent.Info {
  name: string
  description?: string
  mode: "subagent" | "primary" | "all"
  builtIn: boolean
  model?: { providerID, modelID }
  prompt?: string              // Custom system prompt
  tools: Record<string, boolean>  // Enable/disable tools
  permission: {
    edit: "allow" | "deny" | "ask"
    bash: Record<string, Permission>  // Per-command patterns
    webfetch?: Permission
  }
  temperature?: number
  topP?: number
  options: Record<string, any>
}
```

#### Built-in Agents

- **build** (primary): Default agent with full permissions
- **plan** (primary): Planning mode, edit denied, bash restricted
- **general** (subagent): For complex searches, no todo tools

#### Custom Agents

Defined in `.opencode/agent/*.md` with frontmatter:

```markdown
---
description: Agent for writing documentation
model: anthropic/claude-sonnet-4
tools:
  bash: false
  webfetch: true
permission:
  edit: allow
  bash:
    "rm *": deny
    "*": ask
temperature: 0.7
---
You are a technical documentation writer...
```

#### Agent Selection

- User explicitly specifies agent (e.g., `@docs` in input)
- Task tool specifies subagent type
- Default to current message's agent (typically "build")

---

### 4. Provider System (`packages/opencode/src/provider/`)

Provides **multi-LLM support** through a unified abstraction layer.

#### Provider Architecture

```typescript
Provider {
  id: string
  npm: string           // NPM package name
  name: string
  env: string[]         // Environment variable names for API key
  api?: string          // Base URL
  models: Record<string, Model>
}

Model {
  id: string
  name: string
  cost: { input, output, cache_read, cache_write }  // per 1M tokens
  limit: { context, output }
  temperature: boolean
  tool_call: boolean
  reasoning: boolean
  attachment: boolean
  options: Record<string, any>
}
```

#### Provider Sources (Priority Order)

1. **Environment variables** (highest priority)
2. **API keys** (via `Auth.all()`)
3. **Custom loaders** (`CUSTOM_LOADERS`)
4. **Config file** (`opencode.json`)

#### Custom Loaders

Special initialization logic per provider:

- **anthropic**: Adds beta headers for Claude Code features
- **opencode**: Filters models based on API key presence
- **openai/azure**: Uses `responses()` API for structured outputs
- **amazon-bedrock**: AWS credential chain, region-specific model IDs
- **openrouter**: Adds referrer headers

#### Model Selection

```typescript
// Default model priority
priority = ["gemini-2.5-pro-preview", "gpt-5", "claude-sonnet-4"]

// Fallback chain
1. Config `model` field
2. Last used model from TUI state
3. Highest priority available model
```

#### Provider Integration

Uses **Vercel AI SDK** (`ai` package) for:
- Unified `languageModel()` interface
- Streaming with `streamText()`
- Structured outputs with `generateObject()`
- Tool calling abstraction
- Token counting and cost tracking

---

### 5. Project & Instance System (`packages/opencode/src/project/`)

Manages **project context** and **instance-scoped state**.

#### Project

A project represents a git repository or directory:

```typescript
Project.Info {
  id: string          // Git root commit hash or "global"
  worktree: string    // Git worktree root
  vcs?: "git"
  time: {
    created: number
    initialized?: number
  }
}
```

**Project Detection:**
1. Search upward for `.git` directory
2. Find root commit: `git rev-list --max-parents=0 --all`
3. Use commit hash as project ID
4. Fall back to "global" project if no git

#### Instance

Provides **scoped context** for the current working directory:

```typescript
Instance {
  directory: string    // Current working directory
  worktree: string     // Git worktree root
  project: Project.Info
}
```

**Instance.provide()**: Async context wrapper
```typescript
await Instance.provide({
  directory: "/path/to/code",
  init: async () => { /* bootstrap */ },
  fn: async () => {
    // Code here has access to Instance.directory, Instance.project
    const cfg = await Config.get()  // Config loaded for this instance
  }
})
```

**Instance.state()**: Cached state factory
```typescript
const myState = Instance.state(async () => {
  // Expensive initialization
  return { data: await loadData() }
})

// Later: state is cached per instance
const state = await myState()
```

**Use Cases:**
- Config loading (searches from instance directory upward)
- LSP servers (one per project)
- Plugin loading (instance-specific)
- Tool registry (custom tools per instance)

---

### 6. Configuration System (`packages/opencode/src/config/`)

Hierarchical configuration with **merging and overrides**.

#### Config Hierarchy (Low to High Priority)

1. **Global config** (`~/.config/opencode/`)
2. **Project configs** (`.opencode/` directories, root → worktree)
3. **Config files** (`opencode.json`/`opencode.jsonc`, searched upward)
4. **Custom config file** (`OPENCODE_CONFIG` environment variable)
5. **Config content** (`OPENCODE_CONFIG_CONTENT` JSON string)
6. **Well-known providers** (fetched from provider's `/.well-known/opencode`)

#### Config Schema

```typescript
Config {
  model?: string                    // Default model (provider/model-id)
  small_model?: string              // For lightweight tasks
  username?: string
  share?: "auto" | "disabled"
  permission?: PermissionConfig
  tools?: Record<string, boolean>   // Global tool enable/disable
  provider?: Record<string, {       // Provider configuration
    npm?: string
    name?: string
    env?: string[]
    api?: string
    models?: Record<string, ModelConfig>
    options?: Record<string, any>
  }>
  agent?: Record<string, AgentConfig>
  mode?: Record<string, AgentConfig>  // Deprecated, migrated to agent
  command?: Record<string, string>    // Custom slash commands
  plugin?: string[]                   // Plugin paths or npm packages
  lsp?: Record<string, LSPConfig>
  mcp?: Record<string, MCPConfig>
  disabled_providers?: string[]
  keybinds?: Record<string, string>
}
```

#### Config Loading Locations

- **Global**: `~/.config/opencode/opencode.json`
- **Project**: `.opencode/opencode.json` (any level from root to cwd)
- **Agents**: `.opencode/agent/*.md` (frontmatter)
- **Commands**: `.opencode/command/*.md`
- **Tools**: `.opencode/tool/*.{js,ts}`
- **Plugins**: Listed in `plugin` array

#### Special Config Files

**Agent Definitions** (`.opencode/agent/docs.md`):
```markdown
---
description: ALWAYS use this when writing docs
---
You are an expert technical documentation writer...
```

**Command Definitions** (`.opencode/command/commit.md`):
```markdown
commit and push

make sure it includes a prefix like
docs:, core:, tui:, ci:
```

---

## Data Flow

### User Message Flow

```
┌────────────────────────────────────────────────────────────┐
│ 1. User Input                                              │
│    ├─ Text                                                 │
│    ├─ Files (attachments)                                  │
│    └─ Agent selection (@agent-name)                        │
└─────────────────┬──────────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────────────┐
│ 2. Session.create() or Session.touch()                    │
│    └─ Creates/updates session in storage                  │
└─────────────────┬──────────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────────────┐
│ 3. SessionPrompt.prompt()                                  │
│    ├─ Queue management (one prompt per session)           │
│    ├─ Create user message with parts                      │
│    └─ Store in storage/message/{sessionID}/{messageID}    │
└─────────────────┬──────────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────────────┐
│ 4. Build Context                                           │
│    ├─ Load session history                                │
│    ├─ Filter summarized messages                          │
│    ├─ Build system prompt (SystemPrompt.build)            │
│    │   ├─ Provider header                                 │
│    │   ├─ Agent prompt                                    │
│    │   ├─ Tool descriptions                               │
│    │   ├─ MCP tool descriptions                           │
│    │   └─ Project context (CLAUDE.md, git status, etc)    │
│    └─ Convert to ModelMessage[] format                    │
└─────────────────┬──────────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────────────┐
│ 5. Provider.getModel()                                     │
│    ├─ Select provider & model                             │
│    ├─ Load SDK (via BunProc.install if needed)            │
│    └─ Apply provider options & custom loaders             │
└─────────────────┬──────────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────────────┐
│ 6. ToolRegistry.tools()                                    │
│    ├─ Load built-in tools                                 │
│    ├─ Load custom tools from config                       │
│    ├─ Load plugin tools                                   │
│    ├─ Load MCP tools                                      │
│    ├─ Filter by agent permissions                         │
│    └─ Convert to AI SDK tool format                       │
└─────────────────┬──────────────────────────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────────────┐
│ 7. streamText() - LLM Invocation                          │
│    ├─ Provider wrappers (ProviderTransform)               │
│    ├─ Plugin hooks (chat.params)                          │
│    ├─ Temperature, topP from agent config                 │
│    └─ Max output tokens (32k)                             │
└─────────────────┬──────────────────────────────────────────┘
                  │
          ┌───────┴────────┐
          │                │
          ▼                ▼
┌──────────────────┐  ┌─────────────────┐
│ 8a. Text Stream  │  │ 8b. Tool Calls  │
│  └─ TextPart     │  │  └─ ToolPart    │
└────────┬─────────┘  └────────┬────────┘
         │                     │
         │                     ▼
         │            ┌────────────────────────────┐
         │            │ 9. Tool Execution          │
         │            │  ├─ Permission.ask()       │
         │            │  ├─ Plugin hooks (before)  │
         │            │  ├─ Tool.execute()         │
         │            │  ├─ Plugin hooks (after)   │
         │            │  └─ Update ToolPart state  │
         │            └────────┬───────────────────┘
         │                     │
         │                     ▼
         │            ┌────────────────────────────┐
         │            │ 10. Tool Result            │
         │            │   └─ Fed back to LLM       │
         │            │      in next step          │
         │            └────────┬───────────────────┘
         │                     │
         └─────────────────────┘
                  │
                  ▼
┌────────────────────────────────────────────────────────────┐
│ 11. Assistant Message Complete                             │
│     ├─ Calculate cost (tokens × model rates)               │
│     ├─ Store StepFinishPart with token counts             │
│     ├─ Update message.time.completed                       │
│     └─ Bus.publish(MessageV2.Event.Updated)               │
└────────────────────────────────────────────────────────────┘
```

### Storage Flow

```
Instance.directory
       │
       ▼
┌─────────────────────────────┐
│ Storage.state()             │
│  $XDG_DATA_HOME/opencode/   │
│  storage/                   │
└─────────────────┬───────────┘
                  │
         ┌────────┴─────────┐
         │                  │
         ▼                  ▼
┌─────────────────┐  ┌─────────────────┐
│ project/        │  │ session/        │
│   {id}.json     │  │   {proj}/       │
└─────────────────┘  │     {sess}.json │
                     └─────────────────┘
                            │
                   ┌────────┴────────┐
                   │                 │
                   ▼                 ▼
         ┌─────────────────┐  ┌─────────────────┐
         │ message/        │  │ part/           │
         │   {sess}/       │  │   {msg}/        │
         │     {msg}.json  │  │     {part}.json │
         └─────────────────┘  └─────────────────┘
```

**Storage Operations:**
- **Write**: Atomic with lock, JSON pretty-printed
- **Read**: Lock-free, cached by Bun
- **Update**: Read-modify-write with lock
- **List**: Glob scan with sorting

---

## State Management

### Instance-Scoped State

Uses `Instance.state()` for **per-directory cached state**:

```typescript
// Definition (runs once per instance)
const myState = Instance.state(async () => {
  console.log("Expensive initialization")
  return { data: await loadData() }
}, async (state) => {
  // Cleanup on Instance.dispose()
  await state.cleanup()
})

// Usage (returns cached result)
const state = await myState()
```

**Examples:**
- `Config.state`: Loaded config with merged hierarchy
- `Agent.state`: Agent definitions from config
- `Provider.state`: Available providers and models
- `ToolRegistry.state`: Custom tools
- `LSP.state`: LSP clients
- `MCP.state`: MCP clients
- `Plugin.state`: Loaded plugins

**Lifecycle:**
- Initialized on first access within `Instance.provide()`
- Cached in `State` map keyed by instance directory
- Disposed when instance changes or server restarts

### Event Bus

**Pub-sub system** for component communication (`packages/opencode/src/bus/`):

```typescript
// Define event
const MyEvent = Bus.event("my.event", z.object({
  data: z.string()
}))

// Subscribe
Bus.subscribe(MyEvent, (evt) => {
  console.log(evt.properties.data)
})

// Publish
Bus.publish(MyEvent, { data: "hello" })
```

**Key Events:**
- `session.updated`: Session metadata changed
- `session.deleted`: Session removed
- `session.error`: Error occurred in session
- `message.updated`: Message metadata changed
- `message.part.updated`: Part added/modified
- `permission.updated`: Permission request created
- `permission.replied`: User responded to permission

**Event Flow:**
- Events are **instance-scoped** (separate bus per instance)
- **Synchronous** within instance (await all handlers)
- Used for: plugin hooks, share sync, UI updates

---

## Extension Points

### 1. Plugins (`packages/plugin/`)

Plugins extend opencode via hooks:

```typescript
export default async (input: PluginInput): Promise<Hooks> => ({
  // Called on every event
  event: async ({ event }) => {
    if (event.type === "session.updated") { ... }
  },

  // Modify config before use
  config: async (config) => {
    config.model = "my-provider/my-model"
  },

  // Add custom tools
  tool: {
    mytool: {
      args: z.object({ path: z.string() }),
      description: "My custom tool",
      execute: async (args, ctx) => "result"
    }
  },

  // Custom auth methods
  auth: {
    provider: "my-provider",
    methods: [
      {
        type: "oauth",
        label: "Login with OAuth",
        authorize: async () => ({ ... })
      }
    ],
    loader: async (auth, provider) => ({
      apiKey: (await auth()).key
    })
  },

  // Intercept/modify messages
  "chat.message": async (input, output) => {
    // Modify output.message or output.parts
  },

  // Modify LLM parameters
  "chat.params": async (input, output) => {
    output.temperature = 0.5
  },

  // Modify permission responses
  "permission.ask": async (input, output) => {
    if (input.type === "edit") output.status = "allow"
  },

  // Intercept tool execution
  "tool.execute.before": async (input, output) => {
    // Modify output.args
  },

  "tool.execute.after": async (input, output) => {
    // Modify output.title, output.output, output.metadata
  }
})
```

**Plugin Loading:**
1. Listed in `config.plugin` array
2. Can be npm packages (`my-plugin@1.0.0`) or local paths (`file:///path/to/plugin`)
3. Installed via `BunProc.install()` if needed
4. Loaded once per instance
5. Default plugins: `opencode-copilot-auth`, `opencode-anthropic-auth`

### 2. Custom Tools

**Via Config Directory:**
```typescript
// .opencode/tool/deploy.ts
import z from "zod"

export default {
  args: {
    environment: z.enum(["staging", "production"])
  },
  description: "Deploy application",
  async execute(args, ctx) {
    await ctx.$.nothrow`./deploy.sh ${args.environment}`
    return `Deployed to ${args.environment}`
  }
}
```

**Via Plugin:**
```typescript
export default async (input) => ({
  tool: {
    "my-tool": { ... }
  }
})
```

### 3. Custom Agents

**Markdown Files** (`.opencode/agent/my-agent.md`):
```markdown
---
description: My custom agent
model: anthropic/claude-sonnet-4
tools:
  bash: true
  webfetch: false
permission:
  edit: allow
  bash:
    "rm *": deny
    "*": ask
temperature: 0.7
---
You are my custom agent with special instructions...
```

**Via Config:**
```json
{
  "agent": {
    "my-agent": {
      "description": "My custom agent",
      "model": "anthropic/claude-sonnet-4",
      "tools": { "bash": true },
      "permission": { "edit": "allow" }
    }
  }
}
```

### 4. Custom Commands

**Slash Commands** (`.opencode/command/deploy.md`):
```markdown
deploy the application to production
```

Invoked via `/deploy` in TUI.

### 5. LSP Servers

**Custom LSP Configuration:**
```json
{
  "lsp": {
    "my-lsp": {
      "command": ["my-lsp-server", "--stdio"],
      "extensions": [".my"],
      "env": {
        "MY_LSP_CONFIG": "/path/to/config"
      },
      "initialization": {
        "customOption": true
      }
    }
  }
}
```

### 6. MCP Servers

**Model Context Protocol** integration:

```json
{
  "mcp": {
    "my-mcp": {
      "type": "local",
      "command": ["node", "/path/to/mcp-server"],
      "enabled": true
    },
    "remote-mcp": {
      "type": "remote",
      "url": "https://mcp.example.com",
      "headers": {
        "Authorization": "Bearer token"
      }
    }
  }
}
```

---

## Key Subsystems

### 1. Permission System (`packages/opencode/src/permission/`)

**Async permission workflow** with wildcard matching:

```typescript
// Request permission
await Permission.ask({
  sessionID,
  messageID,
  callID,
  type: "bash",
  pattern: "rm *",  // or ["rm *", "del *"]
  title: "Delete files",
  metadata: { command: "rm *.tmp" }
})

// User responds (from TUI)
Permission.respond({
  sessionID,
  permissionID,
  response: "once" | "always" | "reject"
})
```

**Permission Matching:**
- **Wildcard patterns**: `*` matches anything
- **Always approval**: Caches pattern for session
- **Granular bash permissions**: Per-command patterns in config

**Use Cases:**
- Bash tool: Check against command pattern
- Edit tool: Ask before modifying files
- WebFetch tool: Ask before accessing URLs

### 2. LSP Integration (`packages/opencode/src/lsp/`)

**Language Server Protocol** for code intelligence:

```typescript
// Start LSP client for file
const client = await LSP.getClient("/path/to/file.ts")

// Get diagnostics
const diagnostics = await client.diagnostics()

// Get hover info
const hover = await client.hover("/path/to/file.ts", { line: 10, character: 5 })

// Document symbols
const symbols = await client.documentSymbols("/path/to/file.ts")
```

**Built-in LSP Servers:**
- TypeScript: `typescript-language-server`
- Python: `pylsp`
- Rust: `rust-analyzer`
- Go: `gopls`
- More can be configured

**Lifecycle:**
- One LSP client per (server, root) pair
- Spawned on first file access
- Cached in `LSP.state()`
- Disposed on instance cleanup

### 3. MCP Integration (`packages/opencode/src/mcp/`)

**Model Context Protocol** for external tools:

- MCP tools are **merged into tool registry**
- Appear as regular tools to the AI
- Support local (stdio) and remote (HTTP/SSE) servers
- Auto-reconnect on failure

### 4. Share System (`packages/opencode/src/share/`)

**Public session sharing**:

```typescript
// Create share
const share = await Session.share(sessionID)
// => { url: "https://opencode.ai/share/...", secret: "..." }

// Auto-sync to cloud
Bus.subscribe(Session.Event.Updated, (evt) => {
  Share.sync("session/info/" + evt.properties.info.id, evt.properties.info)
})
```

**Architecture:**
- Share links are **public read-only**
- Changes synced via `Share.sync()` (queued, atomic)
- Stored in `storage/share/{sessionID}.json`
- Cloud backend at `https://api.opencode.ai` (or `api.dev.opencode.ai`)

### 5. Compaction System (`packages/opencode/src/session/compaction.ts`)

**Context window management**:

- Triggered when context approaches model limit
- Summarizes old messages using small model (Haiku/Flash)
- Marks tool outputs as compacted (content cleared)
- Preserves recent messages for continuity
- Updates `session.time.compacting`

**Compaction Flow:**
1. Calculate token count for history
2. If > threshold, find split point (keep recent 20% of messages)
3. Generate summary of old messages
4. Store summary as synthetic message
5. Mark `message.summary = true`
6. Clear old tool outputs (`time.compacted = now`)

### 6. Revert System (`packages/opencode/src/session/revert.ts`)

**Undo functionality**:

```typescript
// Capture snapshot before edit
await SessionRevert.capture(sessionID, messageID, partID, files)

// Revert to snapshot
await SessionRevert.revert(sessionID, revertPoint)
```

**Revert Points:**
- **Snapshot**: Full file content before edit
- **Diff**: Git diff for rollback
- Stored in `session.revert`

### 7. Logging System (`packages/opencode/src/util/log.ts`)

**Structured logging** with context:

```typescript
const log = Log.create({ service: "my-service" })
log.info("operation", { key: "value" })
log.error("failure", { error: e })

// With tags
const tagged = log.clone().tag("session", sessionID)
tagged.debug("debug info")
```

**Log Levels**: DEBUG, INFO, WARN, ERROR

**Output:**
- Stderr if `--print-logs` flag
- Always to file at `Log.file()` (`$XDG_STATE_HOME/opencode/logs/`)

---

## Performance Considerations

### 1. Caching Strategies

- **Instance state**: Cached per directory, disposed on change
- **Provider SDKs**: Cached by (package, options) hash
- **LSP clients**: One per (server, root), reused
- **MCP clients**: Persistent connections
- **WebFetch**: 15-minute self-cleaning cache

### 2. Lazy Loading

- **Bun APIs**: Dynamic imports for heavy dependencies
- **BunProc.install()**: Downloads npm packages on-demand
- **Provider SDKs**: Loaded when model first used
- **LSP servers**: Spawned on first file access

### 3. Streaming

- **LLM responses**: Streamed via Server-Sent Events
- **Tool execution**: Results streamed incrementally
- **File reads**: Offset/limit for large files
- **Storage reads**: Lock-free for high throughput

### 4. Concurrency

- **One prompt per session**: Queued to prevent race conditions
- **Storage locks**: Write locks, lock-free reads
- **Bus events**: Async, all handlers awaited
- **Tool execution**: Parallel within step (multiple tool calls)

---

## Testing

### Test Structure

Tests use **Bun's built-in test runner**:

```typescript
import { describe, it, expect } from "bun:test"

describe("MyComponent", () => {
  it("should work", () => {
    expect(myFunction()).toBe(true)
  })
})
```

**Run tests:**
```bash
cd packages/opencode && bun test
```

### Key Test Areas

- Session lifecycle
- Message serialization
- Tool execution
- Permission system
- Config merging
- Provider selection

---

## Debugging

### 1. Enable Debug Logs

```bash
opencode --print-logs --log-level=DEBUG
```

### 2. Inspect Storage

```bash
# Session data
cat ~/.local/share/opencode/storage/session/{projectID}/{sessionID}.json

# Messages
cat ~/.local/share/opencode/storage/message/{sessionID}/{messageID}.json

# Parts
ls ~/.local/share/opencode/storage/part/{messageID}/
```

### 3. Monitor Events

Add logging to event bus:
```typescript
Bus.subscribeAll((event) => {
  console.log(event.type, event.properties)
})
```

### 4. Check Logs

```bash
tail -f $(opencode --print-logs 2>&1 | grep "log file" | cut -d' ' -f4)
```

---

## Security Considerations

### 1. Permission System

- **User approval** required for destructive operations
- **Wildcard matching** for pattern-based permissions
- **Session-scoped** approvals (not persisted across sessions)

### 2. Tool Execution

- **Abort signals** for cancellation
- **Timeout limits** on bash commands (default 2 minutes)
- **Environment isolation** (uses process.env, no sudo)

### 3. File Operations

- **Working directory restriction** (cannot escape git worktree)
- **Read before write** enforcement (prevents accidental overwrites)
- **Lock-based writes** (prevents concurrent modification)

### 4. API Keys

- **Environment variables** (not logged)
- **Config files** (user-managed, .gitignore)
- **Well-known endpoints** (fetched over HTTPS)
- **Plugin isolation** (separate process, limited SDK access)

---

## Future Architecture Considerations

### 1. Remote Server Mode

- Server can run on remote machine
- Clients connect over HTTP/WebSocket
- Authentication layer needed

### 2. Multi-User Support

- Project-level isolation
- Session ownership
- Collaborative editing

### 3. Persistent Agent State

- Agents maintain memory across sessions
- Long-term context storage
- Embeddings for retrieval

### 4. Distributed Tool Execution

- Tools run in sandboxed containers
- Remote tool execution
- Resource limits

---

## Conclusion

opencode's architecture is designed for:

- **Extensibility**: Plugins, custom tools, agents, and commands
- **Multi-LLM Support**: Provider abstraction with custom loaders
- **Flexibility**: Client-server separation, multiple clients
- **Performance**: Caching, lazy loading, streaming
- **Safety**: Permission system, file protection, abort signals

The modular design allows developers to extend any part of the system without modifying core code, while the instance-scoped state management ensures clean isolation between projects.

For more details on specific components, refer to the inline code documentation and the CLAUDE.md file.