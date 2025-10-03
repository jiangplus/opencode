# SDK Package Architecture

**Package**: `packages/sdk`
**Purpose**: Client SDKs for opencode API
**Languages**: TypeScript (js), Go (go)
**Published as**: `@opencode-ai/sdk` (JS), `github.com/sst/opencode-sdk-go` (Go)

---

## Overview

The `sdk` package provides **type-safe client libraries** for interacting with the opencode API. It includes:

- **JavaScript/TypeScript SDK** (`js/`): For web, Node.js, Bun, and Deno
- **Go SDK** (`go/`): For the TUI client and Go applications
- **Stainless Config** (`stainless/`): Configuration for SDK generation

Both SDKs are **auto-generated from OpenAPI specs** to ensure consistency with the server API.

---

## Package Structure

```
packages/sdk/
├── js/                  # TypeScript/JavaScript SDK
│   ├── src/
│   │   ├── index.ts     # Main exports
│   │   ├── client.ts    # Generated client
│   │   └── server.ts    # Server-side utilities
│   ├── script/
│   │   └── build.ts     # Build script
│   └── package.json
├── go/                  # Go SDK
│   ├── *.go             # Generated Go files
│   ├── option/          # Option patterns
│   └── go.mod
└── stainless/           # Stainless configuration
    └── config.yaml
```

---

## JavaScript/TypeScript SDK

### Architecture

**Generation**: Auto-generated using `@hey-api/openapi-ts`

**Process**:
1. opencode server exposes OpenAPI spec at `/doc`
2. Build script fetches spec
3. `@hey-api/openapi-ts` generates TypeScript client
4. Types and client methods created automatically

### Usage

#### Installation

```bash
npm install @opencode-ai/sdk
```

#### Basic Client

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096"
})

// List sessions
const sessions = await client.session.list()

// Get session
const session = await client.session.get({ id: "session_abc" })

// Send prompt
const result = await client.session.prompt({
  sessionID: "session_abc",
  parts: [{ type: "text", text: "Hello!" }]
})
```

### Generated Structure

```typescript
// Main client
export function createOpencodeClient(options: {
  baseUrl: string
  fetch?: typeof fetch
}): OpencodeClient

// Client interface
interface OpencodeClient {
  session: SessionAPI
  config: ConfigAPI
  project: ProjectAPI
  provider: ProviderAPI
  permission: PermissionAPI
  tool: ToolAPI
}

// Session API example
interface SessionAPI {
  list(): Promise<Session[]>
  get(params: { id: string }): Promise<Session>
  create(data: CreateSessionInput): Promise<Session>
  delete(params: { id: string }): Promise<void>
  prompt(data: PromptInput): Promise<PromptResponse>
}
```

### Type Exports

All types from the OpenAPI spec are exported:

```typescript
export type {
  // Session types
  Session,
  Message,
  Part,
  TextPart,
  ToolPart,
  FilePart,
  ReasoningPart,

  // Config types
  Config,
  Agent,
  Provider,
  Model,

  // Permission types
  Permission,

  // Request/response types
  CreateSessionInput,
  PromptInput,
  PromptResponse,
}
```

### Server-Side Utilities

**server.ts** exports utilities for server-side use:

```typescript
// Streaming support
export async function* streamPrompt(
  client: OpencodeClient,
  input: PromptInput
): AsyncGenerator<Event> {
  const response = await client.session.prompt(input)
  const reader = response.body.getReader()
  // Parse SSE stream...
}

// Error handling
export class OpencodeError extends Error {
  constructor(
    public status: number,
    public data: any
  ) {
    super(`Opencode API error: ${status}`)
  }
}
```

### Build Script

**script/build.ts**:

```typescript
// 1. Fetch OpenAPI spec from server
const spec = await fetch("http://localhost:4096/doc").then(r => r.json())

// 2. Generate TypeScript client
await generate({
  input: spec,
  output: "src/client.ts",
  client: "fetch",
  types: {
    dates: true,
    enums: "typescript",
  },
  services: {
    asClass: true,
  }
})

// 3. Build with tsc
execSync("tsc --build")
```

---

## Go SDK

### Architecture

**Generation**: Auto-generated using Stainless (OpenAI's SDK generator)

**Process**:
1. OpenAPI spec fetched from server
2. Stainless generates Go code
3. Idiomatic Go patterns (options, contexts, errors)

### Usage

#### Installation

```bash
go get github.com/sst/opencode-sdk-go
```

#### Basic Client

```go
import (
  "context"
  "github.com/sst/opencode-sdk-go"
  "github.com/sst/opencode-sdk-go/option"
)

client := opencode.NewClient(
  option.WithBaseURL("http://localhost:4096"),
)

// List sessions
sessions, err := client.Session.List(context.Background())
if err != nil {
  panic(err)
}

// Get session
session, err := client.Session.Get(
  context.Background(),
  "session_abc",
)

// Send prompt
result, err := client.Session.Prompt(
  context.Background(),
  opencode.SessionPromptParams{
    SessionID: "session_abc",
    Parts: []opencode.Part{
      {Type: "text", Text: "Hello!"},
    },
  },
)
```

### Generated Structure

```go
// Client
type Client struct {
  Session    *SessionService
  Config     *ConfigService
  Project    *ProjectService
  Provider   *ProviderService
  Permission *PermissionService
  Tool       *ToolService
}

// Session service
type SessionService struct {
  client *Client
}

