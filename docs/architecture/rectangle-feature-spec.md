# Rectangle Drawing Feature - Technical Architecture Specification

**Version:** 1.0
**Date:** 2025-11-10
**Status:** Draft
**Author:** System Architecture Designer

---

## 1. Executive Summary

This specification defines the architecture for adding rectangle drawing functionality to the Market Structure & Volume Profile indicator. Rectangles will visually demarcate volume profile calculation periods, providing traders with clear boundaries for price action analysis.

### 1.1 Key Objectives

- Visualize volume profile calculation periods with rectangles
- Link rectangles to time and price boundaries derived from volume profile data
- Maintain optimal performance within Pine Script's constraints
- Provide flexible user controls for styling and behavior

---

## 2. Rectangle Definition

### 2.1 Boundary Specifications

#### Time Boundaries
```pine
// Start time: lookback period start
rectangle_x1 = bar_index - vpLookback

// End time: current bar
rectangle_x2 = bar_index
```

#### Price Boundaries
```pine
// Top boundary: highest price in lookback period
rectangle_y1 = highestPrice  // Already calculated: ta.highest(high, vpLookback)

// Bottom boundary: lowest price in lookback period
rectangle_y2 = lowestPrice   // Already calculated: ta.lowest(low, vpLookback)
```

### 2.2 Rectangle Properties

| Property | Value | Rationale |
|----------|-------|-----------|
| **Border Color** | User-configurable (default: `color.new(color.blue, 50)`) | Visual consistency with BOS color |
| **Border Width** | 1-2 pixels | Balance visibility with chart clarity |
| **Border Style** | Solid or dashed (user choice) | Differentiate from other indicators |
| **Background Fill** | Optional, semi-transparent | Highlight zones without obscuring price action |
| **Fill Transparency** | 85-95% | Maintain readability of candlesticks |

---

## 3. Implementation Architecture

### 3.1 Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Main Indicator Logic                      │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   Market        │  │   Volume        │  │  Rectangle  │ │
│  │   Structure     │  │   Profile       │  │  Manager    │ │
│  │   Detection     │  │   Calculation   │  │  (NEW)      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
│           │                    │                    │        │
│           └────────────────────┴────────────────────┘        │
│                              │                               │
│                    ┌─────────▼─────────┐                    │
│                    │   Plotting Layer   │                    │
│                    └───────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Integration Points

#### 3.2.1 Input Configuration Section (Lines 4-28)

Add new user inputs after existing Volume Profile settings:

```pine
// Rectangle Settings (NEW)
showRectangles = input.bool(true, "Show VP Rectangles", group="Volume Profile")
rectangleBorderColor = input.color(color.new(color.blue, 50), "Rectangle Border Color", group="Volume Profile")
rectangleBorderWidth = input.int(1, "Rectangle Border Width", minval=1, maxval=3, group="Volume Profile")
rectangleBorderStyle = input.string("Solid", "Border Style", options=["Solid", "Dashed", "Dotted"], group="Volume Profile")
rectangleFillEnabled = input.bool(false, "Fill Rectangle", group="Volume Profile")
rectangleFillColor = input.color(color.new(color.blue, 90), "Rectangle Fill Color", group="Volume Profile")
rectangleMaxDisplay = input.int(50, "Max Rectangles to Display", minval=10, maxval=100, group="Volume Profile")
```

#### 3.2.2 Rectangle Management Module (NEW - Insert after line 148)

```pine
// ==================== RECTANGLE MANAGEMENT ====================

// Rectangle array for lifecycle management
var box[] rectangleArray = array.new_box(0)

// Create rectangle on specific triggers
if showRectangles and showVP and barstate.islast
    // Convert border style string to line.style constant
    borderStyleConst = rectangleBorderStyle == "Solid" ? line.style_solid :
                       rectangleBorderStyle == "Dashed" ? line.style_dashed :
                       line.style_dotted

    // Create new rectangle
    newBox = box.new(
        left = bar_index - vpLookback,
        top = highestPrice,
        right = bar_index,
        bottom = lowestPrice,
        border_color = rectangleBorderColor,
        border_width = rectangleBorderWidth,
        border_style = borderStyleConst,
        bgcolor = rectangleFillEnabled ? rectangleFillColor : na,
        extend = extend.none
    )

    // Add to array for lifecycle management
    array.push(rectangleArray, newBox)

    // Maintain max rectangle limit (FIFO deletion)
    if array.size(rectangleArray) > rectangleMaxDisplay
        oldestBox = array.shift(rectangleArray)
        box.delete(oldestBox)
```

