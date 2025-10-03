# TUI Package Architecture

**Package**: `packages/tui`
**Purpose**: Terminal User Interface (TUI) client for opencode
**Runtime**: Go 1.24+
**Framework**: Bubble Tea (Charm)
**Entry**: `cmd/opencode/main.go`

---

## Overview

The `tui` package provides a **rich terminal interface** for interacting with the opencode server. Built with the Bubble Tea framework from Charm, it offers:

- Real-time streaming of AI responses
- Interactive chat interface
- Session management
- File attachments and image viewing
- Permission prompts
- Markdown rendering with syntax highlighting
- Multi-model selection
- Keyboard-driven navigation

This is the **primary client** for opencode, emphasizing speed, beauty, and terminal power-user workflows.

---

## Package Structure

```
packages/tui/
├── cmd/
│   └── opencode/
│       └── main.go              # Entry point
├── internal/
│   ├── api/                     # API client wrapper (generated)
│   ├── app/                     # Main application state & logic
│   ├── attachment/              # File attachment handling
│   ├── clipboard/               # Clipboard integration
│   ├── commands/                # Command palette
│   ├── completions/             # Autocomplete for @mentions
│   ├── components/              # Reusable UI components
│   ├── id/                      # ID utilities
│   ├── layout/                  # Layout management
│   ├── styles/                  # Lipgloss styles
│   ├── theme/                   # Theme definitions
│   ├── tui/                     # Bubble Tea TUI logic
│   ├── util/                    # Utilities
│   └── viewport/                # Custom viewport with scrolling
└── input/                       # Forked charmbracelet/x/input with patches
```

---

## Architecture

### Bubble Tea Model

The TUI follows the **Elm Architecture** (Model-Update-View) via Bubble Tea:

```
┌────────────────────────────────────────┐
│             tea.Program                │
│                                        │
│  ┌──────────────────────────────────┐ │
│  │           Model                   │ │
│  │  (app.State)                     │ │
│  └──────────────┬───────────────────┘ │
│                 │                      │
│                 ▼                      │
│  ┌──────────────────────────────────┐ │
│  │          Update                   │ │
│  │  (handle messages)               │ │
│  └──────────────┬───────────────────┘ │
│                 │                      │
│                 ▼                      │
│  ┌──────────────────────────────────┐ │
│  │           View                    │ │
│  │  (render to terminal)            │ │
│  └──────────────────────────────────┘ │
│                 │                      │
│                 ▼                      │
│              Terminal                  │
└────────────────────────────────────────┘
```

**Message Types**:
- `tea.KeyMsg`: Keyboard input
- `tea.MouseMsg`: Mouse events
- `tea.WindowSizeMsg`: Terminal resize
- Custom messages: API responses, errors, streaming updates

---

## Module Breakdown

### 1. Main Entry (`cmd/opencode/main.go`)

**Responsibilities**:
- Parse CLI flags (`--model`, `--prompt`, `--agent`, `--session`)
- Read piped stdin for initial prompt
- Initialize API client
- Start Bubble Tea program
- Handle graceful shutdown

**Flags**:
- `--model`: Initial model selection
- `--prompt`: Start with a prompt
- `--agent`: Select agent mode
- `--session`: Attach to existing session

**Environment**:
- `OPENCODE_SERVER`: Server URL (default: localhost:4096)

**Workflow**:
1. Check for piped input
2. Initialize state directory (`~/.local/state/opencode/tui`)
3. Create API client
4. Load previous state (recent models, sessions)
5. Start Bubble Tea program
6. Handle signals (SIGINT, SIGTERM)

---

### 2. App State (`internal/app/`)

**Files**:
- `app.go`: Main application logic and Update function
- `state.go`: State persistence (recent models, sessions)
- `prompt.go`: Prompt building and sending

**State Structure**:
```go
type State struct {
    // API
    client *api.Client

    // Session
    session         *Session
    messages        []Message
    streamingMsg    *Message

    // UI State
    width, height   int
    viewport        viewport.Model
    prompt          textarea.Model
    showHelp        bool
    showCommands    bool
    showModels      bool

    // Current selections
    modelID         string
    providerID      string
    agentName       string

    // Attachments
    attachments     []Attachment

    // Permissions
    pendingPerms    []Permission

    // Error state
    err             error
}
```

**Key Functions**:
- `Init()`: Initialize application state
- `Update(tea.Msg)`: Handle messages and update state
- `View()`: Render UI to string
- `SendPrompt()`: Send user prompt to server
- `LoadState()`: Load persisted state from disk
- `SaveState()`: Persist state to disk

