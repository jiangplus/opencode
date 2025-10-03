# SDKs Architecture

**Directory**: `sdks/`
**Purpose**: IDE integrations and editor extensions
**Current SDKs**: VS Code

---

## Overview

The `sdks` directory contains **IDE integrations** that make it easy to launch and interact with opencode from within code editors. These extensions provide deep integration with the development workflow, allowing developers to use opencode without leaving their editor.

Current integrations:
- **VS Code** - Visual Studio Code extension

Future integrations:
- JetBrains IDEs (IntelliJ, PyCharm, WebStorm, etc.)
- Neovim/Vim
- Emacs
- Sublime Text
- Zed

---

## Directory Structure

```
sdks/
└── vscode/              # VS Code extension
    ├── src/
    │   └── extension.ts # Extension entry point
    ├── images/          # Icons and assets
    ├── package.json     # Extension manifest
    ├── tsconfig.json
    └── esbuild.js       # Build script
```

---

## VS Code Extension

### Overview

The opencode VS Code extension provides seamless integration with Visual Studio Code:

- **Quick Launch**: Open opencode terminal with keyboard shortcut
- **File Context**: Automatically include current file in prompt
- **Split View**: Opens in side-by-side panel
- **Port-based Communication**: Communicates with TUI via HTTP

### Package Info

```json
{
  "name": "opencode",
  "displayName": "opencode",
  "version": "0.13.5",
  "publisher": "sst-dev",
  "engines": {
    "vscode": "^1.94.0"
  }
}
```

---

## Features

### 1. Open Terminal

**Command**: `opencode.openTerminal`
**Keyboard**: `Cmd+Escape` (Mac), `Ctrl+Escape` (Windows/Linux)

Opens opencode in integrated terminal:
- Reuses existing terminal if one exists
- Focuses terminal panel
- Preserves context

**Implementation**:
```typescript
vscode.commands.registerCommand("opencode.openTerminal", async () => {
  const existingTerminal = vscode.window.terminals.find(t => t.name === "opencode")
  if (existingTerminal) {
    existingTerminal.show()
    return
  }
  await openTerminal()
})
```

---

### 2. Open New Terminal

**Command**: `opencode.openNewTerminal`
**Keyboard**: `Cmd+Shift+Escape` (Mac), `Ctrl+Shift+Escape` (Windows/Linux)
**UI**: Button in editor toolbar

Always opens a fresh opencode terminal:
- Creates new terminal instance
- Opens in split view (beside editor)
- Automatically includes active file context

**Implementation**:
```typescript
vscode.commands.registerCommand("opencode.openNewTerminal", async () => {
  await openTerminal()
})
```

---

### 3. Add File to Terminal

**Command**: `opencode.addFilepathToTerminal`
**Keyboard**: `Cmd+Alt+K` (Mac), `Ctrl+Alt+K` (Windows/Linux)

Inserts active file reference into opencode prompt:
- Gets current file path
- Sends to TUI via HTTP API
- Formats as `@file-path` mention

**Implementation**:
```typescript
vscode.commands.registerCommand("opencode.addFilepathToTerminal", async () => {
  const fileRef = getActiveFile()
  if (!fileRef) return

  const terminal = vscode.window.activeTerminal
  if (terminal?.name === "opencode") {
    const port = terminal.creationOptions.env?.["_EXTENSION_OPENCODE_PORT"]
    if (port) {
      await appendPrompt(parseInt(port), fileRef)
    } else {
      terminal.sendText(fileRef)
    }
  }
})
```

---

## Architecture

### Terminal Creation

When opening terminal:
1. Generate random port (16384-65535)
2. Create terminal with custom environment variables
3. Execute `opencode --port <port>` command
4. Wait for server to be ready (polls `/app` endpoint)
5. Append active file to prompt if available

