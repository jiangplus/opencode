# Plugin Package Architecture

**Package**: `packages/plugin`
**Purpose**: Plugin SDK and type definitions
**Runtime**: TypeScript/Bun
**Published as**: `@opencode-ai/plugin`

---

## Overview

The `plugin` package provides the **Plugin SDK** for extending opencode. It exports TypeScript types and utilities that plugin authors use to create custom tools, add authentication providers, intercept events, and modify behavior.

This is a **pure type definition package** with minimal runtime code—it's designed for developer ergonomics and type safety when authoring plugins.

---

## Package Structure

```
packages/plugin/
├── src/
│   ├── index.ts        # Main exports (types, interfaces)
│   ├── tool.ts         # Tool definition types
│   ├── shell.ts        # BunShell type wrapper
│   └── example.ts      # Example plugin implementation
├── package.json
└── tsconfig.json
```

---

## Exports

### Main Export (`index.ts`)

```typescript
export type {
  Event,
  Project,
  Model,
  Provider,
  Permission,
  UserMessage,
  Part,
  Auth,
  Config,
}

export type {
  PluginInput,
  Plugin,
  Hooks,
}
```

### Tool Export (`tool.ts`)

```typescript
export type {
  ToolDefinition,
  ToolContext,
}
```

---

## Core Types

### PluginInput

The input provided to every plugin:

```typescript
type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>
  project: Project
  directory: string
  worktree: string
  $: BunShell
}
```

**Fields**:
- `client`: opencode API client for making requests
- `project`: Current project info (ID, worktree, VCS)
- `directory`: Current working directory
- `worktree`: Git worktree root
- `$`: Bun shell for running commands

---

### Plugin

The main plugin function signature:

```typescript
type Plugin = (input: PluginInput) => Promise<Hooks>
```

**Usage**:
```typescript
export default async function myPlugin(input: PluginInput): Promise<Hooks> {
  return {
    // Implement hooks here
  }
}
```

---

### Hooks

All available plugin hooks:

```typescript
interface Hooks {
  // System hooks
  event?: (input: { event: Event }) => Promise<void>
  config?: (input: Config) => Promise<void>

  // Tool hooks
  tool?: {
    [key: string]: ToolDefinition
  }

  // Auth hooks
  auth?: {
    provider: string
    loader?: (auth: () => Promise<Auth>, provider: Provider) => Promise<Record<string, any>>
    methods: AuthMethod[]
  }

  // Chat hooks
  "chat.message"?: (input: {}, output: { message: UserMessage; parts: Part[] }) => Promise<void>
  "chat.params"?: (
    input: { model: Model; provider: Provider; message: UserMessage },
    output: { temperature: number; topP: number; options: Record<string, any> }
  ) => Promise<void>

  // Permission hooks
  "permission.ask"?: (input: Permission, output: { status: "ask" | "deny" | "allow" }) => Promise<void>

  // Tool execution hooks
  "tool.execute.before"?: (
    input: { tool: string; sessionID: string; callID: string },
    output: { args: any }
  ) => Promise<void>
  "tool.execute.after"?: (
    input: { tool: string; sessionID: string; callID: string },
    output: { title: string; output: string; metadata: any }
  ) => Promise<void>
}
```

---

## Hook Types

### 1. Event Hook

Receives all events from the event bus:

```typescript
event: async ({ event }) => {
  if (event.type === "session.updated") {
    console.log("Session updated:", event.properties.info)
  }
}
```

**Use Cases**:
- Logging
- Analytics
- External integrations
- Notifications

---

### 2. Config Hook

Modify configuration before use:

```typescript
config: async (config) => {
  // Override model
  config.model = "my-provider/my-model"

  // Add custom agent
  config.agent = {
    ...config.agent,
    myagent: {
      description: "My custom agent",
      tools: { bash: false }
    }
  }
}
```

**Use Cases**:
- Inject default settings
- Add environment-specific config
- Validate configuration

---

### 3. Tool Hook

Add custom tools:

```typescript
tool: {
  "my-tool": {
    args: z.object({
      path: z.string()
    }),
    description: "My custom tool",
    async execute(args, ctx) {
      const result = await ctx.$.nothrow`ls ${args.path}`
      return result.text()
    }
  }
}
```

**Use Cases**:
- Add domain-specific tools
- Integrate with external services
- Wrap CLI tools

---

### 4. Auth Hook

Add authentication providers:

```typescript
auth: {
  provider: "my-provider",
  methods: [
    {
      type: "oauth",
      label: "Login with OAuth",
      async authorize() {
        // Return authorization URL and handle callback
        return {
          url: "https://provider.com/oauth",
          instructions: "Visit the URL to authorize",
          method: "auto",
          async callback() {
            // Poll for completion
            return { type: "success", key: "api-key" }
          }
        }
      }
    }
  ],
  async loader(auth, provider) {
    const { key } = await auth()
    return { apiKey: key }
  }
}
```

**Use Cases**:
- Custom OAuth flows
- API key management
- Token refresh

---

### 5. Chat Hooks

#### chat.message

Intercept or modify messages:

```typescript
"chat.message": async (input, output) => {
  // Add metadata to parts
  output.parts[0].metadata = { source: "plugin" }
}
```

#### chat.params

Modify LLM parameters:

```typescript
"chat.params": async (input, output) => {
  // Override temperature
  output.temperature = 0.5

  // Add custom headers
  output.options.headers = {
    "X-Custom-Header": "value"
  }
}
```

---

### 6. Permission Hook

Override permission decisions:

