---
name: tidy-up
description: >-
  Restructure a selected frame or component with auto layout, clean layer
  names, 4px-grid-normalized spacing, and optional variable binding.
  Analyzes vision + layer data to infer semantic groups, normalizes
  asymmetric margins and inconsistent gaps, and rebuilds the hierarchy in
  place. Invoke with "tidy up", "restructure", "clean up layout", "add auto
  layout structure", etc. Part of KMRVID Figma Skills, a 28-skill bundle
  covering AI-slop-resistant page generation, multi-layout exploration,
  layer cleanup, accessibility checks, and tokenization:
  gaspanik.gumroad.com/l/kmrvid-figmaskills
---

# Tidy Up — Auto Layout Restructuring

**Ver:** ver.202609071830

Restructure a selected frame or component into a well-organized, auto-layout-based hierarchy — clean layer names, normalized spacing, and proper nesting — while preserving the original visual appearance.

**Output language:** All user-facing output is English. If the user writes in another language, follow their language instead.

**Environment:** This skill runs inside Figma's agent environment. All reads and writes go through the Plugin API script execution tool (e.g. `evaluate_script`).

## Target

- One or more frames or components selected by the user
- Manually positioned designs with no auto layout applied

## Steps

### 1. Analyze the Current State (Vision + Data + Variables)

- Capture a screenshot with `await node.screenshot()` to visually understand the overall structure
- Use the `buildTree` pattern to inspect the layer tree — get each element's type, position, size, fills, and text content
- Cross-reference the screenshot with layer data to determine:
  - Which elements belong to semantic groups (header, section, card, footer, etc.)
  - Whether there are repeating patterns (card grids, list items, etc.)
  - Whether absolute positioning is intentional (decorative overlaps) or just manual placement

**Check for existing variables and styles in the file:**

Before restructuring, scan the file for variable collections and local styles that can be bound to nodes:

- Use `figma.variables.getLocalVariableCollectionsAsync()` to list all variable collections
- Use `figma.variables.getLocalVariablesAsync('COLOR')` to find color variables
- Use `figma.variables.getLocalVariablesAsync('FLOAT')` to find number variables (spacing, radius, etc.)
- Use `figma.getLocalPaintStylesAsync()` to find paint styles
- Use `figma.getLocalEffectStylesAsync()` to find effect styles
- Build a lookup map of available variables by value (e.g., map hex colors to color variables, spacing values to number variables) for use during the rebuild step
- If no variables or styles exist, proceed without binding — this step is opportunistic, not required

**If variables or styles ARE found, ask the user before proceeding:**

Present the user with a summary of what was found (e.g., "I found 12 color variables and 5 spacing variables in this file") and ask which variable binding strategy they prefer:

1. **Bind to nearest match** — When a node's value is close to an existing variable (within tolerance: ±2/255 per color channel, ±2px for numbers), bind to that variable. The variable's canonical value replaces the original. Best for aligning with an established design system.
2. **Create new variables** — Create new variables for values that don't match any existing ones, and bind them. Best for preserving exact original values while making them manageable. Name new variables based on their semantic role (e.g., `card-bg`, `section-padding`).
3. **Skip binding** — Leave all values hardcoded. Best for quick mockups or temporary designs where variable management isn't needed yet.

Proceed with the user's chosen strategy. If the user doesn't have a preference, default to option 1 (bind to nearest match).

### 2. Record and Normalize Original Spacing

**Before restructuring, always record the original margins and spacing.** These must be accurately reflected as auto layout padding and itemSpacing.

- Calculate top/bottom/left/right margins from each element's position (x, y) within its parent frame
- Calculate gaps between elements (e.g., next element's y − current element's (y + height))
- Identify whether elements have surrounding whitespace (→ parent frame padding) or sit flush against edges
- Distinguish intentional margins from edge-to-edge placement

Example: if an image is at (25, 19) inside a 378px-wide frame, that indicates ~25px horizontal and 19px top padding — preserve these as parent frame padding.

**Spacing normalization (important):**

Loosely built designs often have asymmetric margins and inconsistent gaps. Normalize to clean values using these rules:

- **Asymmetric left/right margins** (e.g., left 23px / right 25px) → round to the nearest multiple of 4 (→ 24px)
- **Inconsistent gaps** (e.g., 12px, 14px, 16px mixed) → unify to the most common value or the nearest multiple of 4
- **Slightly misaligned Y positions** (e.g., cards at Y 358, 360, 362) → auto layout handles alignment automatically
- **Use a 4px grid** as the rounding base (4, 8, 12, 16, 20, 24, 32, 40, 48…). Round fractional values to the nearest multiple of 4
- **Preserve clearly intentional values** (e.g., exactly 15px or 30px) — don't force them onto the 4px grid if they look deliberate