---

### 3. API Client (`internal/api/`)

**Generated Code**: Auto-generated from OpenAPI spec via `oapi-codegen`

**Key Operations**:
- `ListSessions()`: Get all sessions
- `GetSession(id)`: Get session details
- `CreateSession()`: New session
- `SendPrompt()`: Stream prompt response
- `ListProviders()`: Available providers
- `ListModels()`: Available models
- `RespondToPermission()`: Approve/deny permission

**Streaming**:
```go
stream, err := client.SendPrompt(ctx, sessionID, prompt)
for {
    event, err := stream.Recv()
    if err == io.EOF {
        break
    }
    // Handle event (MessageUpdated, PartUpdated, etc.)
}
```

---

### 4. Components (`internal/components/`)

**Reusable UI Elements**:

#### Textarea
- Multi-line input with cursor
- Syntax highlighting for code blocks
- Line numbers
- Autocomplete integration

#### Viewport
- Scrollable content area
- Mouse wheel support
- Keyboard navigation (PgUp/PgDown, Home/End)
- Smooth scrolling

#### Message List
- Render conversation history
- Markdown rendering
- Code syntax highlighting
- Inline images (via sixel/kitty protocols)
- Tool call results
- Reasoning traces

#### Modal Dialogs
- Permission prompts
- Model selection
- Command palette
- Help screen

#### Status Bar
- Current model/agent
- Token usage
- Cost
- Streaming indicator

---

### 5. Layout (`internal/layout/`)

**Responsive Layout System**:

```
┌─────────────────────────────────────────────┐
│  Header (model, session info)              │
├─────────────────────────────────────────────┤
│                                             │
│  Viewport (message history)                │
│   - User messages                           │
│   - Assistant responses                     │
│   - Tool calls                              │
│   - Reasoning traces                        │
│                                             │
├─────────────────────────────────────────────┤
│  Prompt Textarea                            │
│  > Type your message...                     │
├─────────────────────────────────────────────┤
│  Footer (status, shortcuts)                 │
└─────────────────────────────────────────────┘
```

**Layout Modes**:
- **Normal**: Full chat view
- **Compact**: Minimal header/footer
- **Help**: Full-screen help overlay
- **Commands**: Command palette overlay
- **Models**: Model selection overlay

**Responsive Behavior**:
- Adjusts to terminal size changes
- Word wrapping for long lines
- Truncates/scrolls as needed

---

### 6. Theme System (`internal/theme/`)

**Theme Definition**:
```go
type Theme struct {
    Primary       lipgloss.Color
    Secondary     lipgloss.Color
    Background    lipgloss.Color
    Foreground    lipgloss.Color
    Accent        lipgloss.Color
    Error         lipgloss.Color
    Warning       lipgloss.Color
    Success       lipgloss.Color
    Muted         lipgloss.Color

    // Syntax highlighting
    CodeBackground lipgloss.Color
    Keyword       lipgloss.Color
    String        lipgloss.Color
    Comment       lipgloss.Color
    // ...
}
```

**Built-in Themes**:
- **Default**: Adaptive light/dark based on terminal
- **Dark**: Pure dark theme
- **Light**: Pure light theme
- **Custom**: User-defined via config

**Color Adaptation**:
- Detects terminal color support (256 color, TrueColor)
- Falls back gracefully for limited terminals
- Respects `COLORTERM` and terminal capabilities

---

### 7. Markdown Rendering

**Library**: Glamour (Charm)

**Features**:
- Full CommonMark support
- Code syntax highlighting (Chroma)
- Tables
- Lists (ordered, unordered, task lists)
- Blockquotes
- Images (inline via terminal protocols)
- Links (displayed, not clickable)

**Syntax Highlighting**:
- 100+ languages supported
- Theme-aware (matches TUI theme)
- Line numbers for code blocks
- Inline code rendering

**Rendering Pipeline**:
1. Parse Markdown (goldmark)
2. Apply syntax highlighting (Chroma)
3. Style with Lipgloss
4. Wrap to terminal width
5. Render to ANSI string

---

### 8. Attachments (`internal/attachment/`)

**File Attachment Types**:
- Text files: Sent as text parts
- Images: Base64 encoded, displayed inline
- PDFs: Extracted text
- Directories: Tree view

**Image Display**:
- **Sixel**: Legacy terminal protocol
- **Kitty**: Modern terminal protocol
- **Fallback**: ASCII art or placeholder

