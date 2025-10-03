# App Package Architecture

**Package**: `packages/app`
**Purpose**: Web-based UI client for opencode
**Framework**: Solid.js + Vite
**Published as**: Hosted web application

---

## Overview

The `app` package provides a **web-based interface** for opencode, offering an alternative to the TUI for users who prefer browser-based workflows. Built with Solid.js for reactive UI and Vite for fast development.

Features:
- Chat interface with real-time streaming
- Session management
- File attachments
- Markdown rendering with syntax highlighting
- Model/agent selection
- Permission prompts
- Responsive design (desktop and mobile)

---

## Technology Stack

- **Solid.js**: Reactive UI framework
- **Vite**: Build tool and dev server
- **TailwindCSS**: Utility-first styling
- **@opencode-ai/sdk**: TypeScript client SDK
- **Solid Router**: Client-side routing
- **Kobalte**: Accessible UI components
- **Shiki**: Syntax highlighting
- **Marked**: Markdown parsing

---

## Package Structure

```
packages/app/
├── src/
│   ├── components/      # Reusable UI components
│   ├── pages/           # Route pages
│   ├── stores/          # Global state (Solid stores)
│   ├── utils/           # Utilities
│   └── index.tsx        # Entry point
├── scripts/
│   └── build.ts         # Build scripts
├── index.html           # HTML template
├── vite.config.ts       # Vite configuration
└── package.json
```

---

## Key Features

### 1. Chat Interface
- Real-time message streaming
- Markdown rendering with code highlighting
- Tool call results inline
- Reasoning traces expandable

### 2. Session Management
- Create/switch/delete sessions
- Session history sidebar
- Search sessions

### 3. File Attachments
- Drag-and-drop or file picker
- Image preview
- Multiple file support

### 4. Model Selection
- List available providers/models
- Quick model switcher
- Model info (cost, context window)

### 5. Agent Selection
- Switch between agents
- Agent descriptions and permissions

### 6. Permission System
- Modal prompts for approvals
- Once/always/reject options
- Permission history

---

## Architecture

### Reactive State Management

Uses Solid.js stores for global state:

```typescript
// stores/session.ts
import { createStore } from "solid-js/store"

export const [sessionStore, setSessionStore] = createStore({
  current: null as Session | null,
  messages: [] as Message[],
  streaming: false,
})
```

### Component Structure

```
App
├── Header (model selector, session info)
├── Sidebar (session list)
├── Main
│   ├── MessageList (chat history)
│   ├── StreamingMessage (live response)
│   └── PromptInput (textarea + attach)
└── Modals
    ├── PermissionModal
    ├── ModelSelector
    └── Settings
```

### API Integration

Uses `@opencode-ai/sdk` for server communication:

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: import.meta.env.VITE_API_URL || "http://localhost:4096"
})

// Send prompt
const response = await client.session.prompt({
  sessionID: currentSession.id,
  parts: [{ type: "text", text: userInput }]
})
```

### Routing

```typescript
// src/index.tsx
import { Router, Route } from "@solidjs/router"

<Router>
  <Route path="/" component={HomePage} />
  <Route path="/session/:id" component={ChatPage} />
  <Route path="/settings" component={SettingsPage} />
</Router>
```

---

## Development

```bash
cd packages/app
bun dev
```

Runs dev server on `http://localhost:5173`

---

## Building

```bash
bun build
```

Output: `dist/` (static files ready for hosting)

---

## Deployment

Deployed as static site (Cloudflare Pages, Vercel, Netlify):

```bash
bun build
# Upload dist/ to hosting provider
```

---

## Future Enhancements

- PWA support (offline mode)
- Multi-tab sessions
- Voice input (Web Speech API)
- Collaborative sessions (real-time multiplayer)
- Mobile app (Capacitor)