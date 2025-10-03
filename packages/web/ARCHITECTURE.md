# Web Package Architecture

**Package**: `packages/web`
**Purpose**: Documentation website and marketing pages
**Framework**: Astro + Solid.js
**Hosting**: Cloudflare Pages
**URL**: https://opencode.ai

---

## Overview

The `web` package provides the **official documentation and marketing website** for opencode. Built with Astro for static site generation and Solid.js for interactive components, it offers:

- Comprehensive documentation (installation, configuration, API reference)
- Marketing landing pages
- Interactive examples and demos
- API documentation
- Blog posts and changelog

The site is optimized for performance with static generation, edge caching, and minimal JavaScript.

---

## Package Structure

```
packages/web/
├── src/
│   ├── assets/           # Images, logos, icons
│   ├── components/       # Solid.js components
│   ├── content/          # Markdown documentation
│   │   └── docs/         # Documentation pages
│   ├── pages/            # Astro pages (routes)
│   ├── styles/           # Global styles
│   └── types/            # TypeScript types
├── public/               # Static assets
├── astro.config.mjs      # Astro configuration
├── package.json
└── tsconfig.json
```

---

## Technology Stack

### Core Framework
- **Astro 5.7**: Static site generator with islands architecture
- **Solid.js 1.9**: Reactive UI framework for interactive components
- **TypeScript**: Type safety

### Documentation
- **Starlight 0.34**: Astro documentation theme
- **Markdown**: Content authoring
- **Shiki 3.4**: Syntax highlighting
- **Rehype**: Markdown transformations

### Styling
- **CSS**: Scoped styles via Astro
- **IBM Plex Mono**: Monospace font for code

### Build & Deploy
- **Cloudflare Pages**: Edge hosting
- **Vite**: Bundler (via Astro)
- **Sharp**: Image optimization

---

## Architecture

### Astro Islands

Astro uses the **Islands Architecture** for optimal performance:

```
┌─────────────────────────────────────┐
│        Static HTML (SSG)            │
│  ┌─────────────────────────────┐   │
│  │  Header (static)            │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │  Documentation Content      │   │
│  │  (Markdown → HTML)          │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │  Interactive Component      │◄──┼─ Solid.js Island (hydrated)
│  │  (Solid.js)                 │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │  Footer (static)            │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

**Benefits**:
- Most content is static HTML (no JS)
- Interactive components hydrated selectively
- Minimal bundle size
- Fast page loads

---

## Module Breakdown

### 1. Pages (`src/pages/`)

**Astro Pages** define routes:

```
src/pages/
├── index.astro         # Homepage (/)
├── docs/
│   └── [...slug].astro # Documentation routes (/docs/*)
├── blog/
│   └── [...slug].astro # Blog posts (/blog/*)
└── api/
    └── [...slug].astro # API reference (/api/*)
```

**File-based Routing**:
- `index.astro` → `/`
- `about.astro` → `/about`
- `docs/[slug].astro` → `/docs/installation`, etc.

**Dynamic Routes**:
- `[...slug].astro`: Catch-all for dynamic content
- Content loaded from markdown files

---

### 2. Content (`src/content/`)

**Markdown Documentation**:

```
src/content/docs/
├── index.mdx           # Docs homepage
├── installation.md     # Installation guide
├── configuration.md    # Configuration
├── agents.md           # Agents
├── tools.md            # Tools
├── providers.md        # Providers
└── api/
    ├── session.md      # Session API
    ├── config.md       # Config API
    └── ...
```

**Frontmatter**:
```yaml
---
title: Installation
description: How to install opencode
sidebar:
  order: 1
---
```

**Content Collections**:
- Defined in `src/content.config.ts`
- Type-safe frontmatter validation
- Auto-generated types for content

---

### 3. Components (`src/components/`)

**Solid.js Components**:

- **Interactive Examples**: Code playgrounds, live demos
- **API Explorer**: Interactive API documentation
- **Search**: Full-text search (Pagefind)
- **Navigation**: Dynamic sidebar, breadcrumbs
- **Code Blocks**: Copy button, syntax highlighting
- **Tabs**: Tabbed content (e.g., install commands)

**Example Component**:
```tsx
// src/components/CodeBlock.tsx
import { createSignal } from "solid-js"