func (s *SessionService) List(ctx context.Context, opts ...option.RequestOption) ([]Session, error)
func (s *SessionService) Get(ctx context.Context, id string, opts ...option.RequestOption) (*Session, error)
func (s *SessionService) Create(ctx context.Context, body SessionCreateParams, opts ...option.RequestOption) (*Session, error)
func (s *SessionService) Delete(ctx context.Context, id string, opts ...option.RequestOption) error
func (s *SessionService) Prompt(ctx context.Context, body SessionPromptParams, opts ...option.RequestOption) (*PromptResponse, error)
```

### Type Exports

```go
// Session types
type Session struct {
  ID        string    `json:"id"`
  ProjectID string    `json:"projectId"`
  Title     string    `json:"title"`
  Time      TimeInfo  `json:"time"`
  // ...
}

type Message struct {
  ID        string `json:"id"`
  SessionID string `json:"sessionId"`
  Role      string `json:"role"`
  // ...
}

// Part types (union)
type Part interface {
  isPart()
}

type TextPart struct {
  Type string `json:"type"`
  Text string `json:"text"`
  // ...
}
```

### Option Pattern

Go SDK uses functional options:

```go
// Request options
option.WithHeader("X-Custom", "value")
option.WithTimeout(30 * time.Second)
option.WithMaxRetries(3)

// Usage
session, err := client.Session.Get(
  ctx,
  "session_abc",
  option.WithTimeout(10 * time.Second),
)
```

### Error Handling

```go
// Error types
type Error struct {
  Status  int
  Message string
  Data    map[string]interface{}
}

func (e *Error) Error() string {
  return fmt.Sprintf("opencode: %s (status %d)", e.Message, e.Status)
}

// Usage
session, err := client.Session.Get(ctx, "invalid")
if err != nil {
  var opencodeErr *opencode.Error
  if errors.As(err, &opencodeErr) {
    fmt.Println("API error:", opencodeErr.Status)
  }
}
```

### Streaming

```go
// SSE stream handling
stream, err := client.Session.PromptStream(
  ctx,
  opencode.SessionPromptParams{...},
)
if err != nil {
  panic(err)
}
defer stream.Close()

for stream.Next() {
  event := stream.Event()
  switch event.Type {
  case "message.updated":
    // Handle message update
  case "part.updated":
    // Handle part update
  }
}

if err := stream.Err(); err != nil {
  panic(err)
}
```

---

## Stainless Configuration

**stainless/config.yaml**:

```yaml
organization: opencode
package_name: opencode-sdk-go
api_name: Opencode
language: go
output_directory: ../go

features:
  streaming: true
  retries: true
  pagination: true

types:
  discriminators:
    enabled: true
  unions:
    enabled: true

services:
  naming: pascalCase
```

---

## API Coverage

### Endpoints

Both SDKs cover all opencode API endpoints:

| Category | Endpoints |
|----------|-----------|
| **Session** | `list`, `get`, `create`, `delete`, `message`, `prompt` |
| **Config** | `get`, `update` |
| **Project** | `get`, `list` |
| **Provider** | `list`, `models` |
| **Permission** | `list`, `respond` |
| **Tool** | `list`, `ids` |
| **Share** | `create`, `get` |
| **Log** | `stream` |

### Streaming Endpoints

- `session.prompt`: SSE stream of events
- `log.stream`: SSE stream of logs

---

## Testing

### JavaScript/TypeScript

```typescript
import { describe, it, expect } from "bun:test"
import { createOpencodeClient } from "@opencode-ai/sdk"

describe("SDK Client", () => {
  const client = createOpencodeClient({
    baseUrl: "http://localhost:4096"
  })

  it("should list sessions", async () => {
    const sessions = await client.session.list()
    expect(Array.isArray(sessions)).toBe(true)
  })
})
```

### Go

```go
func TestClient(t *testing.T) {
  client := opencode.NewClient(
    option.WithBaseURL("http://localhost:4096"),
  )

  sessions, err := client.Session.List(context.Background())
  if err != nil {
    t.Fatal(err)
  }

  if len(sessions) == 0 {
    t.Error("expected sessions")
  }
}
```

---

## Building

### JavaScript/TypeScript

```bash
cd packages/sdk/js
bun run build
```

**Steps**:
1. Fetch OpenAPI spec from running server
2. Generate client with `@hey-api/openapi-ts`
3. Compile TypeScript
4. Output to `dist/`

### Go

```bash
cd packages/sdk/go
stainless generate
go build ./...
```

**Steps**:
1. Stainless reads `../stainless/config.yaml`
2. Fetches OpenAPI spec
3. Generates Go code
4. Format with `gofmt`

---

## Publishing

### JavaScript/TypeScript

```bash
cd packages/sdk/js
npm publish
```

**Published as**: `@opencode-ai/sdk`

### Go

Go SDK published via git tags:

```bash
git tag sdk/go/v0.13.5
git push --tags
```

**Import**: `github.com/sst/opencode-sdk-go`

---

## Versioning

- SDKs versioned with opencode releases
- Breaking changes follow semver
- OpenAPI spec changes trigger SDK regeneration

---

## Future Enhancements

- **Python SDK**: Stainless supports Python
- **Ruby SDK**: For scripting and automation
- **Rust SDK**: For high-performance clients
- **WebSocket support**: For real-time communication
- **GraphQL layer**: Alternative to REST
- **SDK middleware**: Interceptors, logging, metrics