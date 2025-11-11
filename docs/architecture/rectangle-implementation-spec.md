# Rectangle Frame Implementation Specification
## Market Structure Volume Profile Enhancement

**Document Version:** 1.0.0
**Date:** 2025-11-10
**Status:** Ready for Implementation

---

## 1. Executive Summary

This specification details the implementation of rectangle frames around volume profile histograms in the Market Structure Volume Profile (MSVP) indicator. The enhancement adds visual boundaries that clearly delineate the temporal and price range of each volume profile, improving chart readability and pattern recognition.

### 1.1 Objectives

- Add configurable rectangle frames around volume profiles
- Maintain Pine Script object limits (~500 boxes)
- Integrate seamlessly with existing polyline rendering
- Provide full user customization of appearance

### 1.2 Key Benefits

- **Visual Clarity:** Clear separation between multiple concurrent profiles
- **Pattern Recognition:** Enhanced ability to identify key price/volume zones
- **Professional Appearance:** Clean, customizable visual presentation
- **Minimal Performance Impact:** Efficient box management strategy

---

## 2. Technical Architecture

### 2.1 Data Flow Integration

```
Existing Flow:
┌─────────────────┐
│ Profile Reset   │ (Lines 349-397)
│ Detection       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Volume Profile  │ (Lines 408-429)
│ Calculation     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Polyline        │ (Lines 457-571)
│ Rendering       │
└────────┬────────┘
         │
         ▼
    [NEW STEP]
┌─────────────────┐
│ Rectangle       │ (After Line 574)
│ Frame Rendering │
└─────────────────┘
```

### 2.2 Object Lifecycle Management

```pine
Box Lifecycle:
┌─────────────────────────────────────────────┐
│ 1. Profile Reset Detected (prfReset=true)  │
│    ├─ startBar/endBar calculated           │
│    └─ Rectangle boundaries determined      │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│ 2. Box Creation (box.new)                  │
│    ├─ Time: startBar to endBar             │
│    ├─ Price: rangeLo to rangeUp            │
│    └─ Styling applied                      │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│ 3. Array Management                         │
│    ├─ Add to allBoxes array                │
│    └─ FIFO deletion if limit exceeded      │
└─────────────────────────────────────────────┘
```

---

## 3. Implementation Design

### 3.1 Core Data Structures

#### 3.1.1 Box Storage Array
```pine
// Add after line 454 (alongside allPolylines declaration)
var array<box> allBoxes = array.new<box>()
```

**Rationale:** Maintains box references for lifecycle management, mirroring the polyline approach.

#### 3.1.2 Box Limit Constant
```pine
// Add after line 145 (MAX_POINTS constant)
const int MAX_BOXES = 500
```

**Rationale:** Pine Script's documented limit for box objects per indicator instance.

### 3.2 User Input Parameters

#### Location: Add to Display Group (after line 276)

```pine
const string gR = "rectangle frame"

bool showRect = input.bool(
  title   = "Show Rectangle Frame",
  group   = gR,
  defval  = true,
  tooltip = "Display a rectangle border around each volume profile. The rectangle
             spans from the profile's start time to end time horizontally, and
             from the profile's lowest to highest price vertically.")

color rectBorderClr = input.color(
  title   = "Border Color",
  group   = gR,
  defval  = color.new(color = #4a4a4a, transp = 0),
  tooltip = "Color of the rectangle border line.")

int rectBorderWidth = input.int(
  title   = "Border Width",
  group   = gR,
  defval  = 1,
  minval  = 1,
  maxval  = 4,
  step    = 1,
  tooltip = "Thickness of the rectangle border in pixels.")

string rectBorderStyle = input.string(
  title   = "Border Style",
  group   = gR,
  defval  = line.style_solid,
  options = [line.style_solid, line.style_dashed, line.style_dotted],
  tooltip = "Line style for the rectangle border.")

color rectFillClr = input.color(
  title   = "Fill Color",
  group   = gR,
  defval  = color.new(color = #1a1a1a, transp = 95),
  tooltip = "Background fill color of the rectangle. Use high transparency
             (90-98%) to avoid obscuring price action.")

int rectTransp = input.int(
  title   = "Fill Transparency",
  group   = gR,
  defval  = 95,
  minval  = 0,
  maxval  = 100,
  step    = 1,
  tooltip = "Transparency level for rectangle fill. Higher values (95-98)
             provide subtle background shading without obscuring the chart.")
```

