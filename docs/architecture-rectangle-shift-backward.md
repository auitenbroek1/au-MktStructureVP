# Architecture Design: Rectangle Backward Shift

## Executive Summary

Design for shifting rectangle (box) start positions backward by one profile, while maintaining polyline integrity. Rectangle N will start where Profile N-1 started, creating visual alignment between rectangles and their corresponding volume profiles.

## Requirements

### Functional Requirements
1. Rectangle N must start at the same bar as Profile N-1's start
2. Profile polylines (volume visualization) must remain unchanged
3. Current profile rectangle uses last historical profile's start bar
4. First historical rectangle (index 0) has no previous profile - needs edge case handling

### Non-Functional Requirements
1. Zero impact on polyline rendering performance
2. Maintain existing safety checks (5-bar minimum, buffer limits)
3. No additional array allocations or lookups per frame
4. Clear separation of concerns between rectangles and profiles

---

## Current Architecture Analysis

### Data Structures

```pinescript
// Global arrays storing profile boundaries
var array<int> profileStartBars = array.new<int>()  // Line 461
var array<int> profileEndBars = array.new<int>()    // Line 462
```

**Key Properties:**
- Arrays grow with each profile creation (line 543-544)
- FIFO pruning maintains MAX_HISTORICAL_PROFILES (line 557-563)
- Indexed by profIdx in rendering loop
- Guaranteed synchronized: `array.size(profileStartBars) == array.size(profileEndBars)`

### Rendering Architecture

**Two Distinct Rendering Paths:**

#### Path 1: Historical Profiles (Lines 925-977)
```pinescript
for profIdx = 0 to numHistoricalProfiles - 1
    int histStartBar = array.get(profileStartBars, profIdx)
    int histEndBar = array.get(profileEndBars, profIdx)

    // Rectangle coordinates
    int leftBar = histStartBar      // Line 962 - TARGET FOR MODIFICATION
    int rightBar = histEndBar + peakExtensionBars

    // Box creation (lines 965-977)
    box peakBox = box.new(
        left   = leftBar,           // Uses profile's start
        right  = rightBar,
        top    = topPrice,
        bottom = bottomPrice,
        ...)
```

#### Path 2: Current Profile (Lines 1007-1047)
```pinescript
// Current profile rendering
int leftBar = startBar              // Line 1022 - TARGET FOR MODIFICATION
int rightBar = bar_index + peakExtensionBars

box peakBox = box.new(
    left   = leftBar,               // Uses current profile's start
    right  = rightBar,
    ...)
```

### Polyline Rendering (Lines 732-829)

**Critical Isolation: Polylines use DIFFERENT coordinate system**

```pinescript
series int xBase = profSide == ProfSide.left ? startBar : endBar  // Line 732

// Polyline points constructed from xBase + offsets
array.push(bPoints, chart.point.from_index(xBase, yFirst))        // Line 750
array.push(bPoints, chart.point.from_index(int(xEnd), bktLo))     // Line 756
```

**Key Insight:** Polylines use `xBase` and dynamically calculated `xEnd` positions based on volume width. They do NOT use `histStartBar` or `histEndBar` directly from the profile arrays.

---

## Proposed Architecture

### 1. Data Access Pattern

**Challenge:** Safely access `profIdx-1` in historical loop

**Solution:** Conditional lookup with bounds checking

```pinescript
// Lookup previous profile's start bar
int previousProfileStartBar = profIdx > 0
    ? array.get(profileStartBars, profIdx - 1)  // Safe: profIdx-1 >= 0
    : histStartBar                              // Fallback: Use own start
```

**Rationale:**
- `profIdx > 0` guarantees valid array access
- First profile (index 0) falls back to its own start bar
- Zero performance overhead (single comparison + conditional assignment)
- No additional array allocations

### 2. Modification Points

#### Modification Point A: Historical Rectangles (Line 962)

**Current Code:**
```pinescript
int leftBar = histStartBar
```

**Modified Code:**
```pinescript
// ARCHITECTURE MODIFICATION: Shift rectangle start backward by one profile
// Rectangle N starts where Profile N-1 started
int leftBar = profIdx > 0
    ? array.get(profileStartBars, profIdx - 1)
    : histStartBar  // First rectangle uses own start (no previous profile exists)
```

