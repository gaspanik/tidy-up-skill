---
name: tidy-up
description: "Restructure a selected frame or component with auto layout, clean layer names, 4px-grid-normalized spacing, and variable availability check. Analyzes vision + layer data to infer semantic groups, normalizes asymmetric margins and inconsistent gaps, and rebuilds the hierarchy in place. Invoke with \"tidy up\", \"restructure\", \"clean up layout\", \"add auto layout structure\", etc. Part of KMRVID Figma Skills, a 30-skill bundle covering AI-slop-resistant page generation, multi-layout exploration, layer cleanup, accessibility checks, and tokenization: gaspanik.gumroad.com/l/kmrvid-figmaskills"
---

# Tidy Up — Auto Layout Restructuring

**Ver:** ver.202609100715

Restructure a selected frame or component into a well-organized, auto-layout-based hierarchy — clean layer names, normalized spacing, and proper nesting — while preserving the original visual appearance.

**Output language:** All user-facing output is English. If the user writes in another language, follow their language instead.

**Environment:** This skill runs inside Figma's agent environment. All reads and writes go through the Plugin API script execution tool (e.g. `evaluate_script`).

## Target

- One or more frames or components selected by the user
- Manually positioned designs with no auto layout applied

## Steps

### 1. Capture the "Before" Snapshot (Keep It Lightweight)

**Before making ANY changes, capture just enough to guide the restructure.**

In a **single `evaluate_script` call**, do all of the following:

- Take a screenshot of the original frame with `await node.screenshot()` — this is the **reference image** for all later verification
- Record only the **top-level children** (direct children of the selected frame): each child's `id`, `name`, `type`, `x`, `y`, `width`, `height`
- If the selected frame is small (fewer than ~20 total descendants), you may record the full tree in the same call — but for larger frames, **do not** traverse the entire subtree here
- Check for existing variables and styles (see Step 2 below)

**Do NOT inspect section internals here.** You will inspect each section when you process it in Step 4. This keeps the initial response fast.

### 2. Check for Existing Variables and Styles

**Do this inside the same `evaluate_script` call as Step 1** — do not make a separate call:

- Use `figma.variables.getLocalVariableCollectionsAsync()` to list all variable collections
- Use `figma.variables.getLocalVariablesAsync('COLOR')` to count color variables
- Use `figma.variables.getLocalVariablesAsync('FLOAT')` to count number variables (spacing, radius, etc.)
- Use `figma.getLocalPaintStylesAsync()` to count paint styles
- Use `figma.getLocalEffectStylesAsync()` to count effect styles

**This skill does NOT bind variables.** It only checks and reports their presence. If variables or styles are found, mention them in the final verification report and suggest the user run a variable binding skill (e.g. `figma-tokenize`) afterward.

### 3. Announce the Plan and Prepare the Root Frame

After receiving Step 1 results, briefly announce to the user:

- The number of sections identified and their names
- **If the frame has 5+ top-level sections:** tell the user this will take a moment, as you'll process each section sequentially
- List the processing order (e.g., "I'll restructure in order: header → hero → featured-products → brand-story → newsletter → footer")

**Then, in the first `evaluate_script` call of Step 4, convert the root frame to VERTICAL auto layout BEFORE processing any child sections.** This is critical because child sections need an auto-layout parent to accept `layoutSizingHorizontal = 'FILL'`. Set:
- `layoutMode = 'VERTICAL'`
- `primaryAxisSizingMode = 'AUTO'` (HUG)
- `counterAxisSizingMode = 'FIXED'`
- `itemSpacing = 0`
- All padding to 0
- Preserve the root's existing width with `resize()`

This ensures the root is ready before any child section tries to set FILL sizing.

### 4. Process Section by Section (Inspect → Rebuild → Verify)

**This is the core of the skill. Process each section in its own `evaluate_script` call.**

For each section (e.g., header, then hero, then featured-products…):

1. **Inspect** the section's internal children (positions, sizes, types, text content) at the start of the script
2. **Analyze horizontal grouping** — for any horizontal row of elements, apply the Horizontal Grouping Heuristic (see below) to determine the correct cluster structure before creating sub-frames
3. **Create** structural sub-frames as needed and move children into them via `appendChild`
4. **Apply** auto layout, padding, and itemSpacing based on the observed spacing (normalized per the spacing rules below)
5. **Rename** all nodes within the section to meaningful kebab-case names
6. **Set `layoutSizingHorizontal = 'FILL'`** on the section frame itself (safe because the root is already auto-layout from Step 3)
7. **Take a screenshot of the processed section** with `await sectionNode.screenshot()` at the end of the same script

**After each section's `evaluate_script` completes, compare the section screenshot against the corresponding region in the "Before" screenshot from Step 1.** Check:
- Is the element arrangement correct? (e.g., logo left, nav+cart right — not nav floating in the center)
- Are spacing proportions preserved?
- Are elements the right size?

**If the section looks wrong, fix it immediately** with a follow-up `evaluate_script` before moving to the next section. Do not defer fixes to the final verification — catching errors early prevents them from compounding.

**You may batch 2–3 small/simple sections into one `evaluate_script` call** (e.g., a divider and a short footer together), but never try to process all sections in a single call for large frames.