**Attachment Flow**:
1. User adds file via drag-drop or command
2. File read and encoded
3. Sent to server with prompt
4. Server processes and returns response

---

### 9. Permissions (`internal/app/prompt.go`)

**Permission Workflow**:
1. Server requests permission (via SSE event)
2. TUI shows modal dialog
3. User selects: `once`, `always`, `reject`
4. Response sent to server
5. Tool execution continues/aborts

**Permission Dialog**:
```
┌─────────────────────────────────────┐
│  Permission Required                │
├─────────────────────────────────────┤
│  The AI wants to:                   │
│  Execute: rm *.tmp                  │
│                                     │
│  Allow this action?                 │
│                                     │
│  [Once] [Always] [Reject]          │
└─────────────────────────────────────┘
```

---

### 10. Command Palette (`internal/commands/`)

**Commands**:
- `/new`: New session
- `/attach`: Attach to session
- `/agent`: Change agent
- `/model`: Change model
- `/share`: Share session
- `/export`: Export session
- `/clear`: Clear screen
- `/help`: Show help
- Custom commands from server

**Fuzzy Search**:
- Type to filter commands
- Shows descriptions and shortcuts
- Arrow keys to navigate
- Enter to execute

---

### 11. Completions (`internal/completions/`)

**Autocomplete Features**:
- `@agent-name`: Agent mentions
- `@file-path`: File attachments
- `/command`: Slash commands

**Completion Engine**:
- Trigger on `@` or `/`
- Fuzzy search for matches
- Show popup with suggestions
- Tab/Enter to accept
- Esc to cancel

---

### 12. Clipboard (`internal/clipboard/`)

**Clipboard Integration**:
- Copy code blocks
- Copy full messages
- Paste from clipboard

**Platform Support**:
- macOS: `pbcopy`/`pbpaste`
- Linux: `xclip`, `xsel`, or `wl-copy`/`wl-paste`
- Windows: Built-in clipboard API
- Fallback: OSC 52 escape sequence

---

### 13. Viewport (`internal/viewport/`)

**Custom Viewport Implementation**:
- Extends Bubble Tea's viewport
- Enhanced scrolling (smooth, mouse wheel)
- Sticky header/footer
- Content anchoring (stay at bottom during streaming)
- Performance optimizations (virtual rendering)

**Scrolling Modes**:
- **Auto-scroll**: Follow streaming content
- **Manual**: User controls scroll position
- **Snap**: Snap to top/bottom on input

---

### 14. Utilities (`internal/util/`)

**Helper Functions**:
- `Truncate()`: Truncate strings with ellipsis
- `Wrap()`: Word wrapping
- `Center()`: Center text
- `Ellipsis()`: Smart ellipsis for long text
- `FormatTime()`: Relative time formatting
- `FormatCost()`: Currency formatting
- `FormatTokens()`: Token count formatting

---

## Data Flow

### Startup Flow

```
main.go
  ↓
Parse CLI flags
  ↓
Check for piped stdin
  ↓
Initialize API client
  ↓
Load persisted state
  ↓
Create app.State
  ↓
Start tea.Program
  ↓
Render initial view
```

### User Prompt Flow

```
User types in textarea
  ↓
Press Enter/Ctrl+D
  ↓
app.SendPrompt()
  ↓
API: client.SendPrompt()
  ↓
Server: Create user message
  ↓
Server: Stream assistant response
  ↓
TUI: Receive SSE events
  ↓
Update: Handle events
  │
  ├─ MessageUpdated → Append to messages
  ├─ PartUpdated → Update streaming message
  ├─ ToolCallStarted → Show tool indicator
  ├─ ToolCallCompleted → Show result
  └─ MessageCompleted → Stop streaming
  ↓
View: Re-render viewport
```

### Permission Flow

```
Server: Tool requests permission
  ↓
SSE: PermissionUpdated event
  ↓
TUI: Show permission dialog
  ↓
User: Select once/always/reject
  ↓
API: RespondToPermission()
  ↓
Server: Continue or abort tool
```

---

## Key Dependencies

### Charm Libraries
- **bubbletea**: TUI framework (Elm architecture)
- **lipgloss**: Styling and layout
- **bubbles**: Reusable components (textarea, viewport, spinner)
- **glamour**: Markdown rendering
- **x/ansi**: ANSI escape code utilities