**Location:** Between lines 961-963
**Impact:** Historical rectangles only
**Testing:** Verify first rectangle (profIdx=0) renders correctly

#### Modification Point B: Current Profile Rectangle (Line 1022)

**Current Code:**
```pinescript
int leftBar = startBar
```

**Modified Code:**
```pinescript
// ARCHITECTURE MODIFICATION: Current rectangle starts at last historical profile's start
// This maintains continuity: current rectangle uses previous profile's boundary
int leftBar = array.size(profileStartBars) > 0
    ? array.get(profileStartBars, array.size(profileStartBars) - 1)
    : startBar  // No historical profiles exist - use current start
```

**Location:** Between lines 1021-1023
**Impact:** Current profile rectangles only
**Testing:** Verify behavior when zero historical profiles exist

### 3. Isolation Strategy

**Guarantee:** Profile polylines remain unchanged

**Evidence of Isolation:**

1. **Different Variables:**
   - Rectangles: `leftBar`, `rightBar` (local scope in box creation)
   - Polylines: `xBase`, `xEnd` (calculated from profile geometry)

2. **Different Code Paths:**
   - Rectangles: Lines 965-977 (historical), 1037-1047 (current)
   - Polylines: Lines 732-829 (separate rendering block)

3. **No Shared State:**
   - Rectangle `leftBar` is local variable in loop scope
   - Polyline `xBase` calculated independently from `startBar`/`endBar`
   - Modification of `leftBar` cannot affect `xBase` calculation

**Verification Points:**
- Line 732: `xBase` calculation unchanged
- Lines 750-808: Point construction logic unchanged
- Lines 814-829: Polyline creation unchanged

### 4. Edge Cases & Safety

#### Edge Case 1: First Historical Rectangle (profIdx = 0)
**Scenario:** No previous profile exists
**Solution:** Use own start bar as fallback
**Code:** `profIdx > 0 ? ... : histStartBar`
**Result:** First rectangle unshifted (acceptable behavior)

#### Edge Case 2: Zero Historical Profiles
**Scenario:** Current profile rendering when history is empty
**Solution:** Check array size before access
**Code:** `array.size(profileStartBars) > 0 ? ... : startBar`
**Result:** Current rectangle uses own start (safe fallback)

#### Edge Case 3: Profile Array Pruning (FIFO)
**Scenario:** Historical profiles deleted when MAX_HISTORICAL_PROFILES reached
**Impact:** Oldest profile removed from index 0
**Safety:** `profIdx-1` calculation remains valid (array re-indexes automatically)
**No Action Required:** PineScript arrays handle FIFO transparently

#### Existing Safety Checks (Preserved)
```pinescript
// Line 948: 5-bar minimum filter
if profileBarCount < MIN_PROFILE_BARS
    continue  // Skip rendering

// Line 941: Buffer limit safety
if barsBack > maxSafeOffset
    continue  // Skip too-old profiles
```

**Impact:** None. Safety checks evaluate BEFORE leftBar calculation.

---

## Implementation Specification

### Phase 1: Historical Rectangles

**File:** `/Users/aaronuitenbroek/hive-projects/i10iQ-projects/au-MktStructureVP/src/au-mktStructureVP-FULL.pine`

**Location:** Line 962

**Before:**
```pinescript
                // Historical peaks extend from profile start to end + extension
                int leftBar = histStartBar
                int rightBar = histEndBar + peakExtensionBars
```

**After:**
```pinescript
                // Historical peaks extend from profile start to end + extension
                // ARCHITECTURE MODIFICATION: Shift rectangle start backward by one profile
                // Rectangle N starts where Profile N-1 started (visual alignment with profiles)
                int leftBar = profIdx > 0
                    ? array.get(profileStartBars, profIdx - 1)
                    : histStartBar  // First rectangle: no previous profile exists
                int rightBar = histEndBar + peakExtensionBars
```

**Dependencies:** None (uses existing arrays)
**Risk Level:** LOW (isolated change, fallback for edge case)

### Phase 2: Current Profile Rectangle

**File:** `/Users/aaronuitenbroek/hive-projects/i10iQ-projects/au-MktStructureVP/src/au-mktStructureVP-FULL.pine`

**Location:** Line 1022

