# Plot Visibility Review: Fill() Conditional Logic Analysis

## Executive Summary

**Critical Finding**: The `fill()` calls for shading areas between plot lines are NOT respecting user visibility controls, while the `plot()` calls themselves have no conditional display logic either.

**Impact**: Users cannot control the visibility of filled areas (Value Area shading, StdDev shading) independently from the lines, and there are no checkbox inputs to control ANY plot visibility in the current implementation.

---

## Current Implementation Analysis

### Plot Definitions (Lines 687-739)

The indicator creates 6 plot series:

1. **vaUpPl** (Line 687-693): Upper Value Area boundary
2. **vaLoPl** (Line 694-700): Lower Value Area boundary
3. **poc** (Line 706-712): Point of Control
4. **sgmUpPl** (Line 714-720): Upper StdDev (+1σ)
5. **sgmLoPl** (Line 721-727): Lower StdDev (-1σ)
6. **vwap** (Line 733-739): Volume Weighted Average Price

### Fill Definitions

Two `fill()` calls create shaded regions:

```pine
// Lines 701-705: Value Area Fill
fill(
  plot1 = vaUpPl,
  plot2 = vaLoPl,
  title = "Value Area",
  color = color.new(color = #86549b, transp = 75))

// Lines 728-732: StdDev Area Fill
fill(
  plot1 = sgmLoPl,
  plot2 = sgmUpPl,
  title = "2 StdDev Area",
  color = color.new(color = #b6a72b, transp = 75))
```

---

## Pine Script Best Practices for Conditional Visibility

### 1. Proper Conditional Fill Pattern

**Question**: When plot() has conditional display, should fill() between plots also be conditional?

**Answer**: **YES, ALWAYS.** Fill visibility MUST follow the same conditional logic as its associated plots.

**Correct Pattern**:

```pine
// Method 1: Conditional color in fill() - RECOMMENDED
bool showVA = input.bool(true, "Show Value Area", group="Display")

vaUpPl = plot(...)
vaLoPl = plot(...)

fill(
  plot1 = vaUpPl,
  plot2 = vaLoPl,
  color = showVA ? color.new(#86549b, 75) : na,  // 👈 Conditional color
  title = "Value Area")

// Method 2: Conditional plot series - ALSO VALID
vaUpPl = plot(showVA ? vaUp[offset] : na, ...)
vaLoPl = plot(showVA ? vaLo[offset] : na, ...)
fill(vaUpPl, vaLoPl, color = color.new(#86549b, 75))
```

### 2. Why Current Implementation Fails

**Problem 1: No Input Controls**
- Missing checkbox inputs in the "Display" group (gD) for:
  - Show Value Area
  - Show Point of Control
  - Show VWAP
  - Show StdDev Bands

**Problem 2: No Conditional Logic**
- Plots always render when data exists (`vaUp[offset]`, `poc[offset]`, etc.)
- Fills always render when plot pairs exist
- No `na` conditions based on user preferences

**Problem 3: Rectangle Frame Exception**
- Rectangle frame HAS proper conditional logic (line 638: `if showRect`)
- Shows the CORRECT pattern was understood but not applied to plots/fills

---

## Root Cause Analysis

### Evidence of Design Intent

The presence of:
1. Grouped inputs (gD = "Display" group exists, lines 258-277)
2. Proper rectangle checkbox implementation (showRect, line 291-295)
3. Detailed tooltip documentation patterns

**Suggests**: The developer INTENDED to add visibility controls but either:
- Never completed the implementation
- Lost code during refactoring
- Deprioritized for initial release

### Code Evolution Hypothesis

**Likely Timeline**:
```
Version 0.0.x → Plots without controls (current state)
Version 0.1.0 → Rectangle added WITH controls (line 291)
Version 0.2.0 → Plot controls planned but not implemented?
```

The rectangle controls (lines 290-332) show maturity in design thinking that's absent from plot controls, suggesting they were added AFTER the plotting code was written.

---

## Recommended Fix Implementation

### 1. Add Input Controls (Insert after line 277)

