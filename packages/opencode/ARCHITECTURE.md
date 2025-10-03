# opencode Package Architecture

**Package**: `packages/opencode`
**Purpose**: Core AI agent server, API, and business logic
**Runtime**: Bun + TypeScript
**Entry**: `src/index.ts` (CLI), `bin/opencode` (binary)

---

## Overview

The `opencode` package is the **heart of the system**, containing all server-side logic for:
- AI agent orchestration
- Session and message management
- Tool execution
- Multi-provider LLM integration
- Permission system
- Configuration management
- Storage layer

This package exposes both a **CLI interface** (via yargs) and an **HTTP API** (via Hono) for clients to interact with.

---

## Package Structure

```
packages/opencode/
├── src/
│   ├── agent/          # Agent configuration and management
│   ├── auth/           # Authentication and API key management
│   ├── bus/            # Event bus for pub-sub communication
│   ├── bun/            # Bun-specific utilities (npm install)
│   ├── cli/            # CLI commands and UI
│   ├── command/        # Custom slash command system
│   ├── config/         # Hierarchical configuration
│   ├── file/           # File operations (ripgrep, time, diff)
│   ├── flag/           # Feature flags and environment variables
│   ├── format/         # Output formatting utilities
│   ├── global/         # Global paths (state, config, data)
│   ├── id/             # ULID generation (ascending/descending)
│   ├── ide/            # IDE integration utilities
│   ├── installation/   # Version and installation detection
│   ├── lsp/            # Language Server Protocol integration
│   ├── mcp/            # Model Context Protocol integration
│   ├── permission/     # Permission request/response system
│   ├── plugin/         # Plugin loading and hook execution
│   ├── project/        # Project and instance management
│   ├── provider/       # LLM provider abstraction
│   ├── server/         # HTTP API server (Hono)
│   ├── session/        # Session and message management
│   ├── share/          # Session sharing to cloud
│   ├── snapshot/       # File snapshot for undo
│   ├── storage/        # File-based storage layer
│   ├── tool/           # Built-in and custom tools
│   └── util/           # Utilities (log, error, filesystem)
├── script/
│   ├── build.ts        # Build script
│   ├── publish.ts      # Publishing script
│   └── schema.ts       # OpenAPI schema generation
└── bin/
    └── opencode        # CLI binary entry point
```

---

## Module Breakdown

### 1. Agent (`src/agent/`)

**Purpose**: Manage agent configurations (system prompts, permissions, tool access)

**Key Files**:
- `agent.ts`: Agent definition, loading from config, generation

**Data Structures**:
```typescript
Agent.Info {
  name: string
  description?: string
  mode: "subagent" | "primary" | "all"
  builtIn: boolean
  model?: { providerID, modelID }
  prompt?: string
  tools: Record<string, boolean>
  permission: {
    edit: Permission
    bash: Record<string, Permission>
    webfetch?: Permission
  }
  temperature?: number
  topP?: number
  options: Record<string, any>
}
```

**Built-in Agents**:
- `build`: Default agent, full permissions
- `plan`: Planning mode, no edits
- `general`: Subagent for searches

**Loading Sources**:
1. `.opencode/agent/*.md` files (frontmatter + prompt)
2. `config.agent` object
3. `config.mode` (deprecated, migrated to agent)

**Agent Generation**: Can auto-generate agent configs via `Agent.generate()` using LLM

---

### 2. Auth (`src/auth/`)

**Purpose**: Manage API keys and authentication for providers

**Key Features**:
- Store API keys in encrypted storage
- Support OAuth flows via plugins
- Well-known provider endpoints (`/.well-known/opencode`)

**API Key Types**:
- `api`: Simple API key
- `oauth`: OAuth tokens (access + refresh)
- `wellknown`: Fetched from provider endpoint

---

### 3. Bus (`src/bus/`)

**Purpose**: Event-driven pub-sub system for component communication

**Architecture**:
```typescript
// Define event
const MyEvent = Bus.event("my.event", z.object({ data: z.string() }))

// Subscribe
Bus.subscribe(MyEvent, (evt) => { ... })

// Publish
Bus.publish(MyEvent, { data: "value" })
```

**Key Events**:
- `session.updated`, `session.deleted`, `session.error`
- `message.updated`, `message.part.updated`
- `permission.updated`, `permission.replied`
- `server.connected`

**Scoping**: Events are **instance-scoped** (separate bus per `Instance.directory`)

---

### 4. CLI (`src/cli/`)

**Purpose**: Command-line interface and user interaction