#### 3.2.3 Plotting Section Integration (After line 175)

Rectangle creation is handled in the Rectangle Management module. No changes needed in plotting section, but add comment for clarity:

```pine
// Note: Volume Profile rectangles are managed in RECTANGLE MANAGEMENT section
```

---

## 4. Lifecycle Management

### 4.1 Creation Strategy

**Trigger Conditions:**
1. **Primary Trigger:** `barstate.islast` - Execute only on most recent bar
2. **Volume Profile Active:** `showVP == true`
3. **Rectangles Enabled:** `showRectangles == true`

**Timing:**
- Rectangles created AFTER volume profile calculations complete
- One rectangle per volume profile period
- Real-time updates on new bar formation

### 4.2 Update Strategy

**Static Rectangles (Recommended):**
- Once created, rectangles remain fixed
- Represent historical volume profile periods
- No continuous updates (performance optimization)

**Dynamic Option (Future Enhancement):**
- Update most recent rectangle's right boundary as `bar_index` advances
- Only update if `barstate.isrealtime`

### 4.3 Deletion Strategy

**FIFO (First-In-First-Out) Approach:**

```pine
if array.size(rectangleArray) > rectangleMaxDisplay
    oldestBox = array.shift(rectangleArray)
    box.delete(oldestBox)
```

**Memory Management:**
- Maintain array of active box objects
- Delete oldest when exceeding `rectangleMaxDisplay` limit
- Pine Script automatic garbage collection handles deleted objects

---

## 5. Performance Considerations

### 5.1 Pine Script Constraints

| Resource | Limit | Strategy |
|----------|-------|----------|
| **Maximum Boxes** | ~500 | User-configurable max (default: 50) |
| **Drawing Execution** | Once per bar | Use `barstate.islast` guard |
| **Array Operations** | O(n) complexity | Keep array size bounded |
| **Historical Rendering** | Expensive | Create rectangles only on recent bars |

### 5.2 Optimization Techniques

#### 5.2.1 Conditional Execution
```pine
// Only execute when necessary
if showRectangles and showVP and barstate.islast
    // Rectangle creation logic
```

#### 5.2.2 Bounded Array Size
```pine
// Enforce maximum rectangles
rectangleMaxDisplay = input.int(50, "Max Rectangles", minval=10, maxval=100)
```

#### 5.2.3 Historical vs Real-Time Strategy

**Historical Bars (bar_index < chart.right_visible_bar_time):**
- Skip rectangle creation for bars outside visible range
- Reduces computational load on chart pan/zoom

**Real-Time Bars:**
- Create rectangles normally
- Maintain visual continuity for active trading

```pine
// Example optimization (optional future enhancement)
isVisibleBar = bar_index >= chart.right_visible_bar_time - vpLookback * 2
if showRectangles and showVP and barstate.islast and isVisibleBar
    // Create rectangle
```

---

## 6. User Control Interface

### 6.1 Input Parameters

```pine
// Core Controls
showRectangles       : Toggle rectangle display (bool)
rectangleMaxDisplay  : Maximum concurrent rectangles (int, 10-100)

// Visual Styling
rectangleBorderColor : Border color (color)
rectangleBorderWidth : Border thickness (int, 1-3)
rectangleBorderStyle : Line style (string: Solid/Dashed/Dotted)
rectangleFillEnabled : Enable background fill (bool)
rectangleFillColor   : Fill color with transparency (color)
```

### 6.2 Parameter Grouping