```pine
const string gS = "statistical lines"

bool showVA = input.bool(
  title   = "Show Value Area",
  group   = gS,
  defval  = true,
  tooltip = "Display the Value Area (VA) bands and shaded region. The VA represents the price range where X% of volume occurred (set by 'Value Area' parameter).")

bool showPOC = input.bool(
  title   = "Show Point of Control",
  group   = gS,
  defval  = true,
  tooltip = "Display the Point of Control (POC) line, which marks the price level with the highest volume.")

bool showVWAP = input.bool(
  title   = "Show VWAP",
  group   = gS,
  defval  = true,
  tooltip = "Display the Volume Weighted Average Price (VWAP) line.")

bool showStdDev = input.bool(
  title   = "Show StdDev Bands",
  group   = gS,
  defval  = true,
  tooltip = "Display the Standard Deviation bands (±1σ) around VWAP and shaded region.")
```

### 2. Apply Conditional Logic to Plots (Replace lines 687-739)

```pine
// Value Area Plots - Conditional on showVA
vaUpPl = plot(
  series    = showVA ? vaUp[offset] : na,  // 👈 Conditional series
  title     = "Developing VA Up",
  color     = color.new(color = #86549b, transp = 50),
  style     = plot.style_stepline,
  linewidth = 1,
  offset    = -pivRi,
  display   = showVA ? display.all : display.none)  // 👈 Display control

vaLoPl = plot(
  series    = showVA ? vaLo[offset] : na,  // 👈 Conditional series
  title     = "Developing VA Low",
  color     = color.new(color = #86549b, transp = 50),
  style     = plot.style_stepline,
  linewidth = 1,
  offset    = -pivRi,
  display   = showVA ? display.all : display.none)  // 👈 Display control

// Fill - Conditional color based on showVA
fill(
  plot1 = vaUpPl,
  plot2 = vaLoPl,
  title = "Value Area",
  color = showVA ? color.new(color = #86549b, transp = 75) : na)  // 👈 Conditional color

// POC Plot - Conditional on showPOC
plot(
  series    = showPOC ? poc[offset] : na,  // 👈 Conditional series
  title     = "Developing POC",
  color     = color.new(color = #b8b8b8, transp = 50),
  style     = plot.style_stepline,
  linewidth = 1,
  offset    = -pivRi,
  display   = showPOC ? display.all : display.none)  // 👈 Display control

// StdDev Plots - Conditional on showStdDev
sgmUpPl = plot(
  series    = showStdDev ? (vwap[offset] + stdDev[offset]) : na,  // 👈 Conditional series
  title     = "Developing +StdDev",
  color     = color.new(color = #b6a72b, transp = 50),
  style     = plot.style_line,
  linewidth = 1,
  offset    = -pivRi,
  display   = showStdDev ? display.all : display.none)  // 👈 Display control

sgmLoPl = plot(
  series    = showStdDev ? (vwap[offset] - stdDev[offset]) : na,  // 👈 Conditional series
  title     = "Developing -StdDev",
  color     = color.new(color = #b6a72b, transp = 50),
  style     = plot.style_line,
  linewidth = 1,
  offset    = -pivRi,
  display   = showStdDev ? display.all : display.none)  // 👈 Display control

// Fill - Conditional color based on showStdDev
fill(
  plot1 = sgmLoPl,
  plot2 = sgmUpPl,
  title = "2 StdDev Area",
  color = showStdDev ? color.new(color = #b6a72b, transp = 75) : na)  // 👈 Conditional color

// VWAP Plot - Conditional on showVWAP
plot(
  series    = showVWAP ? vwap[offset] : na,  // 👈 Conditional series
  title     = "Developing VWAP",
  color     = color.new(color = #c67600, transp = 50),
  style     = plot.style_line,
  linewidth = 1,
  offset    = -pivRi,
  display   = showVWAP ? display.all : display.none)  // 👈 Display control
```

---

## Alternative Approach: Display Parameter Only

If you want to keep plots calculating (for alerts) but hide visually:

```pine
vaUpPl = plot(
  series    = vaUp[offset],
  title     = "Developing VA Up",
  color     = color.new(color = #86549b, transp = 50),
  style     = plot.style_stepline,
  linewidth = 1,
  offset    = -pivRi,
  display   = showVA ? display.all : display.none)  // 👈 Only visual control

fill(
  plot1 = vaUpPl,
  plot2 = vaLoPl,
  title = "Value Area",
  color = showVA ? color.new(color = #86549b, transp = 75) : na)  // 👈 Still need conditional color
```