**Commands** (`src/cli/cmd/`):
- `run`: Run a single prompt
- `generate`: Generate code/files
- `tui`: Launch TUI client
- `attach`: Attach to existing session
- `serve`: Start HTTP server
- `auth`: Manage authentication
- `agent`: Manage agents
- `upgrade`: Self-update
- `models`: List available models
- `stats`: Show usage statistics
- `export`: Export session data
- `github`: GitHub integration
- `debug`: Debug utilities
- `mcp`: MCP server management

**UI Utilities** (`src/cli/ui.ts`):
- Formatted output (logo, errors, success)
- Interactive prompts (via @clack/prompts)

---

### 5. Config (`src/config/`)

**Purpose**: Hierarchical configuration system with merging

**Loading Order** (low to high priority):
1. Global config (`~/.config/opencode/`)
2. Project `.opencode/` directories (root → cwd)
3. `opencode.json`/`opencode.jsonc` files (searched upward)
4. `OPENCODE_CONFIG` file path
5. `OPENCODE_CONFIG_CONTENT` JSON string
6. Well-known provider configs

**Key Files**:
- `config.ts`: Main config loading and merging
- `markdown.ts`: Parse markdown config files (agents, commands)

**Config Schema**:
```typescript
Config.Info {
  model?: string
  small_model?: string
  username?: string
  share?: "auto" | "disabled"
  permission?: PermissionConfig
  tools?: Record<string, boolean>
  provider?: Record<string, ProviderConfig>
  agent?: Record<string, AgentConfig>
  command?: Record<string, string>
  plugin?: string[]
  lsp?: Record<string, LSPConfig>
  mcp?: Record<string, MCPConfig>
  disabled_providers?: string[]
  keybinds?: Record<string, string>
}
```

**Special Files**:
- `.opencode/agent/*.md`: Agent definitions
- `.opencode/command/*.md`: Slash commands
- `.opencode/tool/*.{js,ts}`: Custom tools

---

### 6. File (`src/file/`)

**Purpose**: File system operations and utilities

**Modules**:
- `index.ts`: File reading, directory traversal
- `ripgrep.ts`: Fast content search via ripgrep
- `time.ts`: File modification time tracking
- `diff.ts`: Git diff generation

**Key Features**:
- Respect `.gitignore` patterns
- Efficient file scanning with Bun.Glob
- Integration with git for diffs

---

### 7. LSP (`src/lsp/`)

**Purpose**: Language Server Protocol integration for code intelligence

**Architecture**:
```
LSP.state() → {
  servers: Record<string, LSPServer.Info>
  clients: LSPClient.Info[]
  broken: Set<string>
}
```

**LSP Workflow**:
1. File accessed (e.g., via read tool)
2. Determine LSP server by extension
3. Find or spawn LSP client
4. Send LSP requests (diagnostics, hover, symbols)
5. Cache client for reuse

**Built-in Servers**:
- TypeScript, Python, Rust, Go, etc.
- Configurable via `config.lsp`

**Key Files**:
- `index.ts`: Server registry and client management
- `client.ts`: LSP client (JSON-RPC over stdio)
- `server.ts`: Built-in server definitions

---

### 8. MCP (`src/mcp/`)

**Purpose**: Model Context Protocol integration for external tools

**MCP Server Types**:
- **Local**: Spawned via command (stdio transport)
- **Remote**: HTTP/SSE connection

**Integration**:
- MCP tools merged into tool registry
- Appear as regular tools to AI
- Auto-reconnect on failure

**Configuration**:
```json
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ["node", "server.js"]
    }
  }
}
```

---

### 9. Permission (`src/permission/`)

**Purpose**: User approval system for sensitive operations

**Permission Flow**:
1. Tool requests permission via `Permission.ask()`
2. Permission request published to event bus
3. Client (TUI) shows prompt to user
4. User responds: `once`, `always`, or `reject`
5. Permission resolved or rejected

**Wildcard Matching**:
- Patterns like `rm *` match similar commands
- `always` response caches pattern for session

**Permission Types**:
- `edit`: File modifications
- `bash`: Shell command execution (with patterns)
- `webfetch`: Web requests

---

### 10. Plugin (`src/plugin/`)

**Purpose**: Load and execute plugin hooks

**Plugin Lifecycle**:
1. Load plugins from config (`config.plugin`)
2. Install via `BunProc.install()` if needed
3. Import and initialize
4. Execute hooks on events

**Hook Types**:
- `event`: All events
- `config`: Modify config
- `tool`: Add custom tools
- `auth`: Custom auth providers
- `chat.message`: Intercept messages
- `chat.params`: Modify LLM params
- `permission.ask`: Override permissions
- `tool.execute.before/after`: Intercept tool execution

**Default Plugins**:
- `opencode-copilot-auth`: GitHub Copilot auth
- `opencode-anthropic-auth`: Anthropic auth