### Other
- **go-diff/sergi**: Text diffing
- **fsnotify**: File watching
- **fuzzysearch/lithammer**: Fuzzy search
- **uuid/google**: UUID generation
- **toml/BurntSushi**: TOML parsing for state
- **chroma/alecthomas**: Syntax highlighting

### Generated
- **opencode-sdk-go**: Go client SDK (OpenAPI generated)

---

## State Persistence

**State File**: `~/.local/state/opencode/tui` (TOML)

**Persisted Data**:
```toml
[recently_used_models]
provider_id = "anthropic"
model_id = "claude-sonnet-4"
last_used = 2025-10-01T12:00:00Z

[[recent_sessions]]
id = "session_abc123"
title = "My session"
last_accessed = 2025-10-01T11:00:00Z
```

**Save Triggers**:
- Model selection
- Session creation/switch
- Graceful shutdown

---

## Keyboard Shortcuts

### Global
- `Ctrl+C`: Quit
- `Ctrl+L`: Clear screen
- `F1`: Help
- `Ctrl+P`: Command palette
- `Ctrl+M`: Model selection

### Prompt
- `Enter`: Send (with newline via Shift+Enter)
- `Ctrl+D`: Send
- `Ctrl+V`: Paste
- `Tab`: Autocomplete
- `Esc`: Cancel autocomplete

### Viewport
- `↑`/`↓`: Scroll line
- `PgUp`/`PgDn`: Scroll page
- `Home`/`End`: Top/bottom
- `Space`: Page down
- Mouse wheel: Scroll

### Messages
- `Ctrl+Y`: Copy message
- `Ctrl+B`: Copy code block

---

## Performance Optimizations

### 1. Virtual Rendering
- Only render visible lines
- Off-screen content not computed
- Reduces CPU usage for long conversations

### 2. Lazy Markdown Parsing
- Parse markdown on first render
- Cache parsed output
- Re-parse only when content changes

### 3. Diff-Based Updates
- Only re-render changed components
- Bubble Tea efficiently patches terminal
- Minimal flicker

### 4. Concurrent API Calls
- Stream events concurrently
- Non-blocking UI updates
- Responsive during heavy operations

### 5. Debounced Resizes
- Wait for resize to stabilize
- Avoid excessive re-layouts
- Smooth terminal resizing

---

## Terminal Compatibility

### Supported Terminals
- **Full Support**: iTerm2, Kitty, Alacritty, WezTerm, Ghostty
- **Good Support**: Terminal.app, GNOME Terminal, Konsole
- **Basic Support**: Linux console, Windows Terminal, tmux, screen

### Feature Detection
- TrueColor (24-bit color)
- 256 color
- Sixel graphics
- Kitty graphics protocol
- Mouse support
- Bracketed paste

### Graceful Degradation
- Falls back to 256 colors if TrueColor unavailable
- ASCII art instead of images if no graphics protocol
- Text-only mode for limited terminals

---

## Building

```bash
cd packages/tui
go build -o opencode cmd/opencode/main.go
```

**Release Build**:
```bash
goreleaser release --snapshot --clean
```

**Cross-Compilation**:
- macOS (amd64, arm64)
- Linux (amd64, arm64)
- Windows (amd64)

---

## Testing

```bash
cd packages/tui
go test ./...
```

**Test Coverage**:
- App state transitions
- Message rendering
- Layout calculations
- Utilities

---

## Debugging

### Enable Logging

```bash
DEBUG=1 opencode tui
```

Logs written to: `~/.local/state/opencode/tui.log`

### Common Issues

1. **Rendering glitches**: Check terminal compatibility
2. **Slow performance**: Reduce message history (compaction)
3. **Clipboard not working**: Install `xclip`/`xsel` (Linux)
4. **Images not displaying**: Check terminal graphics protocol support

---

## Future Enhancements

### Planned Features
- Split panes (multiple sessions)
- Session search
- Message editing
- Voice input (via local STT)
- Custom keybindings
- Themes from config
- Plugin system for custom commands

### Performance
- Incremental rendering
- Web workers for heavy parsing
- GPU-accelerated scrolling (future terminals)

---

## Design Philosophy

### Speed First
- Native Go performance
- Minimal dependencies
- Efficient rendering

### Terminal Power User
- Keyboard-driven workflow
- Composable with Unix tools
- SSH-friendly (no GUI required)

### Beautiful
- Polished UI with Charm libraries
- Thoughtful animations
- Cohesive design language

### Reliable
- Graceful error handling
- State persistence
- Network resilience (retry, reconnect)