```typescript
async function openTerminal() {
  const port = Math.floor(Math.random() * (65535 - 16384 + 1)) + 16384

  const terminal = vscode.window.createTerminal({
    name: "opencode",
    location: {
      viewColumn: vscode.ViewColumn.Beside,
      preserveFocus: false,
    },
    env: {
      _EXTENSION_OPENCODE_PORT: port.toString(),
      OPENCODE_CALLER: "vscode",
    },
  })

  terminal.show()
  terminal.sendText(`opencode --port ${port}`)

  // Wait for ready
  let tries = 10
  while (tries > 0) {
    try {
      await fetch(`http://localhost:${port}/app`)
      break
    } catch (e) {
      await new Promise(resolve => setTimeout(resolve, 200))
      tries--
    }
  }

  // Append active file
  const fileRef = getActiveFile()
  if (fileRef) {
    await appendPrompt(port, `In ${fileRef}`)
  }
}
```

---

### Port-based Communication

Extension communicates with TUI via HTTP API:

**Endpoint**: `http://localhost:<port>/tui/append-prompt`

**Request**:
```json
{
  "text": "@path/to/file.ts"
}
```

This allows the extension to:
- Insert text into TUI prompt
- Query TUI state
- Send commands

**Advantages**:
- No need for complex IPC
- Works with any terminal emulator
- Simple HTTP API

---

### File Reference Format

Gets active file and formats it:

```typescript
function getActiveFile() {
  const activeEditor = vscode.window.activeTextEditor
  if (!activeEditor) return

  const document = activeEditor.document
  const workspaceFolder = vscode.workspace.getWorkspaceFolder(document.uri)

  if (workspaceFolder) {
    // Relative path within workspace
    const relativePath = vscode.workspace.asRelativePath(document.uri)
    return `@${relativePath}`
  } else {
    // Absolute path
    return `@${document.uri.fsPath}`
  }
}
```

**Output Examples**:
- `@src/components/Button.tsx` (relative)
- `@/Users/name/project/file.ts` (absolute)

---

## UI Integration

### Editor Toolbar Button

Button appears in editor title bar:

```json
{
  "menus": {
    "editor/title": [
      {
        "command": "opencode.openNewTerminal",
        "group": "navigation"
      }
    ]
  }
}
```

**Icons**:
- Light theme: `images/button-dark.svg`
- Dark theme: `images/button-light.svg`

---

### Keyboard Shortcuts

Default keybindings:

| Command | Mac | Windows/Linux | Description |
|---------|-----|---------------|-------------|
| Open Terminal | `Cmd+Escape` | `Ctrl+Escape` | Open or focus terminal |
| Open New Terminal | `Cmd+Shift+Escape` | `Ctrl+Shift+Escape` | Always create new |
| Add File | `Cmd+Alt+K` | `Ctrl+Alt+K` | Insert file reference |

Users can customize in VS Code keyboard settings.

---

## Assets

### Icons

Located in `images/`:
- `icon.png`: Extension icon (128x128)
- `button-light.svg`: Button for dark theme
- `button-dark.svg`: Button for light theme

Icons follow VS Code design guidelines:
- Simple, recognizable shapes
- Monochrome (themed)
- SVG format for scalability

---

## Building

### Development

```bash
cd sdks/vscode
bun install
bun run compile
```

Opens extension development host in VS Code.

### Packaging

```bash
bun run package
```

Creates `.vsix` file for distribution.

### Publishing

```bash
# Install vsce
npm install -g @vscode/vsce

# Package
vsce package

# Publish to marketplace
vsce publish
```

**Published as**: `sst-dev.opencode` on VS Code Marketplace

---

## Configuration

Extension uses VS Code settings:

```json
{
  "opencode.terminalName": "opencode",
  "opencode.defaultPort": null,
  "opencode.autoIncludeFile": true
}
```

*(Settings to be implemented in future versions)*

---

## Testing

### Manual Testing

1. Open VS Code
2. Press `F5` to launch Extension Development Host
3. Open a file
4. Press `Cmd+Escape`
5. Verify terminal opens with file context

### Automated Testing

```bash
bun run test
```