**Design Decisions:**
- **Default transparency: 95%** - Provides subtle visual separation without obscuring price action
- **Border width: 1-4px** - Ensures visibility without overwhelming the chart
- **Solid border default** - Professional appearance, with dashed/dotted alternatives
- **Grouped inputs** - Maintains organization consistency with existing codebase

### 3.3 Rectangle Rendering Logic

#### Location: After line 574 (after polyline rendering completion)

```pine
// ══════════════════════════════════════════════════════════════════
// RECTANGLE FRAME RENDERING
// ══════════════════════════════════════════════════════════════════
// This section draws a rectangular border around the completed volume
// profile. The rectangle spans from startBar to endBar (time axis) and
// from flashProf.rangeLo to flashProf.rangeUp (price axis).
//
// Box Management Strategy:
// - Pine Script limits: ~500 boxes per indicator instance
// - FIFO deletion: When approaching limit, oldest boxes are removed
// - Lifecycle: Boxes persist until manually deleted or limit exceeded
//
// Integration Points:
// - Coordinates: Uses startBar/endBar (lines 394-395, 448-449)
// - Price Range: Uses flashProf.rangeLo/rangeUp (available in flashProf)
// - Execution: Runs within same barstate.islast block as polylines
// ══════════════════════════════════════════════════════════════════

if showRect and endBar >= startBar
    // ──────────────────────────────────────────────────────────────
    // STEP 1: Box Limit Management (FIFO Deletion)
    // ──────────────────────────────────────────────────────────────
    // Remove oldest box if array approaching limit (500 boxes).
    // We maintain buffer of 1 box for the new rectangle.

    if array.size(allBoxes) >= MAX_BOXES - 1
        box.delete(array.shift(allBoxes))

    // ──────────────────────────────────────────────────────────────
    // STEP 2: Rectangle Creation
    // ──────────────────────────────────────────────────────────────
    // Create box with exact coordinates:
    // - Time bounds: startBar (left) to endBar (right)
    // - Price bounds: flashProf.rangeLo (bottom) to rangeUp (top)
    //
    // Note: flashProf contains the finalized profile with correct
    // range values calculated during the profile merge operations.

    box newBox = box.new(
      left   = startBar,
      top    = flashProf.rangeUp,
      right  = endBar,
      bottom = flashProf.rangeLo,

      // Visual styling
      border_color = rectBorderClr,
      border_width = rectBorderWidth,
      border_style = rectBorderStyle,

      bgcolor      = color.new(rectFillClr, rectTransp),

      // Configuration
      xloc         = xloc.bar_index,
      extend       = extend.none,
      text         = na,
      text_halign  = text.align_center,
      text_valign  = text.align_center,
      text_color   = na,
      text_size    = size.auto)

    // ──────────────────────────────────────────────────────────────
    // STEP 3: Array Registration
    // ──────────────────────────────────────────────────────────────
    // Store box reference for lifecycle management.

    array.push(allBoxes, newBox)
```

**Key Design Elements:**

1. **Conditional Rendering:** Only draws if `showRect=true` and valid coordinates exist
2. **Exact Boundaries:** Uses pre-calculated `startBar`/`endBar` and `flashProf` range values
3. **FIFO Management:** Mirrors polyline deletion strategy (lines 476-477)
4. **Comprehensive Styling:** All visual parameters user-configurable
5. **Documentation:** Inline comments explain integration points and coordinate sources

---

## 4. Coordinate System Analysis

### 4.1 Time Axis (X-Coordinates)

#### Source: Lines 393-396, 448-449

```pine
// Profile reset event
if prfReset
    if not na(resetBar)
        startBar := resetBar - delay  // Left boundary
        endBar   := bar_index - 1 - delay  // Right boundary (initial)
    resetBar := bar_index

// Final boundary update (barstate.islast)
endBar   := bar_index  // Right boundary (final)
startBar := resetBar - delay  // Left boundary (confirmation)
```