export function CodeBlock(props) {
  const [copied, setCopied] = createSignal(false)

  const copy = () => {
    navigator.clipboard.writeText(props.code)
    setCopied(true)
    setTimeout(() => setCopied(false), 2000)
  }

  return (
    <div class="code-block">
      <button onClick={copy}>
        {copied() ? "Copied!" : "Copy"}
      </button>
      <pre><code>{props.code}</code></pre>
    </div>
  )
}
```

**Usage in Astro**:
```astro
---
import CodeBlock from '@/components/CodeBlock'
---

<CodeBlock client:load code="npm install opencode-ai" />
```

---

### 4. Assets (`src/assets/`)

**Static Assets**:
- Logos (SVG, PNG)
- Screenshots
- Icons
- Diagrams

**Image Optimization**:
- Astro's `<Image>` component
- Automatic format conversion (WebP, AVIF)
- Responsive images
- Lazy loading

---

### 5. Styles (`src/styles/`)

**Global Styles**:
- CSS variables for theming
- Typography styles
- Responsive layout
- Dark mode support

**Scoped Styles**:
- Component-specific CSS via `<style>` in Astro files
- No global pollution

---

### 6. Documentation Theme (Starlight)

**Starlight Features**:
- Sidebar navigation (auto-generated from content)
- Search (Pagefind)
- Dark/light mode toggle
- Mobile-responsive
- i18n ready
- Social links (GitHub, Discord, Twitter)

**Configuration** (`astro.config.mjs`):
```js
starlight({
  title: 'opencode',
  logo: {
    src: './src/assets/logo.svg',
  },
  social: {
    github: 'https://github.com/sst/opencode',
    discord: 'https://opencode.ai/discord',
  },
  sidebar: [
    { label: 'Getting Started', items: [...] },
    { label: 'Configuration', items: [...] },
    { label: 'API Reference', items: [...] },
  ],
})
```

---

## Build Process

### Development

```bash
cd packages/web
bun dev
```

**Dev Server**:
- Hot module replacement (HMR)
- Fast refresh
- Runs on `http://localhost:4321`

**Environment Variables**:
- `VITE_API_URL`: API endpoint (default: localhost:4096)

---

### Production Build

```bash
cd packages/web
bun build
```

**Build Steps**:
1. Pre-render all pages (SSG)
2. Optimize images (Sharp)
3. Bundle JavaScript (Vite)
4. Minify HTML/CSS/JS
5. Generate sitemap
6. Output to `dist/`

**Output**:
```
dist/
├── index.html
├── docs/
│   ├── installation.html
│   ├── configuration.html
│   └── ...
├── _astro/              # Bundled assets (JS, CSS)
└── assets/              # Optimized images
```

---

### Deployment (Cloudflare Pages)

**Deployment**:
1. Push to `dev` branch
2. Cloudflare Pages detects changes
3. Builds via `astro build`
4. Deploys to edge network
5. Available at `https://opencode.ai`

**Edge Caching**:
- Static assets cached at edge
- Fast global access
- No server required (JAMstack)

---

## Content Authoring

### Writing Documentation

**Create New Page**:
```bash
# Create file
touch src/content/docs/my-new-page.md
```

**Frontmatter**:
```yaml
---
title: My New Page
description: A comprehensive guide to...
sidebar:
  order: 10
---

# My New Page

Content here...
```

**Markdown Features**:
- CommonMark syntax
- Code blocks with syntax highlighting
- Tables
- Admonitions (:::note, :::warning, :::tip)
- Component embedding

**Admonitions**:
```markdown
:::tip
Use `bun` for faster installation!
:::

:::warning
Make sure to backup your data first.
:::
```

---

## Interactive Components

### Embedding Solid.js Components

**In Markdown (.mdx)**:
```mdx
---
title: Interactive Example
---

import Demo from '@/components/Demo'

## Try it yourself

<Demo client:load />
```

**Hydration Directives**:
- `client:load`: Hydrate immediately
- `client:idle`: Hydrate when idle
- `client:visible`: Hydrate when visible
- `client:only`: Client-side only (no SSR)

---

## Search

**Pagefind Integration**:

1. **Build-time indexing**: Pagefind crawls built HTML
2. **Client-side search**: No backend required
3. **Fast & lightweight**: ~1KB per page

**Usage**:
- Search bar in header
- Keyboard shortcut: `/` or `Ctrl+K`
- Fuzzy search
- Results with context

---

## SEO & Meta Tags

**Automatic SEO**:
- Meta tags from frontmatter
- Open Graph tags for social sharing
- Twitter cards
- Canonical URLs
- Sitemap generation

