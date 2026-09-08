---
name: tidy-up
description: >-
  Restructure a selected frame or component with auto layout, clean layer
  names, 4px-grid-normalized spacing, and variable availability check.
  Analyzes vision + layer data to infer semantic groups, normalizes
  asymmetric margins and inconsistent gaps, and rebuilds the hierarchy in
  place. Invoke with "tidy up", "restructure", "clean up layout", "add auto
  layout structure", etc. Part of KMRVID Figma Skills, a 28-skill bundle
  covering AI-slop-resistant page generation, multi-layout exploration,
  layer cleanup, accessibility checks, and tokenization:
  gaspanik.gumroad.com/l/kmrvid-figmaskills
---

# Tidy Up — Auto Layout Restructuring

**Ver:** ver.202609082300

Restructure a selected frame or component into a well-organized, auto-layout-based hierarchy — clean layer names, normalized spacing, and proper nesting — while preserving the original visual appearance.

**Output language:** All user-facing output is English. If the user writes in another language, follow their language instead.

**Environment:** This skill runs inside Figma's agent environment. All reads and writes go through the Plugin API script execution tool (e.g. `evaluate_script`).

## Target

- One or more frames or components selected by the user
- Manually positioned designs with no auto layout applied

## Steps

### 1. Capture the "Before" State (Critical — Do This First)

**Before making ANY changes, capture and preserve the original state for comparison:**

- Take a screenshot of the original frame with `await node.screenshot()` — this is the **reference image** for all later verification
- Record the full layer tree with positions, sizes, and fills of every child element
- For each element, record its **absolute pixel position and size** within the frame — these are ground truth for spacing verification later

**Build a spacing map:** For every element, compute and store:
- Distance from frame left edge (x)
- Distance from frame right edge (parentWidth - x - width)
- Distance from previous sibling's bottom edge (gap between elements)
- Distance from frame top edge (for first element in a group)
- Distance from frame bottom edge (for last element in a group)

This spacing map is the authoritative reference for **macro-level spacing** (see "What to preserve vs. what to correct" below).

### 2. Check for Existing Variables and Styles

Scan the file for variable collections and local styles — this informs the user about what's available for later binding with a dedicated skill (e.g. `figma-tokenize`):

- Use `figma.variables.getLocalVariableCollectionsAsync()` to list all variable collections
- Use `figma.variables.getLocalVariablesAsync('COLOR')` to count color variables
- Use `figma.variables.getLocalVariablesAsync('FLOAT')` to count number variables (spacing, radius, etc.)
- Use `figma.getLocalPaintStylesAsync()` to count paint styles
- Use `figma.getLocalEffectStylesAsync()` to count effect styles

**This skill does NOT bind variables.** It only checks and reports their presence. If variables or styles are found, mention them in the final verification report and suggest the user run a variable binding skill (e.g. `figma-tokenize`) afterward.

### 3. Normalize Spacing Values

**What to preserve vs. what to correct:**

Not all spacing in the original should be faithfully reproduced. The key distinction is between **macro-level layout rhythm** (intentional) and **micro-level element misalignment** (sloppy):

**PRESERVE (macro-level spacing) — derive padding/itemSpacing from these:**
- Gaps between major sections (header → hero → card grid → footer)
- Padding within sections (content inset from section edges)
- Gaps between sibling groups (e.g., space between card row and section title)
- Overall proportions and visual rhythm of the page

**CORRECT (micro-level misalignment) — let auto layout fix these:**
- Off-center text in buttons (e.g., left 68px / right 32px → auto layout CENTER alignment fixes this automatically)
- Inconsistent left-edge alignment of elements within a card or section (e.g., title at x:36, description at x:28, price at x:28 → uniform padding corrects this)
- Slightly misaligned Y positions of sibling cards (e.g., Y 358, 360, 362 → auto layout row alignment fixes this)
- Asymmetric padding that is clearly unintentional (e.g., left 23px / right 25px → normalize to 24px)

**Inferring alignment intent from position:**
- Default to CENTER alignment — most elements in loosely built designs are intended to be centered
- Use MIN (left-align) or MAX (right-align) only when the element's position is **clearly biased** to one side — e.g., left margin is less than 30% of right margin, or vice versa
- When in doubt, choose CENTER — it's more likely to match the designer's intent than preserving a lopsided placement

**Rule of thumb:** If the spacing variation looks like someone carefully placed it (consistent across sections, specific proportions), preserve it. If it looks like someone roughly dragged elements into position (slightly off-center, inconsistent edges within a group), correct it.

**Normalization rules for preserved spacing:**
- Round to the nearest multiple of 4 (4, 8, 12, 16, 20, 24, 32, 40, 48…)
- Preserve clearly intentional values (e.g., exactly 15px or 30px)
- **Never default padding or itemSpacing to 0** unless the spacing map genuinely shows elements flush against the frame edge or touching each other with no gap

### 4. Design the Structure

Based on the visual structure observed in the screenshot, plan the auto layout tree:

- Root frame: VERTICAL (vertically stacked sections)
- Header / Footer: HORIZONTAL, SPACE_BETWEEN or CENTER
- Content sections: VERTICAL, with padding derived from Step 3
- Horizontal rows (card grids, nav items): HORIZONTAL, with itemSpacing derived from Step 3
- Individual cards: VERTICAL, CENTER-aligned
- Buttons: HORIZONTAL, CENTER-aligned (auto layout centering replaces manual text positioning)

### 5. Rebuild

**Critical: never remove() existing nodes — move them to new parent frames with appendChild.**

1. Create all structural frames first (header, sections, cards, etc.)
2. Move existing text and shape nodes into the correct structural frame via `appendChild`
3. For rectangles used as backgrounds, transfer their fill color to the parent frame's `fills` property, then delete the rectangle
4. For "rectangle + text" button patterns, merge into a single frame with padding and CENTER alignment — do not reproduce the original off-center positioning
5. Convert the root frame to auto layout last, then append the structural frames
6. Set `layoutSizingHorizontal = 'FILL'` on child frames
7. **Apply the normalized padding and itemSpacing values from Step 3 to every structural frame** — do not skip this or leave defaults

### 6. Rename Layers

Give every node a meaningful kebab-case name:

- `Rectangle 1` → role-based name (`header`, `card-background`, etc.)
- `Text 3` → content-based name (`hero-title`, `nav-about`, `price`, etc.)
- `Frame 1` → section name (`hero-section`, `menu-section`, etc.)

### 7. Verify and Adjust (Critical — Do Not Skip)

**This step is mandatory. Never report completion without running this verification.**

1. Take a screenshot of the restructured frame with `await node.screenshot()`
2. **Compare side-by-side with the "Before" screenshot from Step 1** — visually check:
   - Are section-level spacings preserved? (gaps between hero, cards, footer, etc.)
   - Are section-internal paddings preserved? (content inset from section edges)
   - Are element sizes preserved? (images, cards not squished or stretched)
   - Is the overall frame height roughly the same?
   - Have micro-level misalignments been corrected? (text centered in buttons, elements aligned within cards)
3. **If any macro-level spacing looks wrong** (sections too close together, padding missing, elements squished):
   - Re-read the spacing map from Step 1
   - Identify which padding or itemSpacing values are incorrect
   - Fix them with a targeted `evaluate_script` call
   - Take another screenshot and re-verify
4. **Repeat the fix-and-verify cycle** until the result visually matches the original at the macro level while being cleaner at the micro level
5. Output the final layer tree
6. If variables/styles were found in Step 2, remind the user they can run a variable binding skill afterward

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
