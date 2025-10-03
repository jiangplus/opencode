# Identity Package Architecture

**Package**: `packages/identity`
**Purpose**: Brand assets and identity guidelines
**Type**: Static assets

---

## Overview

The `identity` package contains **brand assets** for opencode, including logos, icons, and visual identity elements. These assets are used across all opencode properties (website, documentation, apps, marketing materials).

---

## Contents

```
packages/identity/
├── logo-light.svg           # Horizontal logo (light mode)
├── logo-dark.svg            # Horizontal logo (dark mode)
├── logo-ornate-light.svg    # Ornate logo (light mode)
├── logo-ornate-dark.svg     # Ornate logo (dark mode)
├── logo-square-light.svg    # Square logo (light mode)
├── logo-square-dark.svg     # Square logo (dark mode)
├── logomark-light.svg       # Icon/mark only (light mode)
├── logomark-dark.svg        # Icon/mark only (dark mode)
├── avatar-light.png         # Profile avatar (light mode)
└── avatar-dark.png          # Profile avatar (dark mode)
```

---

## Asset Types

### 1. Horizontal Logos
- **logo-light.svg**: Standard logo for light backgrounds
- **logo-dark.svg**: Standard logo for dark backgrounds
- **Usage**: Website headers, documentation, presentations

### 2. Ornate Logos
- **logo-ornate-light.svg**: Decorative logo for light backgrounds
- **logo-ornate-dark.svg**: Decorative logo for dark backgrounds
- **Usage**: Landing pages, hero sections, marketing

### 3. Square Logos
- **logo-square-light.svg**: Square variant for light backgrounds
- **logo-square-dark.svg**: Square variant for dark backgrounds
- **Usage**: Social media profiles, app icons

### 4. Logomarks
- **logomark-light.svg**: Icon only for light backgrounds
- **logomark-dark.svg**: Icon only for dark backgrounds
- **Usage**: Favicons, app icons, small spaces

### 5. Avatars
- **avatar-light.png**: Profile/bot avatar for light mode
- **avatar-dark.png**: Profile/bot avatar for dark mode
- **Usage**: Chat interfaces, profile pictures

---

## Brand Colors

Primary palette (from brand guidelines):

```css
--primary: #0066FF;        /* Blue */
--secondary: #7C3AED;      /* Purple */
--accent: #10B981;         /* Green */
--neutral: #6B7280;        /* Gray */
```

---

## Usage Guidelines

### Logo Clearspace
Maintain minimum clearspace of logo height on all sides

### Minimum Sizes
- Horizontal logo: 120px width minimum
- Square logo: 32px × 32px minimum
- Logomark: 16px × 16px minimum

### File Formats
- **SVG**: Preferred for web and print (scalable)
- **PNG**: Use for raster contexts (with transparency)

### Color Modes
Always use appropriate variant:
- Light backgrounds → dark version
- Dark backgrounds → light version

---

## Exporting

Assets are referenced by other packages:

```typescript
// In web package
import logo from "@opencode/identity/logo-light.svg"
```

```html
<!-- In HTML -->
<img src="/identity/logo-dark.svg" alt="opencode" />
```

---

## Updates

When updating brand assets:
1. Update source files in design tool (Figma)
2. Export as SVG/PNG
3. Optimize with SVGO
4. Commit to repository
5. Increment version if breaking changes