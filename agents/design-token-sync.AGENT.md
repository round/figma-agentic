---
name: design-token-sync
description: Technical reference for token extraction, comparison algorithms, and output formats. Used by the sync-design-tokens skill.
model: inherit
tools:
  - Glob
  - Grep
  - Read
  - Edit
  - Write
  - Bash
  - figma:get_variable_defs
  - figma:get_design_context
---

# Design Token Sync — Technical Reference

This document provides the technical specifications for token synchronization. For the user-facing workflow, see [sync-design-tokens skill](../skills/sync-design-tokens/SKILL.md).

---

## Token Categories

| Category | Figma Source | Code Equivalent |
|----------|--------------|-----------------|
| **Colors** | Color variables, fill styles | CSS `--color-*`, Tailwind `colors` |
| **Spacing** | Number variables | CSS `--space-*`, Tailwind `spacing` |
| **Typography** | Text styles, font variables | CSS `--font-*`, Tailwind `fontSize` |
| **Radii** | Number variables | CSS `--radius-*`, Tailwind `borderRadius` |
| **Shadows** | Effect styles | CSS `--shadow-*`, Tailwind `boxShadow` |
| **Breakpoints** | Number variables | CSS media queries, Tailwind `screens` |

---

## Common Token Locations

### File Patterns to Scan

```bash
# CSS Custom Properties
**/tokens.css
**/variables.css
**/theme.css
**/*.css  # search for :root { --token-name: value }

# Tailwind
tailwind.config.{js,ts,mjs}

# JavaScript/TypeScript
**/tokens.{js,ts}
**/theme.{js,ts}
**/design-tokens.{js,ts}
**/constants/colors.{js,ts}
**/styles/tokens/**

# JSON (Style Dictionary)
**/tokens.json
**/*.tokens.json

# SCSS/Sass
**/_variables.scss
**/_tokens.scss
```

### Format Detection

| Format | Pattern | Example |
|--------|---------|---------|
| **CSS Variables** | `--{category}-{name}` | `--color-primary-500` |
| **Tailwind** | Nested JS object | `colors: { primary: { 500: '#...' } }` |
| **Style Dictionary** | JSON with `$value` | `{ "color": { "$value": "#..." } }` |
| **JS Constants** | Exported object | `export const colors = { primary500: '#...' }` |
| **SCSS** | `$` prefix | `$color-primary-500: #...` |

---

## Comparison Algorithm

### Step 1: Normalize Names

Convert all token names to canonical `{category}-{variant}-{scale}` format:

| Source | Original | Canonical |
|--------|----------|-----------|
| Figma | `Colors/Primary/500` | `color-primary-500` |
| CSS | `--color-primary-500` | `color-primary-500` |
| Tailwind | `colors.primary.500` | `color-primary-500` |
| JS | `colorPrimary500` | `color-primary-500` |
| SCSS | `$color-primary-500` | `color-primary-500` |

**Transformation rules:**
- Strip prefixes: `--`, `$`
- Convert `/` and `.` to `-`
- Convert camelCase to kebab-case
- Lowercase everything

### Step 2: Normalize Values

| Type | Input Variants | Normalized |
|------|----------------|------------|
| **Colors** | `#3B82F6`, `rgb(59,130,246)`, `hsl(217,91%,60%)` | `#3B82F6` (uppercase hex) |
| **Spacing** | `16px`, `1rem`, `4` | `16` (unitless, assume px) |
| **Font size** | `1.25rem`, `20px` | `20` (px, assume 16px base) |
| **Opacity** | `0.5`, `50%` | `0.5` (decimal) |

**Color normalization:**
```
rgb(r, g, b) → #RRGGBB
hsl(h, s%, l%) → convert to RGB → #RRGGBB
rgba(r, g, b, a) → #RRGGBBAA
```

### Step 3: Categorize

