---
description: Figma MCP server rules for design-to-code translation
alwaysApply: true
---

# Figma MCP Integration Rules

These rules define how to translate Figma inputs into code and must be followed for every Figma-driven change.

**Related documentation:**
- [agents.md](agents.md) - Full tool reference and agent capabilities
- [skills/implement-design/skill.md](skills/implement-design/skill.md) - Detailed implementation workflow
- [skills/code-connect-components/skill.md](skills/code-connect-components/skill.md) - Code Connect workflow
- [skills/create-design-system-rules/skill.md](skills/create-design-system-rules/skill.md) - Creating project-specific rules

---

## Asset Handling

- The Figma MCP Server provides an assets endpoint which can serve image and SVG assets
- **IMPORTANT:** If the Figma MCP Server returns a localhost source for an image or SVG, use that source directly
- **IMPORTANT:** DO NOT import/add new icon packages - all assets should come from the Figma payload
- **IMPORTANT:** DO NOT use or create placeholders if a localhost source is provided

---

## Required Workflow

**Follow these steps in order. Do not skip steps.**

1. **Fetch context:** Run `get_design_context` first to get the structured representation for the exact node(s)
2. **Handle large designs:** If the response is too large or truncated, run `get_metadata` to get the high-level node map, then re-fetch only the required node(s) with `get_design_context`
3. **Capture visual reference:** Run `get_screenshot` for a visual reference of the node variant being implemented
4. **Download assets:** Only after you have both `get_design_context` and `get_screenshot`, download any assets needed
5. **Implement:** Translate the output (usually React + Tailwind) into this project's conventions, styles, and framework
6. **Validate:** Compare against Figma for 1:1 look and behavior before marking complete

---

## Design Implementation Principles

When given Figma links via MCP tools, treat designs as **pixel-perfect specifications** that must be meticulously recreated.

### Core Principles

1. **No assumptions** - Every padding, margin, border-radius, font-size, and color must come from the Figma data
2. **Nested components matter** - Don't just look at the top-level; drill into every nested component to extract exact values
3. **Use established tokens** - Map Figma values to existing CSS variables and design tokens
4. **Screenshots are reference, not source** - Use `get_screenshot` to verify, but extract actual values from `get_design_context`

### Translation Rules

- Treat the Figma MCP output (React + Tailwind) as a representation of design and behavior, not as final code style
- Replace Tailwind utility classes with the project's preferred utilities/design-system tokens when applicable
- Reuse existing components (buttons, inputs, typography, icon wrappers) instead of duplicating functionality
- Use the project's color system, typography scale, and spacing tokens consistently
- Respect existing routing, state management, and data-fetch patterns already adopted in the repo
- Strive for 1:1 visual parity with the Figma design
- Validate the final UI against the Figma screenshot for both look and behavior

---

## What to Extract from Figma

| Category | Properties |
|----------|------------|
| **Layout** | `flex`, `gap`, `padding`, `margin`, exact pixel values |
| **Sizing** | Fixed `width`/`height` in pixels, or `fill_container`/`hug_contents` |
| **Typography** | Font family, size, weight, line-height, letter-spacing, color |
| **Colors** | Background fills, text colors, border colors (map to CSS variables) |
| **Effects** | Border radius, shadows, blur, opacity |
| **States** | Hover backgrounds, active states, disabled opacity |

---

## Common Mistakes to Avoid

### Don't approximate values

```typescript
// BAD - guessing based on visual
className="p-2 rounded-lg"

// GOOD - exact values from Figma
className="px-[6px] py-[6px] rounded-full"
```

### Don't ignore nested component styles

```typescript
// BAD - only styling the container
<div className="flex gap-4">
  <Button>Cancel</Button>
  <Button>Continue</Button>
</div>

// GOOD - each element has precise styles from Figma
<div className="flex gap-[4px]">
  <div className="h-[24px] px-[6px] py-[6px] rounded-full">Cancel</div>
  <div className="h-[24px] px-[6px] py-[6px] rounded-full bg-white">Continue</div>
</div>
```

### Don't skip hover/active states

- Check if Figma has multiple variants (Idle, Hover, Active, Disabled)
- Each state may have different colors, backgrounds, or opacity

---

## Example: Implementing a Component from Figma

```typescript
// 1. Get design context from Figma MCP
// 2. Extract from nested "Status" container:
//    - height: 32px
//    - padding: pl-8px pr-4px
//    - border-radius: 9999px (rounded-full)
//    - hover: bg-[var(--muted)]

// 3. Extract from nested "Cancel Button":
//    - height: 24px
//    - padding: px-6px py-6px
//    - border-radius: 9999px
//    - text: 12px, color var(--text-2)

// 4. Extract from nested "Continue Button":
//    - height: 24px
//    - padding: px-6px py-6px
//    - border-radius: 9999px
//    - background: white
//    - text: 12px, color black

// 5. Implement with exact values
<div className="h-[32px] pl-[8px] pr-[4px] rounded-full hover:bg-[var(--muted)]">
  <div className="h-[24px] px-[6px] py-[6px] rounded-full">
    <span className="text-[12px] text-[var(--text-2)]">Cancel</span>
  </div>
  <div className="h-[24px] px-[6px] py-[6px] rounded-full bg-white">
    <span className="text-[12px] text-black">Continue</span>
  </div>
</div>
```

---

## Project-Specific Configuration

To customize these rules for your project, use the [/create-design-system-rules](skills/create-design-system-rules/skill.md) skill to generate project-specific conventions including:

- Component directory paths
- Design token locations
- Styling approach (Tailwind, CSS Modules, styled-components, etc.)
- Naming conventions
- Import patterns
