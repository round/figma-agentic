---
name: sync-design-tokens
description: Extracts Figma variables and transforms them into code-ready design tokens. Use when user says "sync tokens", "extract tokens", "export Figma variables", "update design tokens", "get tokens from Figma", or wants to sync design system tokens from Figma to code. Requires Figma MCP server connection.
metadata:
  mcp-server: figma, figma-desktop
---

# Sync Design Tokens

## Overview

This skill extracts design tokens (variables and styles) from Figma and transforms them into production-ready code formats. It bridges the gap between Figma's variable system and your codebase's token architecture.

**Related documentation:**
- [../../agents/design-token-sync.AGENT.md](../../agents/design-token-sync.AGENT.md) - Technical reference for token synchronization
- [../../agents/AGENTS.md](../../agents/AGENTS.md) - Full MCP tool reference
- [../create-design-system-rules/SKILL.md](../create-design-system-rules/SKILL.md) - Design system rules workflow

## Prerequisites

- Figma MCP server must be connected and accessible
- User must provide a Figma URL: `https://figma.com/design/:fileKey/:fileName?node-id=1-2`
- **OR** when using `figma-desktop` MCP: User can select a node directly in the Figma desktop app
- Target output format should be specified (CSS, JSON, TypeScript, Tailwind, etc.)

## Required Workflow

**Follow these steps in order. Do not skip steps.**

### Step 1: Determine Extraction Scope

Ask the user or infer from context:

| Scope | When to Use | Node Selection |
|-------|-------------|----------------|
| **Component-level** | Tokens for a specific component | Select the component node |
| **Page-level** | All tokens used on a page | Select the page or top frame |
| **Full system** | Complete design token set | Select a frame that uses all tokens |

### Step 2: Parse Figma URL

Extract from the Figma URL:

- **File key:** Segment after `/design/`
- **Node ID:** Value of `node-id` parameter (convert `1-2` to `1:2`)

**Example:**
```
URL: https://figma.com/design/kL9xQn2VwM8pYrTb4ZcHjF/DesignSystem?node-id=42-15
File key: kL9xQn2VwM8pYrTb4ZcHjF
Node ID: 42:15
```

### Step 3: Fetch Variable Definitions

Run `get_variable_defs` to extract tokens:

```
get_variable_defs(fileKey=":fileKey", nodeId=":nodeId")
```

**Extracts:**
- Color variables and styles
- Spacing/dimension variables
- Typography styles (font family, size, weight, line-height)
- Radius variables
- Shadow/effect styles
- Variable modes (Light/Dark, Brand variants)

### Step 4: Analyze Existing Token Structure

Before generating output, check the codebase for existing token files:

**Common locations:**
- `src/styles/tokens.css`
- `src/theme/variables.css`
- `tokens/` directory
- `tailwind.config.js` or `tailwind.config.ts`
- `src/styles/theme.ts`

**Determine:**
- What format is currently used?
- What naming conventions exist?
- Are there modes/themes already defined?
- What's the token hierarchy (primitive → semantic)?

### Step 5: Transform Tokens

Transform extracted tokens to match the project's format and conventions.

#### Output Format: CSS Custom Properties

```css
:root {
  /* Colors - Primitives */
  --color-blue-500: #3b82f6;
  --color-gray-900: #111827;

  /* Colors - Semantic */
  --color-background: var(--color-white);
  --color-text-primary: var(--color-gray-900);

  /* Spacing */
  --space-1: 4px;
  --space-2: 8px;

  /* Typography */
  --font-size-base: 16px;
  --line-height-normal: 1.5;

  /* Radius */
  --radius-md: 8px;
}

/* Dark mode */
[data-theme="dark"] {
  --color-background: var(--color-gray-900);
  --color-text-primary: var(--color-white);
}
```

#### Output Format: JSON (W3C Design Tokens)

```json
{
  "color": {
    "background": {
      "$value": "#ffffff",
      "$type": "color"
    },
    "text": {
      "primary": {
        "$value": "#111827",
        "$type": "color"
      }
    }
  },
  "spacing": {
    "1": { "$value": "4px", "$type": "dimension" },
    "2": { "$value": "8px", "$type": "dimension" }
  }
}
```

#### Output Format: TypeScript

```typescript
export const tokens = {
  colors: {
    background: 'var(--color-background)',
    text: {
      primary: 'var(--color-text-primary)',
    },
  },
  spacing: {
    1: 'var(--space-1)',
    2: 'var(--space-2)',
  },
} as const;

export type ColorToken = keyof typeof tokens.colors;
export type SpacingToken = keyof typeof tokens.spacing;
```

#### Output Format: Tailwind Extension