---

### 11. Project (`src/project/`)

**Purpose**: Project detection and instance management

**Project Detection**:
1. Search for `.git` directory upward
2. Get root commit hash: `git rev-list --max-parents=0 --all`
3. Use commit hash as project ID
4. Fall back to "global" if no git

**Instance**:
- Provides scoped context for current directory
- `Instance.provide()`: Async context wrapper
- `Instance.state()`: Cached state factory
- `Instance.directory`, `Instance.worktree`, `Instance.project`

**Key Files**:
- `project.ts`: Project detection
- `instance.ts`: Instance context management
- `state.ts`: State caching per instance
- `bootstrap.ts`: Instance initialization

---

### 12. Provider (`src/provider/`)

**Purpose**: Multi-LLM provider abstraction

**Provider Architecture**:
```typescript
Provider {
  id: string
  npm: string
  name: string
  env: string[]
  api?: string
  models: Record<string, Model>
}
```

**Loading Sources** (priority):
1. Environment variables (e.g., `ANTHROPIC_API_KEY`)
2. API keys via `Auth.all()`
3. Custom loaders
4. Config file

**Custom Loaders**:
- `anthropic`: Add beta headers
- `opencode`: Filter models by API key
- `openai/azure`: Use `responses()` API
- `amazon-bedrock`: AWS credentials, region logic
- `openrouter`: Add referrer headers

**Model Selection**:
- Priority: `["gemini-2.5-pro-preview", "gpt-5", "claude-sonnet-4"]`
- Fallback: Last used model from TUI state
- Default: Highest priority available

**Key Files**:
- `provider.ts`: Provider loading and model access
- `models.ts`: Model database
- `transform.ts`: Provider-specific transformations

---

### 13. Server (`src/server/`)

**Purpose**: HTTP API server for client communication

**Framework**: Hono (lightweight, fast)

**API Design**: OpenAPI with `hono-openapi`

**Key Endpoints**:
- `/config`: Get/update config
- `/session`: CRUD operations
- `/session/:id/message`: Message management
- `/session/:id/prompt`: Send prompt (SSE streaming)
- `/project`: Project info
- `/provider`: List providers and models
- `/tool`: List tools
- `/permission`: Permission management
- `/share`: Session sharing
- `/log`: Streaming logs (SSE)

**Middleware**:
- CORS enabled
- Request logging
- Instance provisioning per request
- Error handling (NamedError → JSON)

**Key Files**:
- `server.ts`: Main API server
- `project.ts`: Project-specific routes
- `tui.ts`: TUI-specific routes (direct function calls)

---

### 14. Session (`src/session/`)

**Purpose**: Conversation management and message orchestration

**Session Structure**:
```typescript
Session.Info {
  id: string
  projectID: string
  directory: string
  parentID?: string
  title: string
  version: string
  time: { created, updated, compacting? }
  share?: { url }
  revert?: { messageID, partID, snapshot, diff }
}
```

**Message System** (MessageV2):
- **Part-based architecture**: Messages composed of parts
- **Tool states**: pending → running → completed/error
- **Reasoning traces**: Captured from models
- **Snapshots**: File state for undo

**Session Lifecycle**:
1. Create session
2. User sends message
3. Build context (history, system prompt, tools)
4. Stream LLM response
5. Execute tools in real-time
6. Store results
7. Optionally compact old messages

**Key Files**:
- `index.ts`: Session CRUD operations
- `message-v2.ts`: Message and part definitions
- `prompt.ts`: Prompt building and LLM orchestration
- `compaction.ts`: Context window management
- `revert.ts`: Undo functionality
- `system.ts`: System prompt building

---

### 15. Storage (`src/storage/`)

**Purpose**: File-based storage with locking

**Storage Layout**:
```
$XDG_DATA_HOME/opencode/storage/
├── project/{id}.json
├── session/{projectID}/{sessionID}.json
├── message/{sessionID}/{messageID}.json
├── part/{messageID}/{partID}.json
└── share/{sessionID}.json
```

**Operations**:
- `Storage.write()`: Atomic write with lock
- `Storage.read()`: Lock-free read
- `Storage.update()`: Read-modify-write with lock
- `Storage.list()`: Glob scan
- `Storage.remove()`: Delete file

**Migrations**:
- Run on startup
- Migrate old storage formats
- Track migration version

---

### 16. Tool (`src/tool/`)

**Purpose**: Built-in and custom tool implementations

**Tool Registry**:
- Loads built-in tools
- Loads custom tools from `.opencode/tool/`
- Loads plugin tools
- Filters by agent permissions

