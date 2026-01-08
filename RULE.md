---
description: Figma MCP server rules
globs:
alwaysApply: true
---
  - The Figma MCP Server provides an assets endpoint which can serve image and SVG assets
  - IMPORTANT: If the Figma MCP Server returns a localhost source for an image or an SVG, use that image or SVG source directly
  - IMPORTANT: DO NOT import/add new icon packages, all the assets should be in the Figma payload
  - IMPORTANT: do NOT use or create placeholders if a localhost source is provided


## Figma MCP Integration Rules

These rules define how to translate Figma inputs into code for this project and must be followed for every Figma-driven change.

- IMPORTANT: Always use components from `/path_to_your_design_system` when possible
- Prioritize Figma fidelity to match designs exactly
- Avoid hardcoded values, use design tokens from Figma where available
- Place UI components in `/path_to_your_design_system`; avoid inline styles unless truly necessary


### Required flow (do not skip)

1. Run get_design_context first to fetch the structured representation for the exact node(s).
2. If the response is too large or truncated, run get_metadata to get the high‑level node map and then re‑fetch only the required node(s) with get_design_context.
3. Run get_screenshot for a visual reference of the node variant being implemented.
4. Only after you have both get_design_context and get_screenshot, download any assets needed and start implementation.
5. Translate the output (usually React + Tailwind) into this project's conventions, styles and framework.  Reuse the project's color tokens, components, and typography wherever possible.
6. Validate against Figma for 1:1 look and behavior before marking complete.


## Figma Design Implementation

When given Figma links via MCP tools, treat designs as **pixel-perfect specifications** that must be meticulously recreated.

- Treat the Figma MCP output (React + Tailwind) as a representation of design and behavior, not as final code style.
- Replace Tailwind utility classes with the project's preferred utilities/design‑system tokens when applicable.
- Reuse existing components (e.g., buttons, inputs, typography, icon wrappers) instead of duplicating functionality.
- Use the project's color system, typography scale, and spacing tokens consistently.
- Respect existing routing, state management, and data‑fetch patterns already adopted in the repo.
- Strive for 1:1 visual parity with the Figma design. When conflicts arise, prefer design‑system tokens and adjust spacing or sizes minimally to match visuals.
- Validate the final UI against the Figma screenshot for both look and behavior.


### Core Principles

1. **No assumptions** - Every padding, margin, border-radius, font-size, and color must come from the Figma data
2. **Nested components matter** - Don't just look at the top-level; drill into every nested component to extract exact values
3. **Use established tokens** - Map Figma values to existing CSS variables and design tokens (`var(--text-2)`, `var(--muted)`, etc.)
4. **Screenshots are reference, not source** - Use `get_screenshot` to verify, but extract actual values from `get_design_context`


### Required Workflow for Figma Links

1. **First:** Call `get_design_context` with the node ID to get structured component data
2. **Examine nested components:** Look for all nested frames, text nodes, and instances - each has its own styling
3. **Extract exact values:** Note specific padding (`px-[6px]` not `px-2`), heights (`h-[24px]`), border-radius (`rounded-full` vs `rounded-[8px]`)
4. **Map to tokens:** Convert Figma variables to existing CSS custom properties
5. **Verify:** Call `get_screenshot` to visually confirm implementation matches design


### Common Mistakes to Avoid

❌ **Don't approximate values**
```typescript
// BAD - guessing based on visual
className="p-2 rounded-lg"

// GOOD - exact values from Figma
className="px-[6px] py-[6px] rounded-full"
```

❌ **Don't ignore nested component styles**
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

❌ **Don't skip hover/active states**
- Check if Figma has multiple variants (Idle, Hover, Active, Disabled)
- Each state may have different colors, backgrounds, or opacity

### What to Extract from Figma

- **Layout:** `flex`, `gap`, `padding`, `margin`, exact pixel values
- **Sizing:** Fixed `width`/`height` in pixels, or `fill_container`/`hug_contents`
- **Typography:** Font family, size, weight, line-height, letter-spacing, color
- **Colors:** Background fills, text colors, border colors (map to CSS variables)
- **Effects:** Border radius, shadows, blur, opacity
- **States:** Hover backgrounds, active states, disabled opacity

### Example: Implementing a Status Bar from Figma

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

## Error Recovery

### Model Failures
```typescript
try {
  for await (const chunk of provider.stream(prompt, config)) {
    updateBlock(blockId, chunk);
  }
} catch (error) {
  // Show error in block UI with retry button
  // Preserve user prompt and context
  // Log error details (sanitize tokens/secrets)
  showBlockError(blockId, error.message, { retry: () => retryGeneration() });
}
```