| Status | Condition |
|--------|-----------|
| **Match** | Name exists in both, normalized values equal |
| **Drift** | Name exists in both, normalized values differ |
| **Figma Only** | Name in Figma, not in code |
| **Code Only** | Name in code, not in Figma |
| **Renamed** | Same value, different name (flag for review) |

---

## Report Generation

### Summary Format

```markdown
# Token Sync Report

**Generated:** [ISO timestamp]
**Figma:** [file name] ([fileKey]) / [node name] ([nodeId])
**Code:** [list of scanned files]

## Summary

| Category | Match | Drift | Figma Only | Code Only |
|----------|-------|-------|------------|-----------|
| Colors   | 24    | 3     | 2          | 1         |
| Spacing  | 8     | 0     | 1          | 0         |
| ...      | ...   | ...   | ...        | ...       |
| **Total**| **X** | **Y** | **Z**      | **W**     |
```

### Drift Detail Format

```markdown
### `color-primary-500` — DRIFT

| Source | Value | Location |
|--------|-------|----------|
| Figma  | #3B82F6 | Colors/Primary/500 |
| Code   | #2563EB | src/styles/tokens.css:15 |

**Delta:** Hue 217° → 225° (+8°)
**Action:** Update code to match Figma
```

### Missing Token Format

```markdown
### Missing in Code

| Token | Figma Value | Suggested |
|-------|-------------|-----------|
| `color-accent-400` | #FBBF24 | `--color-accent-400: #FBBF24;` |
```

---

## Update Patch Generation

### CSS Variables

```css
/* Token Sync Patch
 * Source: [Figma file] ([fileKey])
 * Generated: [timestamp]
 */

:root {
  /* UPDATED */
  --color-primary-500: #3B82F6; /* was: #2563EB */

  /* NEW */
  --color-accent-400: #FBBF24;
}
```

### Tailwind Config

```javascript
// Token Sync Patch — add to theme.extend
{
  colors: {
    primary: {
      500: '#3B82F6', // updated
    },
    accent: {
      400: '#FBBF24', // new
    },
  },
}
```

### Style Dictionary JSON

```json
{
  "color": {
    "primary": {
      "500": {
        "$value": "#3B82F6",
        "$description": "Synced from Figma"
      }
    }
  }
}
```

### SCSS Variables

```scss
// Token Sync Patch
$color-primary-500: #3B82F6; // updated
$color-accent-400: #FBBF24; // new
```

---

## Theme/Mode Handling

### Figma Variable Modes

Figma variables can have multiple modes (e.g., Light, Dark). Extract mode values:

```
Variable: Colors/Primary/500
  Mode "Light": #3B82F6
  Mode "Dark": #60A5FA
```

### Code Theme Mapping

| Figma Mode | CSS Pattern | Tailwind |
|------------|-------------|----------|
| Light | `:root { }` | Default values |
| Dark | `[data-theme="dark"] { }` or `.dark { }` | `darkMode: 'class'` |

### Theme-Aware Patch

```css
:root {
  --color-primary-500: #3B82F6;
}

[data-theme="dark"] {
  --color-primary-500: #60A5FA;
}
```

---

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `No variables found` | Node doesn't use variables | Select node with variable references |
| `Parse error` | Unknown token format | Ask user to describe format |
| `Ambiguous match` | Multiple tokens have same normalized name | Ask user to resolve |
| `Mode mismatch` | Figma has modes, code doesn't | Create theme structure in code |
| `Value tolerance` | Colors differ by rounding | Flag as "close match" if ΔE < 1 |

---

## Sync Metadata

Embed in token files for tracking:

```css
/*
 * Design Tokens
 * Synced: 2025-01-08T14:30:00Z
 * Figma: DesignSystem (kL9xQn2VwM8pYrTb4ZcHjF)
 * Node: Tokens (1:5)
 * Hash: a3f8b2c1
 */
```

Hash = first 8 chars of SHA256 of normalized token JSON, for quick drift detection.
