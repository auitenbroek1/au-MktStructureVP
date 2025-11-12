# Research: Rectangle-Profile Synchronization

## Executive Summary

**Finding**: The original implementation **does NOT draw rectangles**. Rectangle functionality was added in the current v1.0.2 implementation as a new feature.

**Key Insight**: Since rectangles are new, there is no "proven approach" from the original to follow. However, the current implementation has architectural patterns that can be analyzed for synchronization.

---

## 1. Does the Original Draw Rectangles?

**Answer: NO**

### Evidence from Original Code:
- **Line 454**: Only `var array<polyline> allPolylines` exists
- **No box arrays**: No `var array<box>` declarations
- **No box.new() calls**: Search reveals zero box creation code
- **No rectangle inputs**: No user input group for rectangle configuration

### Visualization in Original:
The original only renders:
1. **Polylines** (lines 454-571): Volume profile polygons
2. **Plot lines** (lines 573-625): Developing metrics (POC, VA, VWAP, StdDev)

---

## 2. Rectangle Implementation in Current Version

### Architecture Overview (v1.0.2)

The current implementation adds rectangles through:

```pine
// Lines 145-151: Constants
const int MAX_BOXES = 500
const int MAX_HISTORICAL_PROFILES = 50
const int MIN_PROFILE_BARS = 5

// Lines 455-458: Rectangle storage
var array<box> allBoxes = array.new<box>()              // Historical profile boxes
var array<box> currentProfileBoxes = array.new<box>()  // Current profile boxes
var int lastCurrentProfileBar = na                      // Track rendering state
```

---

## 3. Synchronization Mechanism

### 3.1 Coordinate Coupling

**Primary Synchronization Point**: `startBar` and `endBar` variables

Both polylines and rectangles use the same time coordinates:

```pine
// Lines 518-520: Profile boundary calculation (SHARED by both)
if prfReset
    startBar := resetBar - delay    // Profile start time
    endBar   := bar_index - 1      // Profile end time

// Lines 647-648: Final boundary setting
endBar   := bar_index
startBar := resetBar - delay

// Line 871-874: Rectangle uses SAME coordinates
box newBox = box.new(
  left   = startBar,     // ← Same as polyline xBase calculation (line 732)
  right  = endBar,       // ← Same as polyline boundary
  top    = flashProf.rangeUp,
  bottom = flashProf.rangeLo)
```

**Synchronization Pattern**:
1. Profile resets trigger new `startBar`/`endBar` calculation (lines 528-535)
2. Polylines render using these boundaries (lines 730-731)
3. Rectangles render using **identical** boundaries (lines 871-874)
4. Both execute in same `if barstate.islast` block (line 629)

### 3.2 Price Range Coupling

**Profile boundaries drive rectangle Y-axis**:

```pine
// Lines 598-600: Profile range initialization
msProf.setRanges(
  rangeUp = nz(rangeUp[delay], ...),
  rangeLo = nz(rangeLo[delay], ...))

// Lines 873-875: Rectangle uses profile's calculated range
top    = flashProf.rangeUp,    // ← From profile range calculation
bottom = flashProf.rangeLo     // ← Matches profile's price bounds
```

**Key Mechanism**:
- Profile's `rangeUp`/`rangeLo` automatically expand to fit volume data
- Rectangle inherits these dynamically calculated boundaries
- No separate range calculation needed

### 3.3 Lifecycle Synchronization

**Both follow identical lifecycle**:

```pine
// PROFILE RESET (Lines 528-600)
if prfReset
    // 1. Capture previous profile boundaries
    startBar := resetBar - delay
    endBar   := bar_index - 1

    // 2. Reset current profile
    msProf.clear()
    msProf.setRanges(...)

// RENDERING (Lines 655-898)
if endBar >= startBar           // ← Same condition for both
    // Polyline rendering (655-831)
    // Rectangle rendering (851-898)
```

**Synchronization Timeline**:
```
Bar N-1: Profile accumulating data
Bar N:   prfReset triggers
         ├─ Capture startBar/endBar for previous profile
         ├─ Clear msProf for new profile
         └─ Initialize new ranges

Bar N (barstate.islast):
         ├─ Build flashProf with complete data
         ├─ Render polyline using flashProf + startBar/endBar
         └─ Render rectangle using SAME coordinates + flashProf.range
```

---