All rectangle parameters grouped under `"Volume Profile"` to maintain logical organization with related settings.

---

## 7. Integration with Existing Features

### 7.1 Volume Profile Linkage

Rectangles directly utilize volume profile calculations:

```pine
// Direct dependencies on existing variables
rectangle_top    = highestPrice   // Line 78
rectangle_bottom = lowestPrice    // Line 79
rectangle_left   = bar_index - vpLookback  // Line 14 input
rectangle_right  = bar_index
```

**No modifications required to volume profile logic.**

### 7.2 Market Structure Independence

Rectangles are visually independent of market structure detection:
- No logical dependencies on BOS/ChoCh events
- Rectangles represent time-price zones, not structure changes
- Future enhancement: Color rectangles based on trend direction

### 7.3 Plotting Order

Recommended z-index (visual stacking):
1. **Bottom Layer:** Rectangles (drawn first)
2. **Middle Layer:** Volume profile lines (POC, VAH, VAL)
3. **Top Layer:** Market structure labels (BOS, ChoCh)

---

## 8. Code Structure Changes

### 8.1 New Code Sections

```
Line 28: ADD - Rectangle input parameters (7 new inputs)
Line 148: ADD - RECTANGLE MANAGEMENT section (~30 lines)
```

### 8.2 Modified Sections

```
Line 177-180: MODIFY - Add cleanup for rectangle array on bar reset
```

### 8.3 File Organization

```
/src/au-mktStructureVP.pine (modified)
├── INPUTS (existing + 7 new)
├── MARKET STRUCTURE DETECTION (unchanged)
├── VOLUME PROFILE CALCULATION (unchanged)
├── RECTANGLE MANAGEMENT (NEW)
└── PLOTTING (unchanged, add reference comment)
```

---

## 9. Testing Strategy

### 9.1 Unit Testing Scenarios

| Test Case | Expected Behavior |
|-----------|-------------------|
| `showRectangles = false` | No rectangles rendered |
| `vpLookback = 100` | Rectangle width = 100 bars |
| `rectangleMaxDisplay = 50` | Array never exceeds 50 elements |
| Rapid bar updates | No lag, smooth rendering |
| Chart pan/zoom | Rectangles persist correctly |

### 9.2 Integration Testing

1. **With Volume Profile:** Rectangles align with POC/VAH/VAL lines
2. **With Market Structure:** Labels display above rectangles
3. **With Multiple Timeframes:** Rectangles scale appropriately
4. **With Historical Data:** Past rectangles remain stable

### 9.3 Performance Testing

```
Test Metrics:
- Script execution time: < 100ms per bar
- Memory usage: Monitor array.size(rectangleArray)
- Visual lag: No stuttering on chart interaction
- Max rectangles stress test: 100 rectangles simultaneous
```

---

## 10. Future Enhancements

### 10.1 Phase 2 Features

1. **Conditional Rectangle Coloring**
   ```pine
   rectangleColor = trend == 1 ? color.green : color.red
   ```

2. **Rectangle Labels**
   - Display volume profile metrics inside rectangle
   - Show period duration

3. **Smart Rectangle Triggers**
   - Create rectangle only on BOS/ChoCh events
   - Variable lookback based on volatility

4. **Rectangle Interaction**
   - Clickable labels with detailed statistics
   - Dynamic tooltip on hover (if Pine Script supports)

### 10.2 Advanced Optimizations

1. **Viewport Culling**
   - Only render rectangles within visible chart area
   - Requires Pine Script viewport detection capabilities

2. **Adaptive Max Display**
   - Auto-adjust `rectangleMaxDisplay` based on timeframe
   - Higher timeframes = more rectangles

---

## 11. Risk Assessment

### 11.1 Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Pine Script box limit exceeded | Medium | High | Enforce `rectangleMaxDisplay` |
| Performance degradation | Low | Medium | Conditional execution, bounded arrays |
| Visual clutter | Medium | Low | Default transparency, user controls |
| Compatibility with older Pine versions | Low | High | Test on v5 thoroughly |