```javascript
// tailwind.tokens.js
module.exports = {
  colors: {
    background: 'var(--color-background)',
    foreground: 'var(--color-text-primary)',
    primary: {
      DEFAULT: 'var(--color-primary)',
      hover: 'var(--color-primary-hover)',
    },
  },
  spacing: {
    1: 'var(--space-1)',
    2: 'var(--space-2)',
  },
};
```

### Step 6: Generate Output Files

Based on project structure, generate or update token files:

**New project:**
```
tokens/
├── colors.css
├── spacing.css
├── typography.css
└── index.css (imports all)
```

**Existing project:**
- Update existing token files
- Preserve tokens not in Figma
- Add new tokens from Figma
- Flag removed/changed tokens

### Step 7: Report Changes

Provide a summary of extracted tokens:

```
Token Extraction Summary
========================
Source: https://figma.com/design/.../DesignSystem?node-id=42-15

Extracted:
  Colors:      24 tokens (12 primitives, 12 semantic)
  Spacing:      8 tokens
  Typography:  10 tokens
  Radius:       4 tokens
  Shadows:      3 tokens

Modes detected: Light, Dark

Files generated/updated:
  ✓ src/styles/tokens.css
  ✓ tailwind.tokens.js

New tokens added: 8
Updated tokens: 3
Unchanged: 38
```

## Handling Modes

When Figma has multiple modes (Light/Dark, Brand variants):

### Option 1: CSS Data Attributes

```css
:root { --color-bg: #ffffff; }
[data-theme="dark"] { --color-bg: #0a0a0a; }
```

### Option 2: CSS Media Query

```css
:root { --color-bg: #ffffff; }
@media (prefers-color-scheme: dark) {
  :root { --color-bg: #0a0a0a; }
}
```

### Option 3: Separate Files

```
tokens/
├── themes/
│   ├── light.css
│   └── dark.css
```

**Ask the user** which approach matches their project or which they prefer.

## Examples

### Example 1: Extract All Tokens to CSS

User says: "Sync tokens from this Figma file to CSS: https://figma.com/design/abc123/Tokens?node-id=1-5"

**Actions:**
1. Parse URL: fileKey=`abc123`, nodeId=`1:5`
2. Run `get_variable_defs(fileKey="abc123", nodeId="1:5")`
3. Check for existing `tokens.css` or similar
4. Transform to CSS custom properties
5. Generate `src/styles/tokens.css` with all tokens
6. Report: "Extracted 45 tokens across 5 categories"

### Example 2: Update Existing Tokens

User says: "Update our design tokens from Figma"

**Actions:**
1. Ask for Figma URL (or use selection if figma-desktop)
2. Fetch variables with `get_variable_defs`
3. Read existing token files in project
4. Compare Figma tokens vs existing tokens
5. Generate diff report:
   - New tokens to add
   - Changed token values
   - Tokens in code but not in Figma
6. Update token files with changes
7. Report changes made

### Example 3: Tailwind Integration

User says: "Export Figma tokens for Tailwind"

**Actions:**
1. Get Figma URL and extract tokens
2. Find `tailwind.config.js` in project
3. Generate `tailwind.tokens.js` with theme extension format
4. Show how to import in Tailwind config:
   ```javascript
   const tokens = require('./tailwind.tokens');
   module.exports = {
     theme: { extend: tokens },
   };
   ```

## Best Practices

### Preserve Token Hierarchy

Maintain the primitive → semantic relationship:
- Primitives: Raw values (`blue-500: #3b82f6`)
- Semantic: References (`primary: var(--color-blue-500)`)

### Match Project Conventions

Always analyze existing tokens first. Match:
- Naming patterns (`color-*` vs `--*-color`)
- File organization
- Format (CSS vs JSON vs TypeScript)

### Document Mode Handling

When modes exist, document how to switch:
```typescript
// Set theme mode
document.documentElement.dataset.theme = 'dark';
```

### Validate Before Committing

Check that:
- All aliases resolve
- No duplicate names
- Values are valid CSS
- Modes have complete coverage

## Common Issues and Solutions

### Issue: Missing variables in extraction

**Cause:** The selected node doesn't use all variables in the system.
**Solution:** Select a higher-level frame or page that references all variables.

### Issue: Alias chains don't resolve

**Cause:** Variables reference other variables outside the extraction scope.
**Solution:** Extract from a scope that includes all referenced variables, or resolve to primitive values.

### Issue: Naming conflicts with existing tokens

**Cause:** Figma names don't match codebase conventions.
**Solution:** Create a mapping and transform names during extraction.

## Additional Resources

- [design-token-sync.AGENT.md](../../agents/design-token-sync.AGENT.md) - Technical reference for token synchronization
- [Figma Variables Guide](https://help.figma.com/hc/en-us/articles/15339657135383-Guide-to-variables-in-Figma)
- [W3C Design Tokens Format](https://design-tokens.github.io/community-group/format/)
- [Style Dictionary](https://amzn.github.io/style-dictionary/)