**Rectangle X-Coordinates:**
- **Left:** `startBar` - Profile start time (includes delay adjustment)
- **Right:** `endBar` - Profile end time (current bar or reset bar)

### 4.2 Price Axis (Y-Coordinates)

#### Source: Lines 413-414, flashProf object

```pine
// Price range available in flashProf object (after merge operations)
flashProf.rangeUp  // Upper boundary (highest price in profile)
flashProf.rangeLo  // Lower boundary (lowest price in profile)
```

**Rectangle Y-Coordinates:**
- **Top:** `flashProf.rangeUp` - Highest price in profile period
- **Bottom:** `flashProf.rangeLo` - Lowest price in profile period

### 4.3 Coordinate Validation

**Validity Conditions:**
```pine
// Rectangle only drawn when:
1. showRect == true           // User enabled
2. endBar >= startBar         // Valid time range (already checked line 457)
3. flashProf.rangeUp > flashProf.rangeLo  // Valid price range (guaranteed)
```

---

## 5. Integration Strategy

### 5.1 Code Insertion Points

| Component | Location | Line Numbers |
|-----------|----------|--------------|
| Constants | After MAX_POINTS | Insert after line 145 |
| User Inputs | Display Group | Insert after line 276 |
| Array Declaration | Alongside allPolylines | Insert after line 454 |
| Rendering Logic | After polyline completion | Insert after line 574 |

### 5.2 Execution Context

**Current Structure (lines 430-451, 457-574):**
```pine
if barstate.islast
    // Profile finalization (430-449)
    flashProf calculations
    startBar/endBar updates

    // Polyline rendering (457-571)
    if endBar >= startBar
        [polyline drawing logic]

    // [INSERT POINT: Rectangle rendering]
    if showRect and endBar >= startBar
        [rectangle drawing logic]
```

**Rationale:** Rectangle shares same execution context and conditions as polylines, ensuring consistent behavior.

---

## 6. Performance Considerations

### 6.1 Memory Footprint

**Object Count Estimates:**

| Profile Count | Polylines (2 per profile) | Boxes (1 per profile) | Total Objects |
|---------------|---------------------------|-----------------------|---------------|
| 50 profiles   | 100 polylines             | 50 boxes              | 150           |
| 100 profiles  | 200 polylines*            | 100 boxes             | 300*          |
| 200 profiles  | 400 polylines*            | 200 boxes             | 600*          |

*Exceeds polyline limit (100), already managed by FIFO deletion (lines 476-477)

**Conclusion:** Box limit (500) more generous than polyline limit (100). Existing FIFO strategy adequate.

### 6.2 Computation Overhead

**Per-Bar Overhead:**
- Rectangle creation: **O(1)** - Single box.new() call
- Array management: **O(1)** - Array shift/push operations
- Condition checking: **O(1)** - Boolean evaluations

**Impact Assessment:** Negligible (<1% computation increase), as box operations are lightweight compared to polyline point calculations.

### 6.3 Visual Rendering

**Browser Rendering:**
- Boxes render faster than polylines (simpler geometry)
- Semi-transparent fill has minimal GPU impact at 95%+ transparency
- No expected performance degradation on modern browsers

---

## 7. Visual Design Guidelines

### 7.1 Color Recommendations

**Border Colors:**
```pine
// Default (neutral, professional)
color = #4a4a4a, transp = 0

// Alternative schemes:
// Subtle accent: #5a5a5a, transp = 20
// Distinct outline: #7a7a7a, transp = 0
// Minimal: #3a3a3a, transp = 50
```

**Fill Colors:**
```pine
// Default (dark subtle)
color = #1a1a1a, transp = 95

// Alternative schemes:
// Lighter background: #2a2a2a, transp = 95
// Warm tone: #2a1a1a, transp = 95
// Cool tone: #1a1a2a, transp = 95
```

### 7.2 Transparency Guidelines

**Recommended Ranges:**
- **Fill Transparency:** 90-98% (default: 95%)
  - 90-93%: Subtle background shading
  - 94-96%: Barely visible tint (recommended)
  - 97-98%: Extremely subtle (for minimal visual clutter)