## 4. Triggers for Rectangle Updates

### 4.1 Profile Reset Trigger

**When**: `prfReset = true` (lines 349, 363, 387)

**Events**:
1. **Swing Mode** (lines 478-490):
   - Pivot high/low confirmed (after pivRi bars)
   - Impulse baseline changes

2. **Structure Mode** (lines 491-502):
   - Market structure break confirmed
   - Price trend state changes

3. **Delta Mode** (lines 503-514):
   - Cumulative delta structure break
   - CVD trend state changes

### 4.2 Rectangle Creation Flow

```pine
if prfReset
    // STEP 1: Capture previous profile boundaries (528-535)
    startBar := resetBar - delay
    endBar   := bar_index - 1

    // STEP 2: Clear profile for new period (597-600)
    msProf.clear()
    msProf.setRanges(...)

    // STEP 3: Render completed profile (655-898)
    if barstate.islast and endBar >= startBar
        // Build flashProf
        // Render polyline
        // Render rectangle ← SAME boundaries as polyline
```

---

## 5. Rectangle Cleanup and Recreation

### 5.1 FIFO Deletion Strategy

**Historical Rectangles** (lines 858-859):
```pine
if array.size(allBoxes) >= MAX_BOXES - 1
    box.delete(array.shift(allBoxes))  // Delete oldest box
```

**Pattern**: When approaching 500-box limit, oldest boxes automatically deleted

### 5.2 Current Profile Rectangle Management

**Problem Solved**: Prevent overlapping rectangles on real-time bar updates

**Solution** (lines 991-997):
```pine
bool isNewBar = bar_index != lastCurrentProfileBar
if isNewBar and array.size(currentProfileBoxes) > 0
    // Delete all current profile boxes from previous bar
    for i = 0 to array.size(currentProfileBoxes) - 1
        box.delete(array.get(currentProfileBoxes, i))
    array.clear(currentProfileBoxes)
```

**Lifecycle**:
1. Bar N: Render current profile rectangles
2. Bar N+1: Delete Bar N's rectangles
3. Bar N+1: Render new rectangles with updated profile data

### 5.3 Historical Profile Capture

**Architecture**: Flattened arrays for efficient storage (lines 461-468)

```pine
// Profile boundaries
var array<int> profileStartBars = array.new<int>()
var array<int> profileEndBars = array.new<int>()

// Peak data (for peak rectangles)
var array<int> profilePeakCounts = array.new<int>()
var array<float> allPeakStartPrices = array.new<float>()
var array<float> allPeakEndPrices = array.new<float>()
```

**Capture Trigger** (lines 541-584):
```pine
if prfReset and not isFirstReset and not na(resetBar)
    // Store completed profile data
    array.push(profileStartBars, startBar)
    array.push(profileEndBars, endBar)
    // Store peak data...

    // FIFO cleanup if exceeding MAX_HISTORICAL_PROFILES
    if array.size(profileStartBars) > 50
        // Remove oldest profile data
```

---

## 6. Key Differences: Original vs Current

| Aspect | Original (v0.1.0) | Current (v1.0.2) |
|--------|-------------------|------------------|
| **Rectangles** | None | Full frame + peak rectangles |
| **Box Arrays** | 0 | 2 arrays (historical + current) |
| **Historical Storage** | None | Flattened arrays for 50 profiles |
| **Cleanup Logic** | N/A | FIFO deletion + per-bar cleanup |
| **User Inputs** | 6 groups | 7 groups (added "rectangle frame") |
| **Constants** | 2 | 7 (added box/profile limits) |

---

## 7. Best Practices for Rectangle-Profile Coupling

### 7.1 Coordinate Sharing
✅ **Do**: Use same `startBar`/`endBar` for both polylines and rectangles
```pine
// Single source of truth for time boundaries
if prfReset
    startBar := resetBar - delay
    endBar   := bar_index - 1

// Both use same coordinates
box.new(left = startBar, right = endBar, ...)
```

❌ **Don't**: Calculate separate coordinates for rectangles

### 7.2 Price Range Synchronization
✅ **Do**: Use profile's dynamic range boundaries
```pine
top    = flashProf.rangeUp,    // Profile calculates optimal range
bottom = flashProf.rangeLo
```

❌ **Don't**: Calculate custom price ranges for rectangles