**Built-in Tools**:
- `bash`, `read`, `write`, `edit`, `glob`, `grep`, `ls`
- `task`, `todowrite`, `todoread`
- `webfetch`, `patch`
- LSP tools (diagnostics, hover)

**Tool Execution**:
1. Permission check
2. Plugin `tool.execute.before` hooks
3. Tool execution with abort signal
4. Plugin `tool.execute.after` hooks
5. Result stored in ToolPart

**Key Files**:
- `tool.ts`: Tool interface definition
- `registry.ts`: Tool registry and loading
- `{tool-name}.ts`: Individual tool implementations

---

### 17. Util (`src/util/`)

**Purpose**: Common utilities

**Key Modules**:
- `log.ts`: Structured logging
- `error.ts`: Named error types (NamedError)
- `filesystem.ts`: File system traversal
- `context.ts`: Async context management
- `lazy.ts`: Lazy initialization
- `lock.ts`: File locking
- `wildcard.ts`: Wildcard pattern matching
- `defer.ts`: Promise deferral

---

## Data Flow

### User Prompt Flow

```
User Input
    ↓
SessionPrompt.prompt()
    ↓
┌─────────────────────────────────────┐
│ 1. Create user message with parts  │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│ 2. Build context                    │
│    - Load history                   │
│    - Build system prompt            │
│    - Convert to ModelMessage[]      │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│ 3. Get provider and model           │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│ 4. Get tools (filtered by agent)    │
└────────────┬────────────────────────┘
             ↓
┌─────────────────────────────────────┐
│ 5. streamText() - LLM invocation    │
└────────────┬────────────────────────┘
             ↓
      ┌──────┴────────┐
      ↓               ↓
  Text Stream    Tool Calls
      ↓               ↓
  TextPart      ┌──────────────┐
                │ Execute Tool │
                │ - Permission  │
                │ - Plugin hooks│
                │ - Tool.execute│
                └──────┬─────────┘
                       ↓
                   ToolPart
                       ↓
                 Result to LLM
                  (next step)
```

---

## Dependencies

### Core Dependencies
- **hono**: HTTP server framework
- **ai**: Vercel AI SDK for LLM orchestration
- **zod**: Schema validation
- **yargs**: CLI argument parsing

### AI/LLM
- **@ai-sdk/amazon-bedrock**: Bedrock support
- **@modelcontextprotocol/sdk**: MCP integration

### File/Code
- **tree-sitter**: Code parsing
- **chokidar**: File watching
- **ignore**: .gitignore support
- **minimatch**: Pattern matching

### Utilities
- **decimal.js**: Precise cost calculations
- **diff**: Text diffing
- **ulid**: Unique ID generation
- **turndown**: HTML to Markdown

### Workspace
- **@opencode-ai/plugin**: Plugin SDK
- **@opencode-ai/sdk**: TypeScript client SDK

---

## Testing

```bash
cd packages/opencode
bun test
```

Tests cover:
- Session lifecycle
- Message serialization
- Tool execution
- Permission system
- Config merging

---

## Building

```bash
cd packages/opencode
bun run build
```

Build script:
1. Compiles TypeScript
2. Bundles dependencies
3. Creates standalone binary

---

## Key Design Patterns

### 1. Instance-Scoped State
```typescript
const myState = Instance.state(async () => {
  // Expensive initialization (runs once per instance)
  return { data: await loadData() }
})

// Usage (returns cached result)
const state = await myState()
```

### 2. Event-Driven Architecture
```typescript
Bus.subscribe(Session.Event.Updated, (evt) => {
  // React to session updates
})
```

### 3. Provider Abstraction
```typescript
const model = await Provider.getModel("anthropic", "claude-sonnet-4")
const result = await streamText({ model: model.language, ... })
```

### 4. Tool Registry
```typescript
const tools = await ToolRegistry.tools(providerID, modelID)
// Returns tools filtered by agent permissions
```

### 5. Permission System
```typescript
await Permission.ask({
  type: "bash",
  pattern: "rm *.tmp",
  title: "Delete temporary files",
  ...
})
```

---

## Security

- **Permission system**: User approval for sensitive operations
- **Wildcard matching**: Pattern-based permissions
- **Abort signals**: Cancel long-running operations
- **File locking**: Prevent concurrent writes
- **Environment isolation**: No sudo, respects working directory

---

## Performance

- **Instance state caching**: Avoid re-initialization
- **Lazy loading**: Dynamic imports for heavy deps
- **Streaming**: LLM responses and tool execution
- **Lock-free reads**: High throughput storage access
- **Provider SDK caching**: Reuse SDK instances

---

## Future Enhancements

- Remote server mode (HTTP/WebSocket)
- Multi-user support with sessions
- Persistent agent memory
- Distributed tool execution
- Enhanced caching strategies