### 11.2 User Experience Risks

| Risk | Mitigation |
|------|------------|
| Chart becomes too busy | Default `showRectangles = false` |
| Rectangles obscure price action | High transparency (90%) for fill |
| Confusion with existing features | Clear naming, grouped inputs |

---

## 12. Implementation Checklist

- [ ] Add rectangle input parameters (7 inputs)
- [ ] Implement rectangle creation logic
- [ ] Implement FIFO array management
- [ ] Add border style conversion logic
- [ ] Test with various `vpLookback` values
- [ ] Test maximum rectangle limit enforcement
- [ ] Verify visual stacking order
- [ ] Document new parameters in code comments
- [ ] Update user guide (if exists)
- [ ] Performance testing on historical data
- [ ] Cross-browser testing (TradingView platform)

---

## 13. Code Example - Complete Implementation

```pine
// ==================== RECTANGLE MANAGEMENT ====================

// Rectangle configuration
var box[] rectangleArray = array.new_box(0)

if showRectangles and showVP and barstate.islast
    // Convert string to line style constant
    borderStyleConst = rectangleBorderStyle == "Solid" ? line.style_solid :
                       rectangleBorderStyle == "Dashed" ? line.style_dashed :
                       line.style_dotted

    // Create new volume profile rectangle
    newRectangle = box.new(
        left = bar_index - vpLookback,
        top = highestPrice,
        right = bar_index,
        bottom = lowestPrice,
        border_color = rectangleBorderColor,
        border_width = rectangleBorderWidth,
        border_style = borderStyleConst,
        bgcolor = rectangleFillEnabled ? rectangleFillColor : na,
        extend = extend.none,
        text = "", // Optional: Add label like "VP-" + str.tostring(bar_index)
        text_size = size.small,
        text_color = rectangleBorderColor
    )

    // Add to lifecycle management array
    array.push(rectangleArray, newRectangle)

    // Enforce maximum rectangle limit (FIFO deletion)
    while array.size(rectangleArray) > rectangleMaxDisplay
        oldestRectangle = array.shift(rectangleArray)
        box.delete(oldestRectangle)
```

---

## 14. Architectural Decision Records (ADRs)

### ADR-001: Static vs Dynamic Rectangles

**Decision:** Implement static rectangles (fixed after creation)

**Rationale:**
- Lower computational overhead
- Clearer historical volume profile visualization
- Aligns with typical volume profile usage patterns

**Consequences:**
- Future dynamic option requires separate implementation
- Users cannot see "live" rectangle updates

---

### ADR-002: FIFO Deletion Strategy

**Decision:** Use First-In-First-Out deletion when exceeding limit

**Rationale:**
- Preserves most recent volume profile periods (most relevant for trading)
- Simple O(1) array operations
- Predictable memory usage

**Consequences:**
- Historical rectangles eventually removed
- Users cannot "pin" specific rectangles

---

### ADR-003: Integration with Existing Code

**Decision:** Minimize modifications to existing volume profile logic

**Rationale:**
- Reduce regression risk
- Maintain separation of concerns
- Easier testing and debugging

**Consequences:**
- Rectangle module is isolated (good)
- Future refactoring may consolidate modules

---

## 15. Glossary

| Term | Definition |
|------|------------|
| **POC** | Point of Control - price level with highest volume |
| **VAH/VAL** | Value Area High/Low - boundaries of 70% volume zone |
| **BOS** | Break of Structure - market structure shift continuation |
| **ChoCh** | Change of Character - market structure reversal |
| **FIFO** | First-In-First-Out - queue data structure behavior |
| **bar_index** | Current bar's sequential number in chart |

---

## 16. References

- Pine Script v5 Documentation: https://www.tradingview.com/pine-script-docs/en/v5/
- Box Drawing Functions: https://www.tradingview.com/pine-script-reference/v5/#fun_box{dot}new
- Volume Profile Theory: Market Profile analysis methodology

---

## 17. Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-10 | System Architect | Initial specification |

---

**End of Specification**