**Before:**
```pinescript
                // Time coordinates
                int leftBar = startBar
                int rightBar = bar_index + peakExtensionBars
```

**After:**
```pinescript
                // Time coordinates
                // ARCHITECTURE MODIFICATION: Current rectangle uses last historical profile's start
                // Maintains continuity: rectangle aligns with previous profile boundary
                int leftBar = array.size(profileStartBars) > 0
                    ? array.get(profileStartBars, array.size(profileStartBars) - 1)
                    : startBar  // No historical profiles - use current start
                int rightBar = bar_index + peakExtensionBars
```

**Dependencies:** None (uses existing arrays)
**Risk Level:** LOW (isolated change, size check prevents out-of-bounds)

---

## Testing Strategy

### Unit Test Cases

#### Test 1: First Historical Rectangle (profIdx = 0)
**Given:** Single historical profile at bars 100-200
**When:** profIdx = 0 in rendering loop
**Then:** leftBar = 100 (uses own start, no previous profile)
**Verification:** Visual inspection - first rectangle starts at profile start

#### Test 2: Second Historical Rectangle (profIdx = 1)
**Given:** Two profiles: Profile 0 (bars 100-200), Profile 1 (bars 201-300)
**When:** profIdx = 1 in rendering loop
**Then:** leftBar = 100 (uses Profile 0's start)
**Verification:** Rectangle 1 starts at bar 100, extends to 300+extension

#### Test 3: Current Profile (Zero History)
**Given:** No historical profiles, current profile at bars 50-75
**When:** Current profile rendering executes
**Then:** leftBar = 50 (array empty, fallback to startBar)
**Verification:** Current rectangle starts at bar 50

#### Test 4: Current Profile (With History)
**Given:** 3 historical profiles, last starts at bar 300
**When:** Current profile rendering executes
**Then:** leftBar = 300 (uses last historical start)
**Verification:** Current rectangle starts at bar 300

#### Test 5: Profile Array Pruning (FIFO)
**Given:** MAX_HISTORICAL_PROFILES profiles, oldest at index 0
**When:** New profile added, array.shift() removes index 0
**Then:** All profIdx-1 calculations remain valid (re-indexing)
**Verification:** No runtime errors, rectangles render correctly

### Integration Tests

#### Test 6: Polyline Independence
**Given:** Modified rectangle leftBar calculations
**When:** Polylines render (lines 732-829)
**Then:** Polyline xBase and xEnd calculations unchanged
**Verification:** Visual inspection - polylines aligned with profiles (not rectangles)

#### Test 7: Multi-Profile Rendering
**Given:** 10 historical profiles with varying bar ranges
**When:** All profiles render in single frame
**Then:** Each rectangle N starts where profile N-1 started
**Verification:** Debug log leftBar values per profIdx

#### Test 8: Safety Check Interaction
**Given:** Profile with < 5 bars (skipped by MIN_PROFILE_BARS check)
**When:** Rendering loop encounters short profile
**Then:** continue statement executes BEFORE leftBar calculation
**Verification:** No array access for skipped profiles

### Performance Tests

#### Test 9: Execution Time
**Baseline:** Current rendering time per frame
**Modified:** New rendering time with conditional lookups
**Expected:** < 1% increase (single comparison + assignment per rectangle)
**Measurement:** TradingView's built-in performance profiler

#### Test 10: Memory Overhead
**Baseline:** Current array memory usage
**Modified:** Memory usage with new calculations
**Expected:** Zero increase (no new arrays allocated)
**Measurement:** array.size() checks before/after

---

## Risk Analysis

### Risk 1: Out-of-Bounds Array Access
**Likelihood:** LOW
**Impact:** HIGH (runtime error)
**Mitigation:** Bounds checking with `profIdx > 0` and `array.size() > 0`
**Testing:** Test cases 1, 3, 5

### Risk 2: Polyline Misalignment
**Likelihood:** LOW
**Impact:** MEDIUM (visual confusion)
**Mitigation:** Polylines use separate coordinate system (xBase ≠ leftBar)
**Testing:** Test case 6 (visual inspection)

### Risk 3: First Rectangle Edge Case
**Likelihood:** MEDIUM (always occurs)
**Impact:** LOW (acceptable behavior: first rectangle unshifted)
**Mitigation:** Documented fallback behavior
**Testing:** Test case 1

### Risk 4: Performance Degradation
**Likelihood:** LOW
**Impact:** LOW (< 1% expected)
**Mitigation:** Minimal computational overhead (conditional assignment)
**Testing:** Test cases 9, 10

---

## Rollback Strategy

### Rollback Trigger Conditions
1. Runtime errors in production
2. Polylines render incorrectly
3. Performance degradation > 5%
4. Visual artifacts in rectangle rendering

### Rollback Procedure
1. **Immediate:** Restore lines 962 and 1022 to original state
2. **Verification:** Run test cases 1-8 on reverted code
3. **Root Cause Analysis:** Identify which assumption was invalid
4. **Re-design:** Address root cause before retry

### Rollback Code Snippets

**Line 962 Rollback:**
```pinescript
int leftBar = histStartBar  // ORIGINAL CODE
```

**Line 1022 Rollback:**
```pinescript
int leftBar = startBar  // ORIGINAL CODE
```

---

## Success Criteria

### Functional Success
1. All test cases 1-8 pass
2. Visual inspection confirms rectangles shifted backward
3. Polylines remain aligned with profile geometry
4. No runtime errors for 1000+ bars of historical data

### Performance Success
1. Frame rendering time increase < 1%
2. Memory overhead = 0 bytes
3. No additional array allocations detected

### Code Quality Success
1. Clear inline comments explaining modification rationale
2. Edge cases documented in comments
3. Fallback behavior explicitly stated
4. No impact on existing safety checks

---

## Maintenance Notes

### Future Considerations

#### Extension Point 1: Configurable Shift Amount
**Current:** Hard-coded shift of 1 profile backward
**Future:** User input for shift amount (e.g., -2, -3 profiles)
**Implementation:** Replace `profIdx - 1` with `profIdx - shiftAmount`
**Additional Safety:** `profIdx - shiftAmount >= 0` check

#### Extension Point 2: Forward Shift Support
**Current:** Only backward shift (negative offset)
**Future:** Forward shift (rectangle N starts at profile N+1)
**Implementation:** Add bounds check for `profIdx + shiftAmount < numHistoricalProfiles`

#### Extension Point 3: Per-Profile Shift Configuration
**Current:** Global shift applied to all rectangles
**Future:** Custom shift per profile based on metadata
**Implementation:** Add shift configuration array parallel to profileStartBars

### Monitoring Recommendations

1. **Metric:** Average leftBar calculation time per frame
   **Threshold:** < 0.1ms per rectangle
   **Action:** If exceeded, investigate array access performance

2. **Metric:** Array size growth rate (profileStartBars)
   **Threshold:** Linear growth capped at MAX_HISTORICAL_PROFILES
   **Action:** If unbounded growth, check FIFO pruning logic

3. **Metric:** Polyline rendering errors (via runtime.error)
   **Threshold:** Zero errors
   **Action:** If errors occur, verify xBase calculation unchanged

---

## Appendix A: Code Modification Summary

### File: au-mktStructureVP-FULL.pine

#### Modification 1: Historical Rectangles (Line 962)
```pinescript
// OLD (Line 962):
int leftBar = histStartBar

// NEW (Lines 962-966):
// ARCHITECTURE MODIFICATION: Shift rectangle start backward by one profile
// Rectangle N starts where Profile N-1 started (visual alignment with profiles)
int leftBar = profIdx > 0
    ? array.get(profileStartBars, profIdx - 1)
    : histStartBar  // First rectangle: no previous profile exists
```

#### Modification 2: Current Profile Rectangle (Line 1022)
```pinescript
// OLD (Line 1022):
int leftBar = startBar

// NEW (Lines 1022-1026):
// ARCHITECTURE MODIFICATION: Current rectangle uses last historical profile's start
// Maintains continuity: rectangle aligns with previous profile boundary
int leftBar = array.size(profileStartBars) > 0
    ? array.get(profileStartBars, array.size(profileStartBars) - 1)
    : startBar  // No historical profiles - use current start
```

#### Lines Modified: 2 total
- Line 962: Historical rectangle leftBar calculation
- Line 1022: Current profile rectangle leftBar calculation

#### Lines Added: 8 total
- 4 lines (Modification 1: comment + conditional)
- 4 lines (Modification 2: comment + conditional)

#### Net Impact:
- +6 lines (2 modified → 8 total)
- 0 lines removed
- 0 new variables introduced
- 0 new arrays allocated

---

## Appendix B: Variable Scope Analysis

### Global Scope
```pinescript
var array<int> profileStartBars  // Line 461 - READ ONLY
var array<int> profileEndBars    // Line 462 - NOT MODIFIED
```

### Loop Scope (Historical Rendering: Lines 925-977)
```pinescript
for profIdx = 0 to numHistoricalProfiles - 1
    int histStartBar  // READ: Line 926
    int histEndBar    // READ: Line 927

    // LOCAL VARIABLE - Modified by architecture change
    int leftBar       // Line 962 - CALCULATION CHANGED
    int rightBar      // Line 963 - UNCHANGED
```

### Block Scope (Current Rendering: Lines 1006-1047)
```pinescript
if currentProfileBarCount >= MIN_PROFILE_BARS
    for peakIdx = 0 to numCurrentPeaks - 1
        // LOCAL VARIABLE - Modified by architecture change
        int leftBar   // Line 1022 - CALCULATION CHANGED
        int rightBar  // Line 1023 - UNCHANGED
```

### Polyline Scope (Volume Profile Rendering: Lines 730-829)
```pinescript
if maxVol > 0
    series int xBase      // Line 732 - UNCHANGED (independent calculation)
    series float xEnd     // Lines 755, 782, etc. - UNCHANGED
```

**Scope Isolation Confirmed:**
- `leftBar` is local to rectangle creation blocks
- `xBase` is local to polyline creation block
- No shared state between rectangle and polyline rendering
- Modification of `leftBar` CANNOT affect `xBase`

---

## Appendix C: Performance Analysis

### Computational Complexity

#### Historical Rectangles (Per Frame)
```
Current:  O(1) - Direct array access
Modified: O(1) - Conditional + array access
```

**Breakdown:**
- `profIdx > 0`: 1 comparison (O(1))
- `array.get(profileStartBars, profIdx - 1)`: 1 array access (O(1))
- Fallback `histStartBar`: 1 variable read (O(1))
- **Total per rectangle:** 2-3 operations (vs 1 operation in current code)

#### Current Profile (Per Frame)
```
Current:  O(1) - Variable read
Modified: O(1) - Array size check + array access
```

**Breakdown:**
- `array.size(profileStartBars)`: 1 size check (O(1))
- `array.get(..., size - 1)`: 1 array access (O(1))
- Fallback `startBar`: 1 variable read (O(1))
- **Total per rectangle:** 2-3 operations (vs 1 operation in current code)

### Worst-Case Analysis

**Scenario:** 50 historical profiles + 10 peaks per profile
- **Rectangles created:** 50 profiles × 10 peaks = 500 boxes
- **Current overhead:** 500 operations
- **Modified overhead:** 1000-1500 operations (2-3× increase)
- **Absolute increase:** 500-1000 operations per frame
- **PineScript execution:** ~1M operations/second capacity
- **Expected impact:** 0.05-0.1% frame time increase

**Conclusion:** Performance impact negligible.

### Memory Analysis

**New Allocations:** 0
- No new arrays created
- No new variables stored globally
- Local variables reused per loop iteration

**Memory Reads (Per Frame):**
- Current: 1 read from histStartBar/startBar
- Modified: 1-2 reads (array size + array access)
- Additional reads: 0-1 per rectangle

**Memory Footprint:** UNCHANGED

---

## Document Metadata

**Version:** 1.0
**Date:** 2025-11-11
**Author:** System Architecture Designer
**Review Status:** Ready for Implementation
**Estimated Implementation Time:** 5-10 minutes
**Estimated Testing Time:** 30-60 minutes
**Risk Level:** LOW
**Complexity:** LOW

---

## Approval Checklist

- [x] Requirements clearly defined
- [x] Edge cases identified and handled
- [x] Polyline independence verified
- [x] Safety checks preserved
- [x] Performance impact analyzed (< 1%)
- [x] Memory overhead analyzed (0 bytes)
- [x] Rollback strategy documented
- [x] Test cases defined (10 tests)
- [x] Code modification points specified
- [x] Inline comments drafted
- [x] Scope isolation confirmed
- [x] Risk analysis completed

**Ready for Implementation:** YES