**Advantage**: Alerts (lines 748-796) continue to fire even when plots are hidden.

**Disadvantage**: Slightly more computation overhead.

---

## Pine Script Technical Details

### Fill() Behavior Rules

1. **Fill requires valid plot references**: Both plot1 and plot2 must exist
2. **Color na = invisible fill**: Setting `color = na` hides the fill completely
3. **Fill inherits plot data**: If plot series is na, fill automatically becomes invisible
4. **No conditional attribute**: `fill()` has no `display` parameter like `plot()`

### Best Practice Hierarchy

**Option A: Dual Control (RECOMMENDED)**
```pine
series_value = condition ? value : na  // Controls calculation
display      = condition ? display.all : display.none  // Controls visibility
fill_color   = condition ? actualColor : na  // Controls fill
```

**Option B: Visual Only (For Alert Preservation)**
```pine
series_value = value  // Always calculates
display      = condition ? display.all : display.none  // Controls visibility
fill_color   = condition ? actualColor : na  // Controls fill
```

**Option C: Minimal (Current Issue)**
```pine
series_value = value  // Always calculates
// No visibility control - WRONG!
```

---

## Testing Checklist

After implementing fixes, verify:

- [ ] Value Area checkbox hides/shows both lines AND shaded fill
- [ ] POC checkbox hides/shows POC line
- [ ] VWAP checkbox hides/shows VWAP line
- [ ] StdDev checkbox hides/shows both bands AND shaded fill
- [ ] Alerts continue to fire when plots are hidden (if using Option B)
- [ ] No console errors when toggling checkboxes rapidly
- [ ] Settings persist after refresh
- [ ] Default values match user expectations (all true recommended)

---

## Design Rationale

### Why Fills Need Conditional Logic

**Visual Consistency**: Users expect checkboxes to control ALL related visual elements:
- Line visibility → Controlled by checkbox
- Fill visibility → MUST also be controlled by same checkbox

**Performance**: Rendering fills on large datasets is computationally expensive. Users should be able to disable fills to improve performance.

**Customization**: Advanced users may want:
- Lines only (no fills)
- Fills only (no lines) - less common but valid
- Both visible
- Both hidden

### Why Current Code Lacks Controls

**Hypothesis 1: MVP Rush**
Version 0.1.0 (line 122-123: "Initial release") suggests this was an MVP. Plot visibility controls were likely deprioritized for core functionality.

**Hypothesis 2: Copy-Paste from Example**
The plot code structure matches Pine Script documentation examples, which often omit conditional logic for simplicity.

**Hypothesis 3: Incomplete Refactor**
Rectangle controls were added later (lines 289-333) with proper patterns, but the refactor didn't backport to existing plots.

---

## Conclusion

### Key Findings

1. **No visibility controls exist** for statistical lines (VA, POC, VWAP, StdDev)
2. **Fill() calls are unconditional** - always render when data exists
3. **Rectangle frame shows correct pattern** - proves developer knows the pattern
4. **Code suggests incomplete implementation** - likely MVP or refactor issue

### Recommended Action

**Priority: HIGH**

Implement Option A (Dual Control) because:
- Cleanest user experience
- Most predictable behavior
- Matches rectangle frame implementation
- Reduces unnecessary computation when hidden

### File Locations

**Inputs**: Insert after line 277 (in "Display" group)
**Plots**: Replace lines 687-739
**Testing**: Use checkbox toggle + visual inspection

### Estimated Impact

- **User Satisfaction**: HIGH (requested feature)
- **Code Complexity**: LOW (simple boolean conditions)
- **Performance Impact**: POSITIVE (can disable expensive fills)
- **Breaking Changes**: NONE (defaults to true = current behavior)

---

## Appendix: Pine Script Fill() Documentation Reference

From Pine Script v6 documentation:

> `fill(plot1, plot2, color, title, editable, show_last, fillgaps, display)`
>
> Fills background between two plots with a color. If `color` is `na`, the fill is not displayed.

**Key Insight**: The ONLY way to conditionally control fill visibility is through the `color` parameter. There is no `display` parameter for fills (unlike plots).

**Therefore**: The correct pattern is:
```pine
color = condition ? actualColor : na
```

NOT:
```pine
color = actualColor
display = condition ? display.all : display.none  // ❌ Does not exist for fill()
```