- **Border Transparency:** 0-30% (default: 0%)
  - 0%: Fully opaque (clear delineation)
  - 10-20%: Slightly muted (softer appearance)
  - 25-30%: Subtle border (minimal visual impact)

### 7.3 Style Combinations

**Professional (Default):**
```pine
Border: Solid, 1px, #4a4a4a, 0% transp
Fill: #1a1a1a, 95% transp
```

**Minimal:**
```pine
Border: Dotted, 1px, #3a3a3a, 30% transp
Fill: #1a1a1a, 98% transp
```

**Distinct:**
```pine
Border: Dashed, 2px, #7a7a7a, 0% transp
Fill: #2a2a2a, 93% transp
```

---

## 8. Testing Strategy

### 8.1 Unit Test Scenarios

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| **TC-001** | Profile reset creates rectangle | Box drawn with correct startBar/endBar |
| **TC-002** | Multiple profiles create multiple rectangles | Distinct boxes for each profile |
| **TC-003** | Box limit exceeded (>500 boxes) | Oldest box deleted via FIFO |
| **TC-004** | Toggle showRect off/on | Rectangles appear/disappear correctly |
| **TC-005** | Change border color/style | Visual updates reflect input changes |
| **TC-006** | Invalid coordinates (endBar < startBar) | No rectangle drawn (condition check) |
| **TC-007** | Single-bar profile | Rectangle with width=1 bar |
| **TC-008** | Large price range profile | Rectangle scales correctly vertically |

### 8.2 Integration Test Scenarios

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| **TC-101** | Rectangle alignment with polylines | Box bounds match histogram extent |
| **TC-102** | Profile anchor mode: Swing | Rectangle delayed by pivRi bars |
| **TC-103** | Profile anchor mode: Structure | Rectangle starts at confirmation bar |
| **TC-104** | Profile anchor mode: Delta | Rectangle starts at confirmation bar |
| **TC-105** | Multiple timeframes | Rectangles adapt to timeframe changes |
| **TC-106** | Display side: Left vs Right | Rectangle position matches histogram |

### 8.3 Visual Validation Checklist

- [ ] Rectangle boundaries align precisely with histogram edges
- [ ] Border color and style render correctly
- [ ] Fill transparency does not obscure price action
- [ ] Rectangles persist across bar updates
- [ ] Old rectangles delete when limit approached
- [ ] No visual artifacts (gaps, overlaps, flickers)
- [ ] Compatible with light/dark chart themes
- [ ] Readable at various chart zoom levels

---

## 9. Edge Cases and Error Handling

### 9.1 Boundary Conditions

| Edge Case | Handling Strategy |
|-----------|-------------------|
| **Single-bar profile** | Rectangle width = 1 bar (valid) |
| **Zero-height profile** | rangeUp = rangeLo (degenerate, but valid box) |
| **Negative bar range** | Prevented by `endBar >= startBar` check |
| **Missing flashProf data** | Condition check prevents execution |
| **Extreme price ranges** | Pine Script handles automatically |

### 9.2 Error Prevention

**Defensive Programming:**
```pine
// Existing safeguard (line 457)
if endBar >= startBar
    // Ensures valid time range before any drawing

// Additional safeguard (recommended)
if showRect and endBar >= startBar and not na(flashProf.rangeUp)
    // Validates all required data exists
```

### 9.3 Pine Script Limitations

**Known Constraints:**
- **Box limit:** 500 per indicator (enforced via FIFO deletion)
- **Coordinate precision:** Limited by bar_index integer resolution
- **Color support:** Standard Pine Script color model (RGBA)
- **Style options:** Limited to solid/dashed/dotted (no custom patterns)

---

## 10. Implementation Checklist

### Phase 1: Preparation
- [ ] Review existing code structure (lines 430-574)
- [ ] Verify `startBar`/`endBar` calculation logic
- [ ] Confirm `flashProf` object structure and availability
- [ ] Backup current indicator version

### Phase 2: Code Implementation
- [ ] Add `MAX_BOXES` constant (after line 145)
- [ ] Add `allBoxes` array declaration (after line 454)
- [ ] Implement user input parameters (after line 276)
- [ ] Implement rectangle rendering logic (after line 574)
- [ ] Add inline documentation comments