### 3. Design the Structure

Based on the visual structure observed in the screenshot, plan the auto layout tree:

- Root frame: VERTICAL (vertically stacked sections)
- Header / Footer: HORIZONTAL, SPACE_BETWEEN or CENTER
- Content sections: VERTICAL, with appropriate padding
- Horizontal rows (card grids, nav items): HORIZONTAL, uniform itemSpacing
- Individual cards: VERTICAL, CENTER-aligned

### 4. Rebuild

**Critical: never remove() existing nodes — move them to new parent frames with appendChild.**

1. Create all structural frames first (header, sections, cards, etc.)
2. Move existing text and shape nodes into the correct structural frame via `appendChild`
3. For rectangles used as backgrounds, transfer their fill color to the parent frame's `fills` property, then delete the rectangle
4. For "rectangle + text" button patterns, merge into a single frame with padding
5. Convert the root frame to auto layout last, then append the structural frames
6. Set `layoutSizingHorizontal = 'FILL'` on child frames

### 5. Bind Variables and Styles

Apply the variable binding strategy chosen by the user in Step 1:

**If "Bind to nearest match":**
- **Color variables:** When a node's fill color is within tolerance of a color variable's value, bind using `figma.variables.setBoundVariableForPaint(paint, 'color', variable)`
- **Spacing variables:** When padding or itemSpacing values match (±2px) a number variable, bind using `setBoundVariable('paddingTop', variable)`, `setBoundVariable('itemSpacing', variable)`, etc.
- **Corner radius variables:** When a cornerRadius matches a number variable, bind using `setBoundVariable('topLeftRadius', variable)`, etc.
- **Paint styles:** When a fill matches a local paint style, apply with `node.fillStyleId = styleId`
- **Effect styles:** When effects match a local effect style, apply with `node.effectStyleId = styleId`
- **Tolerance:** ±2/255 per channel for colors, ±2px for numbers
- If no match is found within tolerance, leave as hardcoded

**If "Create new variables":**
- For each unique hardcoded value that has no existing match, create a new variable in an appropriate collection
- Name variables semantically based on their role (e.g., `card-bg`, `header-padding`, `button-radius`)
- Bind the new variable to all nodes using that value
- If a collection doesn't exist yet, create one (e.g., "Colors", "Spacing")

**If "Skip binding":**
- Leave all values hardcoded — no variable operations

### 6. Rename Layers

Give every node a meaningful kebab-case name:

- `Rectangle 1` → role-based name (`header`, `card-background`, etc.)
- `Text 3` → content-based name (`hero-title`, `nav-about`, `price`, etc.)
- `Frame 1` → section name (`hero-section`, `menu-section`, etc.)

### 7. Verify

- Capture a screenshot of the restructured frame with `await node.screenshot()`
- Compare against the original to confirm **visual appearance is preserved** (especially margins and spacing)
- Output the layer tree to verify structure is correct
- Report how many variables/styles were bound (if any), and which strategy was used
- Fix any issues found

## Important Notes

- Load all fonts before any operation (`getStyledTextSegments` to discover them)
- Fonts must also be loaded before reparenting text nodes
- `resize()` resets sizing modes to FIXED — always call resize first, then set sizing modes
- `layoutSizingHorizontal = 'FILL'` must be set after `appendChild`
- If intentional absolute positioning is suspected (decorative overlaps, etc.), ask the user before restructuring
- For very large targets, suggest incremental restructuring at the component or section level

### Preventing Frame Height Collapse (Critical)

Frame height collapsing to near-zero is a frequent issue when applying auto layout. Follow these rules strictly:

1. **After reparenting text nodes, set `textAutoResize = 'HEIGHT'`** — fixed height prevents proper auto layout sizing
2. **Set `layoutSizingVertical = 'HUG'` on text nodes** — FIXED causes the parent's HUG calculation to ignore actual text height
3. **Re-verify `primaryAxisSizingMode = 'AUTO'` (HUG) on parent frames after all children are appended** — it can get reset during the appendChild process
4. **Mind the resize() timing** — resize resets sizing modes to FIXED, so always follow the order: resize → set sizing modes
5. **Explicitly set `layoutSizingVertical = 'HUG'` on newly created wrapper frames** — the default is FIXED, which causes height collapse
6. **If a frame's height is unexpectedly small during verification, check every child's `layoutSizingVertical` and `textAutoResize`**