### 7.3 Lifecycle Coupling
✅ **Do**: Execute rectangle rendering in same `if barstate.islast` block
```pine
if endBar >= startBar  // Single condition guards both
    // Render polyline
    // Render rectangle
```

❌ **Don't**: Use separate rendering logic or conditions

### 7.4 State Management
✅ **Do**: Separate historical and current profile rectangles
```pine
var array<box> allBoxes = array.new<box>()              // Historical
var array<box> currentProfileBoxes = array.new<box>()  // Current (cleaned up)
```

✅ **Do**: Delete current profile rectangles on new bars
```pine
bool isNewBar = bar_index != lastCurrentProfileBar
if isNewBar
    // Delete previous bar's rectangles
```

### 7.5 Safety Checks
✅ **Do**: Implement minimum bar count filter (Issue #6 fix)
```pine
const int MIN_PROFILE_BARS = 5
int profileBarCount = histEndBar - histStartBar + 1
if profileBarCount < MIN_PROFILE_BARS
    continue  // Skip rendering
```

✅ **Do**: Implement buffer limit protection (Issue #2 fix)
```pine
const int SAFETY_MARGIN = 100
int maxSafeOffset = MAX_BARS_BACK_LIMIT - SAFETY_MARGIN
int barsBack = bar_index - histStartBar
if barsBack > maxSafeOffset
    continue  // Skip historical profile
```

---

## 8. Synchronization Checklist

When adding rectangles to profile indicators:

- [ ] Use profile's `startBar`/`endBar` for rectangle X coordinates
- [ ] Use profile's `rangeUp`/`rangeLo` for rectangle Y coordinates
- [ ] Execute rectangle rendering in same `if barstate.islast` block as polylines
- [ ] Use same `if endBar >= startBar` condition guard
- [ ] Implement FIFO deletion for historical rectangles
- [ ] Implement per-bar cleanup for current profile rectangles
- [ ] Add minimum bar count filter (≥5 bars)
- [ ] Add buffer limit protection (max_bars_back - safety_margin)
- [ ] Store historical profiles in flattened arrays for efficiency
- [ ] Track rendering state with `lastCurrentProfileBar`

---

## 9. Conclusions

### Key Findings:

1. **No Original Implementation**: Rectangles are a new feature, not present in original code
2. **Coordinate Sharing**: Rectangles synchronized via shared `startBar`/`endBar` variables
3. **Price Coupling**: Rectangle Y-axis uses profile's dynamic `rangeUp`/`rangeLo`
4. **Lifecycle Alignment**: Both rendered in same execution block with same conditions
5. **Cleanup Strategy**: Two-tier approach (historical FIFO + current per-bar deletion)

### Architecture Strengths:

✅ **Single Source of Truth**: Profile boundaries drive both polylines and rectangles
✅ **Automatic Synchronization**: No manual coordinate tracking needed
✅ **Efficient Storage**: Flattened arrays for historical data
✅ **Safety Mechanisms**: Buffer limits and minimum bar filters

### Potential Issues:

⚠️ **No Original Pattern**: Implementation is custom, not battle-tested
⚠️ **Complexity**: Current profile cleanup adds state management
⚠️ **Memory**: Historical storage limited to 50 profiles

---

## 10. Recommended Approach

For maintaining rectangle-profile synchronization:

1. **Keep coordinate sharing**: Continue using profile's `startBar`/`endBar`
2. **Maintain single execution block**: Render both in same `if barstate.islast`
3. **Preserve cleanup logic**: Keep FIFO deletion and per-bar cleanup
4. **Monitor safety checks**: Ensure MIN_PROFILE_BARS and SAFETY_MARGIN remain effective
5. **Document changes**: Any modifications to profile boundaries must update rectangles

### Future Enhancements:

- Consider storing rectangle metadata alongside profile data
- Add rectangle visibility toggle for performance on crowded charts
- Implement rectangle caching to reduce per-bar recalculation
- Add rectangle color coding based on profile characteristics

---

**Research Date**: 2025-11-11
**Original File**: `/Users/aaronuitenbroek/hive-projects/i10iQ-projects/au-MktStructureVP/src/au-mktStructureVP-original.txt`
**Current File**: `/Users/aaronuitenbroek/hive-projects/i10iQ-projects/au-MktStructureVP/src/au-mktStructureVP-FULL.pine`
**Version Comparison**: v0.1.0 (original) vs v1.0.2 (current)