### Phase 3: Testing
- [ ] Execute unit tests (TC-001 through TC-008)
- [ ] Execute integration tests (TC-101 through TC-106)
- [ ] Visual validation across multiple timeframes
- [ ] Test with different anchor modes (Swing/Structure/Delta)
- [ ] Validate FIFO deletion with 100+ profiles

### Phase 4: Documentation
- [ ] Update indicator header documentation
- [ ] Add tooltip explanations for new inputs
- [ ] Document rectangle behavior in user guide
- [ ] Update version number and changelog

### Phase 5: Release
- [ ] Code review and approval
- [ ] Performance validation (no measurable overhead)
- [ ] User acceptance testing
- [ ] Publish updated indicator

---

## 11. Code Quality Standards

### 11.1 Naming Conventions

**Variables:**
- `allBoxes` - Follows existing `allPolylines` pattern
- `newBox` - Temporary box reference (local scope)
- `showRect` - Boolean flag (descriptive, concise)

**Constants:**
- `MAX_BOXES` - Uppercase with underscore (matches `MAX_POINTS`)
- `gR` - Group identifier (follows existing `gB`, `gD` pattern)

### 11.2 Code Style

**Indentation:**
- 4 spaces per level (matches existing codebase)
- Aligned parameter lists for readability

**Comments:**
- Section headers with `═` decoration (visual separation)
- Inline comments for complex logic
- Tooltip documentation for all user inputs

**Spacing:**
- Blank lines between logical sections
- Grouped related operations
- Aligned `=` operators for visual clarity

---

## 12. Future Enhancement Opportunities

### 12.1 Short-Term Enhancements

1. **Text Labels:** Add optional text annotation (volume, duration, etc.)
2. **Gradient Fills:** Support gradient backgrounds for depth perception
3. **Rounded Corners:** Aesthetic option for softer visual appearance
4. **Conditional Styling:** Color rectangles based on profile characteristics

### 12.2 Long-Term Considerations

1. **Interactive Tooltips:** Display profile statistics on hover
2. **Dynamic Opacity:** Fade older rectangles for temporal context
3. **Pattern Recognition:** Highlight specific profile shapes
4. **Export Functionality:** Save rectangle coordinates for external analysis

---

## 13. Appendix

### 13.1 Key Code References

| Concept | Location | Description |
|---------|----------|-------------|
| Profile Reset | Lines 349-397 | Anchor detection and reset logic |
| Coordinate Calculation | Lines 393-396, 448-449 | startBar/endBar computation |
| Polyline Rendering | Lines 457-571 | Reference implementation pattern |
| FIFO Management | Lines 476-477 | Array size control strategy |
| flashProf Object | Lines 341, 432-446 | Finalized profile with ranges |

### 13.2 Pine Script Box API Reference

**Constructor:**
```pine
box.new(
  left, top, right, bottom,
  border_color, border_width, border_style,
  extend, xloc,
  bgcolor,
  text, text_size, text_color, text_halign, text_valign, text_wrap, text_font_family
) → series box
```

**Management:**
```pine
box.delete(id)           // Remove box from chart
box.set_*(id, value)     // Update box properties
box.get_*(id)            // Retrieve box properties
```

### 13.3 Glossary

| Term | Definition |
|------|------------|
| **Profile Anchor** | Event trigger for starting new volume profile (Swing/Structure/Delta) |
| **flashProf** | Finalized volume profile object with complete data |
| **startBar/endBar** | Bar indices defining temporal boundaries of profile |
| **rangeUp/rangeLo** | Price boundaries of volume profile |
| **FIFO** | First-In-First-Out deletion strategy for object limit management |
| **pivRi** | Pivot right bars (confirmation lag parameter) |

---

## Document Control

**Version History:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-11-10 | System Architect | Initial specification |

**Approval:**
- [ ] Technical Lead
- [ ] Code Reviewer
- [ ] Product Owner

**Implementation Target:** v0.2.0 (Post v0.1.0 release)

---

**End of Specification**