Uses `@vscode/test-electron` for integration tests.

---

## Distribution

### VS Code Marketplace

Published to: https://marketplace.visualstudio.com/items?itemName=sst-dev.opencode

Users install via:
```
ext install sst-dev.opencode
```

Or search "opencode" in Extensions panel.

### Manual Installation

Users can install `.vsix` file:
```bash
code --install-extension opencode-0.13.5.vsix
```

---

## Future IDE Integrations

### JetBrains IDEs

**Platform**: IntelliJ IDEA, PyCharm, WebStorm, etc.
**Language**: Kotlin (IntelliJ Plugin SDK)

**Features**:
- Tool window integration
- Action buttons in editor
- File context awareness
- Project model access

**Structure**:
```
sdks/jetbrains/
├── src/
│   └── main/
│       ├── kotlin/
│       │   └── OpencodePlugin.kt
│       └── resources/
│           └── META-INF/
│               └── plugin.xml
└── build.gradle.kts
```

---

### Neovim

**Platform**: Neovim (Lua plugin)
**Language**: Lua

**Features**:
- Lua API integration
- Terminal buffer
- Keymaps
- Autocomplete

**Structure**:
```
sdks/nvim/
├── lua/
│   └── opencode/
│       ├── init.lua
│       ├── terminal.lua
│       └── config.lua
└── plugin/
    └── opencode.vim
```

**Usage**:
```lua
-- In init.lua
require("opencode").setup({
  keybinds = {
    open_terminal = "<leader>oc",
    add_file = "<leader>of",
  }
})
```

---

### Emacs

**Platform**: Emacs (Elisp package)
**Language**: Emacs Lisp

**Features**:
- Minor mode
- Interactive functions
- Keybindings
- Comint integration

**Structure**:
```
sdks/emacs/
├── opencode.el           # Main package
├── opencode-term.el      # Terminal integration
└── opencode-autoloads.el
```

**Usage**:
```elisp
;; In .emacs or init.el
(require 'opencode)
(global-set-key (kbd "C-c o") 'opencode-open-terminal)
```

---

### Sublime Text

**Platform**: Sublime Text (Python plugin)
**Language**: Python

**Features**:
- Command palette
- Sidebar buttons
- Status bar
- Build system

**Structure**:
```
sdks/sublime/
├── opencode.py
├── Default.sublime-commands
├── Default.sublime-keymap
└── opencode.sublime-build
```

---

### Zed

**Platform**: Zed (Rust extension)
**Language**: Rust

**Features**:
- Language server integration
- Panel UI
- Keybindings
- Project context

**Structure**:
```
sdks/zed/
├── src/
│   ├── lib.rs
│   └── terminal.rs
├── Cargo.toml
└── extension.json
```

---

## Design Principles

### 1. Minimal Friction
- Single keypress to launch
- Automatic context awareness
- No configuration required

### 2. Editor Native
- Follow IDE conventions
- Use native UI components
- Respect editor themes

### 3. Lightweight
- Small bundle size
- Fast activation
- No background processes

### 4. Consistent UX
- Same shortcuts across IDEs where possible
- Uniform behavior
- Predictable interactions

---

## Dependencies

### VS Code Extension

```json
{
  "devDependencies": {
    "@types/vscode": "^1.94.0",
    "@typescript-eslint/eslint-plugin": "^8.31.1",
    "@typescript-eslint/parser": "^8.31.1",
    "eslint": "^9.25.1",
    "esbuild": "^0.25.3",
    "typescript": "^5.8.3"
  }
}
```

**Runtime**: None (uses VS Code API only)

---

## Future Enhancements

### VS Code Specific
- Custom views (sidebar panel)
- Webview for rich UI
- Settings UI
- Inline suggestions
- Command history

### Cross-IDE
- Unified configuration format
- Shared icon set
- Common HTTP API
- Plugin marketplace

### Advanced Features
- Multi-cursor support
- Inline diff previews
- Code actions integration
- Debugger integration
- Git integration