**Example**:
```astro
---
// Automatically generates:
<meta name="description" content="..." />
<meta property="og:title" content="..." />
<meta property="og:description" content="..." />
<meta name="twitter:card" content="summary" />
---
```

---

## Performance

### Metrics

- **Lighthouse Score**: 100/100 (Performance, Accessibility, Best Practices, SEO)
- **First Contentful Paint (FCP)**: < 1s
- **Time to Interactive (TTI)**: < 2s
- **Total Bundle Size**: < 50KB JS

### Optimizations

1. **Static Generation**: All pages pre-rendered
2. **Code Splitting**: Per-route bundles
3. **Image Optimization**: WebP/AVIF, responsive sizes
4. **Lazy Loading**: Images and components
5. **Edge Caching**: Cloudflare CDN
6. **Minimal JavaScript**: Only for interactive islands

---

## Accessibility

- **ARIA Labels**: Proper semantic HTML
- **Keyboard Navigation**: All interactive elements accessible
- **Screen Reader Support**: Descriptive alt text, labels
- **Color Contrast**: WCAG AA compliant
- **Focus Indicators**: Visible focus states

---

## Internationalization (i18n)

**Ready for Localization**:
- Starlight supports multiple languages
- Content in `src/content/docs/{lang}/`
- Language switcher in header
- Currently: English only (future: add translations)

---

## Analytics

**Optional Integrations**:
- Google Analytics
- Plausible Analytics
- Cloudflare Web Analytics

**Privacy-Focused**:
- No cookies by default
- Anonymized data
- GDPR compliant

---

## Development Workflow

### Local Development

```bash
# Start dev server
bun dev

# Build for production
bun build

# Preview production build
bun preview
```

### Adding Content

1. Create markdown file in `src/content/docs/`
2. Add frontmatter
3. Write content
4. Test locally (`bun dev`)
5. Commit and push
6. Auto-deploys to Cloudflare

### Updating Components

1. Edit component in `src/components/`
2. Import and use in Astro/MDX file
3. Add `client:*` directive for hydration
4. Test interactivity
5. Build and deploy

---

## Dependencies

### Core
- **astro**: Static site generator
- **@astrojs/solid-js**: Solid.js integration
- **@astrojs/starlight**: Documentation theme
- **@astrojs/cloudflare**: Cloudflare Pages adapter

### Markdown & Syntax
- **@astrojs/markdown-remark**: Markdown processing
- **shiki**: Syntax highlighting
- **marked**: Markdown parser
- **rehype-autolink-headings**: Auto-link headers

### Utilities
- **luxon**: Date/time formatting
- **diff**: Text diffing (for examples)
- **remeda**: Utility functions
- **lang-map**: Language detection

### Development
- **opencode**: Workspace dependency (for generating docs)
- **typescript**: Type checking

---

## Configuration Files

### `astro.config.mjs`

```js
export default defineConfig({
  integrations: [
    solid(),
    starlight({
      title: 'opencode',
      // ...
    }),
  ],
  adapter: cloudflare(),
  output: 'static',
})
```

### `tsconfig.json`

```json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "solid-js"
  }
}
```

---

## Future Enhancements

### Planned Features
- Interactive API playground
- Video tutorials
- Community showcase
- Plugin marketplace
- Multi-language support
- Blog with RSS feed

### Performance
- Even smaller bundle sizes
- Partial hydration optimizations
- Service worker for offline docs

---

## Design System

### Colors
- **Primary**: Blue (#0066FF)
- **Secondary**: Purple (#7C3AED)
- **Success**: Green (#10B981)
- **Warning**: Yellow (#F59E0B)
- **Error**: Red (#EF4444)

### Typography
- **Heading**: System font stack
- **Body**: System font stack
- **Code**: IBM Plex Mono

### Components
- Consistent spacing (8px grid)
- Rounded corners (4px, 8px)
- Subtle shadows
- Smooth transitions

---

## Contributing to Documentation

1. Fork repository
2. Create branch: `git checkout -b docs/my-improvement`
3. Edit markdown files in `src/content/docs/`
4. Test locally: `bun dev`
5. Commit: `git commit -m "docs: improve installation guide"`
6. Push and create PR

**Documentation Style Guide**:
- Use active voice
- Keep paragraphs short (2-3 sentences)
- Include code examples
- Add screenshots for visual features
- Use admonitions for important notes