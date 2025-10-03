# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Environment

- **Package Manager**: Bun 1.2.21 (required)
- **Runtime**: TypeScript/JavaScript (Bun runtime), Go 1.24.x for TUI
- **Monorepo**: Turbo-based workspace with multiple packages
- **Primary Language**: TypeScript with Bun APIs

## Common Commands

### Development
```bash
# Run opencode locally (from root)
bun dev

# Run opencode from packages/opencode
cd packages/opencode && bun dev

# Type checking
bun typecheck

# Run tests in opencode package
cd packages/opencode && bun test

# Build project
bun turbo build

# Install dependencies
bun install
```

### Testing
```bash
# Run single test file
bun test path/to/test.ts

# Run tests in opencode package
cd packages/opencode && bun test
```

### Web/Documentation
```bash
# Run documentation site locally
cd packages/web && bun dev

# Build documentation
cd packages/web && bun build
```

## Architecture Overview

### Client-Server Model
opencode uses a client-server architecture where:
- **Server**: TypeScript backend (`packages/opencode/src/server/server.ts`) using Hono framework
- **TUI Client**: Go-based terminal interface (`packages/tui`) that connects to the server
- **API**: RESTful endpoints with OpenAPI specs generated via hono-openapi

This separation allows multiple client implementations (TUI, web, mobile) to interact with the same server.

### Core Components

#### Session Management (`packages/opencode/src/session/`)
- Sessions represent individual chat conversations with the AI
- Messages are versioned (MessageV2 in `message-v2.ts`)
- Supports session compaction, revert, and hierarchical parent-child sessions
- Session state persisted in storage layer

#### Tool System (`packages/opencode/src/tool/`)
Built-in tools available to the AI:
- `bash`, `read`, `write`, `edit`, `grep`, `glob`, `ls`
- `task` (launches sub-agents), `todo` (task tracking)
- `webfetch` (web content retrieval)
- Custom tools can be added via plugins in `.opencode/tool/*.{js,ts}` or through plugin system

Tool registry (`tool/registry.ts`) manages both built-in and custom tools.

#### Agent System (`packages/opencode/src/agent/`)
Multiple agent types with different permissions and purposes:
- **general**: Default subagent for complex searches and multi-step tasks
- **build**: Primary agent for building and development
- Custom agents defined in `.opencode/agent/*.md` files (frontmatter + prompt)
- Agent permissions control access to tools (edit, bash, webfetch)

Agents are configured with:
- Mode: `subagent`, `primary`, or `all`
- Model/provider overrides
- Tool enable/disable flags
- Permission settings (allow/deny/ask)

#### Provider System (`packages/opencode/src/provider/`)
Multi-provider LLM support:
- Anthropic (recommended, with custom headers for claude-code beta features)
- OpenAI, Google, Azure, Amazon Bedrock, local models
- Provider configuration in `provider/models.ts`
- Custom loaders for provider-specific initialization
- Environment variables and config-based authentication

#### Configuration (`packages/opencode/src/config/`)
Hierarchical config system merging:
1. Global config (`~/.config/opencode/`)
2. Project configs (`.opencode/` directories from root to worktree)
3. `opencode.json`/`opencode.jsonc` files (searched upward)
4. Environment flags (`OPENCODE_CONFIG`, `OPENCODE_CONFIG_CONTENT`)

Config includes:
- Agent definitions
- Custom commands (`.opencode/command/*.md`)
- Permissions
- Provider/model settings
- Plugins

#### Project & Instance (`packages/opencode/src/project/`)
- **Project**: Represents a git repository or directory
- **Instance**: Context-specific state for a project directory
- State management via `Instance.state()` for caching expensive operations
- Bootstrap logic loads config, LSP servers, plugins

#### Plugin System (`packages/plugin/`)
Plugins extend opencode via:
- Custom tools (via `tool` hook)
- Event handlers (via `event` hook)
- Config modifications (via `config` hook)
- Custom auth providers (via `auth` hook)
- Lifecycle hooks: `chat.message`, `chat.params`, `permission.ask`, `tool.execute.before/after`

Plugins defined in config and loaded from filesystem or npm packages.

#### LSP Integration (`packages/opencode/src/lsp/`)
Language Server Protocol integration for:
- Code diagnostics
- Hover information
- Code intelligence features
- Per-project LSP server management

#### Storage (`packages/opencode/src/storage/`)
Persistent storage for sessions, messages, and metadata.

### Package Structure

- **packages/opencode**: Core AI agent logic, server, tools, providers
- **packages/tui**: Go-based terminal UI client
- **packages/web**: Documentation website (Astro + Solid.js)
- **packages/app**: Application layer
- **packages/plugin**: Plugin SDK and type definitions
- **packages/sdk/js**: TypeScript SDK for API clients
- **packages/console**: Management console components
- **packages/identity**: Authentication/identity
- **packages/function**: Serverless functions

### Monorepo Structure

Uses Bun workspaces with catalog dependencies for shared versions:
- Common deps: `typescript`, `zod`, `hono`, `ai`, `solid-js`, `remeda`
- Workspace links: `@opencode-ai/plugin`, `@opencode-ai/sdk`

## Code Style

From `AGENTS.md`:
- Keep functions monolithic unless composable/reusable
- Avoid unnecessary destructuring
- Avoid `else` statements unless necessary
- Avoid `try/catch` where possible
- Avoid `any` type
- Avoid `let` statements
- Prefer single-word variable names
- Use Bun APIs extensively (e.g., `Bun.file()`, `Bun.Glob`)

Commit message prefixes (from `.opencode/command/commit.md`):
- `docs:`, `tui:`, `core:`, `ci:`, `ignore:`, `wip:`

## Key Implementation Details

### Message Flow
1. User input → Server endpoint
2. Server creates/updates session
3. SessionPrompt builds context from config, tools, history
4. Provider streams LLM response with tool calls
5. Tools execute with permission checks
6. Results stream back to client
7. Session state updated and persisted

### Tool Execution
1. Tool registry provides available tools based on agent config
2. LLM requests tool use via structured function calling
3. Permission system checks (`packages/opencode/src/permission/`)
4. Tool executes with session context
5. Results returned with title, output, metadata
6. Plugin hooks can intercept before/after execution

### Custom Agents
Agent files in `.opencode/agent/*.md` with frontmatter:
```yaml
---
description: Agent description
model: provider-id/model-id  # optional
tools:
  bash: false  # disable specific tools
permission:
  edit: deny   # override permissions
---
Your custom system prompt here
```

### API Client Generation
After modifying `packages/opencode/src/server/server.ts`:
- API endpoints defined with hono-openapi
- Stainless SDK generation required (done by opencode team)
- SDK published to `packages/sdk/js`

## Development Workflow

1. Make changes to TypeScript code
2. Test locally with `bun dev`
3. Run type checking with `bun typecheck`
4. Run tests if applicable
5. Commit with appropriate prefix (e.g., `core:`, `tui:`)
6. PRs must be bug fixes, LLM improvements, provider support, or documentation (not core features)

## Important Notes

- This is an open-source alternative to Claude Code with multi-provider support
- TUI is a first-class citizen (built by neovim users)
- Server can run remotely while clients connect from anywhere
- Provider-agnostic design allows using any LLM (Anthropic, OpenAI, local, etc.)
- Sessions are versioned (`Installation.VERSION`) for compatibility