```typescript
"permission.ask": async (input, output) => {
  // Auto-approve specific patterns
  if (input.type === "edit" && input.metadata.path.includes("test")) {
    output.status = "allow"
  }
}
```

**Statuses**:
- `"ask"`: Show prompt to user (default)
- `"allow"`: Auto-approve
- `"deny"`: Auto-reject

---

### 7. Tool Execution Hooks

#### tool.execute.before

Modify tool arguments before execution:

```typescript
"tool.execute.before": async (input, output) => {
  if (input.tool === "bash") {
    // Add timeout to all bash commands
    output.args.timeout = 60000
  }
}
```

#### tool.execute.after

Modify tool results after execution:

```typescript
"tool.execute.after": async (input, output) => {
  if (input.tool === "grep") {
    // Add metadata
    output.metadata.lines = output.output.split("\n").length
  }
}
```

---

## Tool Definition

### ToolDefinition Type

```typescript
type ToolDefinition = {
  args: Record<string, ZodType>
  description: string
  execute: (args: any, ctx: ToolContext) => Promise<string>
}
```

### ToolContext

```typescript
type ToolContext = {
  sessionID: string
  messageID: string
  callID: string
  agent: string
  abort: AbortSignal
  $: BunShell
}
```

**Fields**:
- `sessionID`, `messageID`, `callID`: Identifiers
- `agent`: Current agent name
- `abort`: Abort signal for cancellation
- `$`: Bun shell for running commands

---

## Example Plugin

### Basic Plugin

```typescript
import type { Plugin, PluginInput, Hooks } from "@opencode-ai/plugin"
import z from "zod"

export default async function examplePlugin(input: PluginInput): Promise<Hooks> {
  return {
    event: async ({ event }) => {
      console.log("Event:", event.type)
    },

    tool: {
      "example-tool": {
        args: {
          message: z.string()
        },
        description: "Example tool",
        async execute(args, ctx) {
          return `You said: ${args.message}`
        }
      }
    },

    "chat.params": async (input, output) => {
      output.temperature = 0.7
    }
  }
}
```

---

## BunShell Type (`shell.ts`)

Provides types for Bun's `$` shell:

```typescript
type BunShell = {
  (strings: TemplateStringsArray, ...args: any[]): ShellPromise
  quiet(): BunShell
  nothrow(): BunShell
  cwd(dir: string): BunShell
  env(vars: Record<string, string>): BunShell
}

type ShellPromise = Promise<ShellOutput> & {
  text(): Promise<string>
  json(): Promise<any>
  quiet(): ShellPromise
  nothrow(): ShellPromise
}
```

**Usage in Tools**:
```typescript
async execute(args, ctx) {
  const result = await ctx.$.nothrow`ls ${args.path}`
  return result.text()
}
```

---

## Publishing Plugins

### Package Structure

```
my-opencode-plugin/
├── src/
│   └── index.ts        # Plugin implementation
├── package.json
├── tsconfig.json
└── README.md
```

### package.json

```json
{
  "name": "my-opencode-plugin",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "dependencies": {
    "@opencode-ai/plugin": "^0.13.5"
  }
}
```

### Publishing

```bash
# Build
npm run build

# Publish to npm
npm publish
```

### Installation

```bash
# In user's opencode config
npm install my-opencode-plugin
```

```json
// opencode.json
{
  "plugin": [
    "my-opencode-plugin@1.0.0"
  ]
}
```

---

## Best Practices

### 1. Type Safety

```typescript
// Use Zod for validation
import z from "zod"

tool: {
  "my-tool": {
    args: {
      path: z.string().min(1),
      count: z.number().int().positive().optional()
    },
    // ...
  }
}
```

### 2. Error Handling

```typescript
async execute(args, ctx) {
  try {
    const result = await ctx.$.nothrow`command ${args.input}`
    if (result.exitCode !== 0) {
      return `Error: ${result.stderr.toString()}`
    }
    return result.stdout.toString()
  } catch (error) {
    return `Unexpected error: ${error.message}`
  }
}
```

### 3. Abort Signals

```typescript
async execute(args, ctx) {
  const result = await fetch("https://api.example.com", {
    signal: ctx.abort  // Respect cancellation
  })
  return await result.text()
}
```

### 4. Async Initialization

```typescript
export default async function myPlugin(input: PluginInput): Promise<Hooks> {
  // Async initialization
  const config = await loadConfig()

  return {
    tool: {
      "my-tool": {
        // Use initialized config
        async execute(args, ctx) {
          return config.apiKey
        }
      }
    }
  }
}
```

---

## Testing Plugins

### Local Testing

```bash
# In plugin directory
npm link

# In opencode config
npm link my-opencode-plugin
```

```json
// opencode.json
{
  "plugin": [
    "file:///path/to/my-opencode-plugin"
  ]
}
```

### Unit Testing

```typescript
import { describe, it, expect } from "bun:test"
import myPlugin from "./src/index"

describe("My Plugin", () => {
  it("should register tool", async () => {
    const hooks = await myPlugin({
      client: mockClient,
      project: mockProject,
      directory: "/test",
      worktree: "/test",
      $: mockShell
    })

    expect(hooks.tool).toHaveProperty("my-tool")
  })
})
```

---

## Dependencies

- **@opencode-ai/sdk**: TypeScript client SDK (for `createOpencodeClient` type)
- **zod**: Schema validation (for tool arguments)

---

## Future Enhancements

- Plugin marketplace
- Plugin versioning and compatibility checks
- Hot-reloading during development
- Plugin sandboxing for security
- Plugin metrics and telemetry