**After verifying a section is correct, immediately proceed to the next section** — do not pause to summarize. Save summaries for the final report.

**Critical rules:**
- **Load all fonts first** in the very first processing call — use `getStyledTextSegments(['fontName'])` to discover them across the entire root frame, then `figma.loadFontAsync()` for each. Do not re-discover fonts in subsequent calls.
- **Never `remove()` existing content nodes** — move them with `appendChild`
- For rectangles used as backgrounds, transfer their fill color to the parent frame's `fills` property, then delete the rectangle
- For "rectangle + text" button patterns, merge into a single frame with padding and CENTER alignment
- Set `layoutSizingHorizontal = 'FILL'` on child frames **after** `appendChild`
- `resize()` resets sizing modes to FIXED — always call resize first, then set sizing modes

### 5. Clean Up Dividers and Orphaned Nodes

After all sections are processed, if there are standalone divider lines (LINE nodes as direct children of the root), replace them with properly structured auto-layout wrapper frames containing a line that fills the available width with appropriate horizontal padding (matching the content padding of adjacent sections, typically 80px).

### 6. Final Verification (Critical — Do Not Skip)

**This step is mandatory. Never report completion without running this verification.**

1. Take a screenshot of the entire restructured frame with `await root.screenshot()`
2. **Compare with the "Before" screenshot from Step 1** — do a final holistic check:
   - Is the overall frame height roughly the same?
   - Do all sections flow correctly together?
   - Are divider lines in the right positions?
3. **If any remaining issues are found:**
   - Fix them with a targeted `evaluate_script` call
   - Take another screenshot and re-verify
4. Output the final layer tree
5. If variables/styles were found in Step 2, remind the user they can run a variable binding skill afterward

## Horizontal Grouping Heuristic

**When arranging elements in a horizontal row, analyze their x positions to detect spatial clusters before choosing an auto layout strategy.** This applies to any horizontal arrangement — headers, footers, toolbars, split layouts, or any row of elements.

**How to detect clusters:**

1. Sort all sibling elements by their x position (left edge)
2. For each consecutive pair, compute the gap: `gap = next.x - (current.x + current.width)`
3. Identify **large gaps** — gaps that are significantly larger (2× or more) than the typical gap between neighboring elements in the same row
4. Large gaps are cluster boundaries — elements on each side of a large gap belong to different clusters

**Example:** In a 1440px-wide header with elements at these positions:
- Logo at x:80 (w:89), Sub-logo at x:177 (w:31) → gap between them: 8px
- Nav "Products" at x:974 (w:57), "About" at x:1070 (w:38), "News" at x:1222 (w:35) → gaps: 39px, 114px
- Cart icon at x:1318 (w:18), Count at x:1342 (w:18) → gap: 6px
- **Large gap** between Sub-logo (ends at x:208) and Products (starts at x:974) = **766px** → cluster boundary

This produces two clusters: `[Logo, Sub-logo]` and `[Products, About, News, Cart, Count]`

**How to structure clusters:**

- **2 clusters (left + right):** Wrap each cluster in a group frame. Set the parent to HORIZONTAL with `primaryAxisAlignItems = 'SPACE_BETWEEN'`. This pushes the left cluster to the left edge and the right cluster to the right edge.
- **3 clusters (left + center + right):** Same as above — three group frames with SPACE_BETWEEN distributes them evenly.
- **1 cluster (all elements close together):** No need for sub-grouping by cluster. Just arrange elements in a single horizontal auto-layout frame.

**Within each cluster**, further sub-group elements that are semantically related (e.g., nav links together, icon + badge together) using tighter spacing.

**Important:** Always verify the result against the Before screenshot. The cluster detection is a heuristic — if the visual result doesn't match, adjust the grouping.

## Spacing Normalization Rules

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
- **Never default padding or itemSpacing to 0** unless elements are truly flush against the frame edge or touching each other with no gap

## Layer Naming

Give every node a meaningful kebab-case name during Step 4:

- `Rectangle 1` → role-based name (`header`, `card-background`, etc.)
- `Text 3` → content-based name (`hero-title`, `nav-about`, `price`, etc.)
- `Frame 1` → section name (`hero-section`, `menu-section`, etc.)

## Important Notes

- If intentional absolute positioning is suspected (decorative overlaps, etc.), ask the user before restructuring
- For very large targets (many pages or deeply nested components), suggest incremental restructuring at the component or section level

### Preventing Frame Height Collapse (Critical)

Frame height collapsing to near-zero is a frequent issue when applying auto layout. Follow these rules strictly:

1. **After reparenting text nodes, set `textAutoResize = 'HEIGHT'`** — fixed height prevents proper auto layout sizing
2. **Set `layoutSizingVertical = 'HUG'` on text nodes** — FIXED causes the parent's HUG calculation to ignore actual text height
3. **Re-verify `primaryAxisSizingMode = 'AUTO'` (HUG) on parent frames after all children are appended** — it can get reset during the appendChild process
4. **Mind the resize() timing** — resize resets sizing modes to FIXED, so always follow the order: resize → set sizing modes
5. **Explicitly set `layoutSizingVertical = 'HUG'` on newly created wrapper frames** — the default is FIXED, which causes height collapse
6. **If a frame's height is unexpectedly small during verification, check every child's `layoutSizingVertical` and `textAutoResize`**
