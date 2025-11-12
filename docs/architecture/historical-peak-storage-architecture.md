# Historical Profile Peak Data Storage Architecture
## Pine Script v6 Constraint-Compatible Design

**Document Version:** 1.0
**Date:** 2025-11-11
**Author:** System Architecture Designer
**Status:** Recommended Implementation

---

## Executive Summary

This document provides a comprehensive architectural design for storing historical volume profile peak data within Pine Script v6's constraint that **nested arrays are not supported**. After evaluating four design options, we recommend **Option A: Flattened Array with Index Mapping** as the optimal solution.

**Key Constraint:** Pine Script v6 does NOT support `array<array<int>>` or any nested array structures.

---

## Requirements Analysis

### Functional Requirements

1. **Storage Capacity:** Up to 50 historical profiles
2. **Variable Peak Count:** 2-5 peaks typically, maximum ~10 peaks per profile
3. **Peak Data Structure:**
   - `startRow` (int): Starting row index of peak
   - `endRow` (int): Ending row index of peak
4. **Profile Boundaries:**
   - `startBar` (int): Starting bar index
   - `endBar` (int): Ending bar index

### Non-Functional Requirements

1. **Memory Efficiency:** Minimize array size and memory overhead
2. **Access Performance:** Fast read/write operations for rendering
3. **Maintainability:** Clear, understandable code structure
4. **Pine Script Compliance:** No nested arrays, stay within platform limits

### Current Implementation Issues

The existing code (lines 606-610 in `au-mktStructureVP-FULL.pine`) attempts to use nested arrays:

```pinescript
var array<array<int>> historicalPeakStarts = array.new<array<int>>()  // ❌ NOT SUPPORTED
var array<array<int>> historicalPeakEnds = array.new<array<int>>()    // ❌ NOT SUPPORTED
```

This violates Pine Script v6's fundamental constraint.

---

## Design Options Evaluation

### Option A: Flattened Array with Index Mapping ✅ **RECOMMENDED**

#### Architecture Overview

Store all peak data in flat, linear arrays with a separate index mapping system to track profile boundaries.

#### Data Structures

```pinescript
// Profile boundary tracking
var array<int> profileStartBars = array.new<int>()     // Length: N profiles
var array<int> profileEndBars = array.new<int>()       // Length: N profiles

// Peak data tracking (flattened)
var array<int> profilePeakCounts = array.new<int>()    // Length: N profiles
var array<int> allPeakStartRows = array.new<int>()     // Length: Total peaks across all profiles
var array<int> allPeakEndRows = array.new<int>()       // Length: Total peaks across all profiles

// Index mapping for quick access
var array<int> profilePeakStartIndex = array.new<int>() // Length: N profiles
```

#### Data Layout Example

For 3 profiles with varying peak counts:
- Profile 0: 3 peaks
- Profile 1: 2 peaks
- Profile 2: 4 peaks

**Storage Layout:**
```
profileStartBars:        [100, 250, 400]
profileEndBars:          [249, 399, 549]
profilePeakCounts:       [3, 2, 4]
profilePeakStartIndex:   [0, 3, 5]

allPeakStartRows:        [5, 12, 18,  8, 20,  3, 9, 15, 22]
                          └─Profile 0─┘ └Profile 1┘ └──Profile 2──┘

allPeakEndRows:          [8, 15, 21,  11, 23,  6, 12, 18, 25]
                          └─Profile 0─┘ └Profile 1┘ └──Profile 2──┘
```

#### Algorithms

**1. Adding New Profile with Peaks**

```pinescript
// @function Store completed profile with peaks
// @param startBar Profile start bar index
// @param endBar Profile end bar index
// @param peakStarts Array of peak start rows for this profile
// @param peakEnds Array of peak end rows for this profile
storeHistoricalProfile(int startBar, int endBar, array<int> peakStarts, array<int> peakEnds) =>
    // Store profile boundaries
    array.push(profileStartBars, startBar)
    array.push(profileEndBars, endBar)

    // Store peak count
    int peakCount = array.size(peakStarts)
    array.push(profilePeakCounts, peakCount)

    // Store starting index for this profile's peaks
    int totalPeaksSoFar = array.size(allPeakStartRows)
    array.push(profilePeakStartIndex, totalPeaksSoFar)

    // Append all peaks to flattened arrays
    for i = 0 to peakCount - 1
        array.push(allPeakStartRows, array.get(peakStarts, i))
        array.push(allPeakEndRows, array.get(peakEnds, i))

    void
```

**2. Retrieving Profile Peaks**

```pinescript
// @function Get peaks for a specific historical profile
// @param profileIdx Index of profile (0 to N-1)
// @returns [array<int>, array<int>] Tuple of peak start/end arrays
getProfilePeaks(int profileIdx) =>
    // Get peak count for this profile
    int peakCount = array.get(profilePeakCounts, profileIdx)

    // Get starting index in flattened arrays
    int startIdx = array.get(profilePeakStartIndex, profileIdx)

    // Extract peaks for this profile
    array<int> starts = array.new<int>()
    array<int> ends = array.new<int>()

    for i = 0 to peakCount - 1
        int globalIdx = startIdx + i
        array.push(starts, array.get(allPeakStartRows, globalIdx))
        array.push(ends, array.get(allPeakEndRows, globalIdx))

    [starts, ends]
```

**3. FIFO Cleanup (Maintaining 50 Profile Limit)**

```pinescript
// @function Remove oldest profile when limit exceeded
// @returns void
cleanupOldestProfile() =>
    if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
        // Remove profile boundaries
        array.shift(profileStartBars)
        array.shift(profileEndBars)

        // Get peak count of removed profile
        int removedPeakCount = array.shift(profilePeakCounts)

        // Remove starting index entry
        array.shift(profilePeakStartIndex)

        // Remove peaks from flattened arrays
        for i = 0 to removedPeakCount - 1
            array.shift(allPeakStartRows)
            array.shift(allPeakEndRows)

        // Adjust all remaining profile indices
        for i = 0 to array.size(profilePeakStartIndex) - 1
            int currentIdx = array.get(profilePeakStartIndex, i)
            array.set(profilePeakStartIndex, i, currentIdx - removedPeakCount)

    void
```

**4. Rendering Historical Peaks**

```pinescript
// @function Render all historical profile peaks
// @returns void
renderHistoricalPeaks() =>
    int numProfiles = array.size(profileStartBars)

    for profIdx = 0 to numProfiles - 1
        int startBar = array.get(profileStartBars, profIdx)
        int endBar = array.get(profileEndBars, profIdx)

        // Get peaks for this profile
        [peakStarts, peakEnds] = getProfilePeaks(profIdx)

        int numPeaks = array.size(peakStarts)

        for peakIdx = 0 to numPeaks - 1
            int startRow = array.get(peakStarts, peakIdx)
            int endRow = array.get(peakEnds, peakIdx)

            // Render peak rectangle
            // (convert row indices to prices using profile bucket bounds)
            // [Implementation details from existing code]

    void
```

#### Performance Analysis

**Time Complexity:**
- **Add Profile:** O(P) where P = number of peaks in profile (typically 2-5)
- **Get Profile Peaks:** O(P) to extract peaks for one profile
- **FIFO Cleanup:** O(P + N) where N = number of remaining profiles (index adjustment)
- **Render All:** O(N × P_avg) where N = profiles, P_avg = average peaks per profile

**Space Complexity:**
- **Profile Metadata:** 4 arrays × 50 profiles = 200 integers
- **Peak Data:** 2 arrays × (50 profiles × 5 peaks avg) = 500 integers
- **Total:** ~700 integers ≈ 2.8 KB (assuming 4 bytes/int)

#### Advantages ✅

1. **Pine Script Compliant:** No nested arrays required
2. **Memory Efficient:** No wasted space for unused peak slots
3. **Flexible:** Handles variable peak counts naturally
4. **Predictable Performance:** Linear operations with known bounds
5. **Maintainable:** Clear data flow with documented algorithms

#### Disadvantages ⚠️

1. **Index Management:** Requires careful tracking of indices (mitigated by helper functions)
2. **FIFO Complexity:** Cleanup requires index adjustment (O(N) operation)
3. **No Random Access:** Must iterate to find specific profile (acceptable for rendering)

---

### Option B: Fixed-Size Peak Arrays per Profile

#### Architecture Overview

Pre-allocate a fixed number of peak slots for each profile, using sentinel values for unused slots.

#### Data Structures

```pinescript
const int MAX_PEAKS_PER_PROFILE = 10

var array<int> profileStartBars = array.new<int>()  // Length: 50 profiles
var array<int> profileEndBars = array.new<int>()    // Length: 50 profiles

// Fixed-size peak storage (50 profiles × 10 peaks each)
var array<int> allPeakStartRows = array.new<int>()  // Length: 500
var array<int> allPeakEndRows = array.new<int>()    // Length: 500
```

#### Data Layout Example

For 3 profiles (slots 0-9, 10-19, 20-29):
```
Profile 0 (3 peaks):
  Slots 0-9: [5, 12, 18, -1, -1, -1, -1, -1, -1, -1]

Profile 1 (2 peaks):
  Slots 10-19: [8, 20, -1, -1, -1, -1, -1, -1, -1, -1]

Profile 2 (4 peaks):
  Slots 20-29: [3, 9, 15, 22, -1, -1, -1, -1, -1, -1]
```

#### Algorithms

**1. Adding New Profile**

```pinescript
storeHistoricalProfile(int startBar, int endBar, array<int> peakStarts, array<int> peakEnds) =>
    int profileIdx = array.size(profileStartBars)

    array.push(profileStartBars, startBar)
    array.push(profileEndBars, endBar)

    int baseIdx = profileIdx * MAX_PEAKS_PER_PROFILE
    int peakCount = math.min(array.size(peakStarts), MAX_PEAKS_PER_PROFILE)

    // Fill peak slots
    for i = 0 to MAX_PEAKS_PER_PROFILE - 1
        if i < peakCount
            array.push(allPeakStartRows, array.get(peakStarts, i))
            array.push(allPeakEndRows, array.get(peakEnds, i))
        else
            array.push(allPeakStartRows, -1)  // Sentinel value
            array.push(allPeakEndRows, -1)

    void
```

**2. Retrieving Profile Peaks**

```pinescript
getProfilePeaks(int profileIdx) =>
    array<int> starts = array.new<int>()
    array<int> ends = array.new<int>()

    int baseIdx = profileIdx * MAX_PEAKS_PER_PROFILE

    for i = 0 to MAX_PEAKS_PER_PROFILE - 1
        int startRow = array.get(allPeakStartRows, baseIdx + i)

        if startRow != -1  // Skip sentinel values
            array.push(starts, startRow)
            array.push(ends, array.get(allPeakEndRows, baseIdx + i))

    [starts, ends]
```

#### Performance Analysis

**Time Complexity:**
- **Add Profile:** O(MAX_PEAKS) = O(10) - constant time
- **Get Profile Peaks:** O(MAX_PEAKS) = O(10) - constant time
- **FIFO Cleanup:** O(MAX_PEAKS) = O(10) - constant time
- **Render All:** O(N × MAX_PEAKS) = O(500) - predictable

**Space Complexity:**
- **Profile Metadata:** 2 arrays × 50 profiles = 100 integers
- **Peak Data:** 2 arrays × (50 profiles × 10 slots) = 1000 integers
- **Total:** 1100 integers ≈ 4.4 KB

**Memory Efficiency:**
- **Actual Usage:** 50 profiles × 5 peaks avg × 2 values = 500 integers
- **Allocated:** 1000 integers
- **Waste:** 500 integers (50% overhead)

#### Advantages ✅

1. **Simple Implementation:** Direct index calculation
2. **Constant Time Access:** O(1) to locate profile's peak block
3. **Easy FIFO:** No index adjustment needed
4. **Predictable Memory:** Fixed allocation size

#### Disadvantages ⚠️

1. **Memory Waste:** 50% overhead for typical usage (5 peaks vs 10 slots)
2. **Hard Limit:** Cannot handle profiles with >10 peaks (rare but possible)
3. **Sentinel Value Checks:** Must filter -1 values during retrieval
4. **Inflexible:** Changing MAX_PEAKS requires code changes

---

### Option C: Concatenated String Encoding

#### Architecture Overview

Encode peak data as strings, parse when needed for rendering.

#### Data Structures

```pinescript
var array<int> profileStartBars = array.new<int>()
var array<int> profileEndBars = array.new<int>()
var array<string> profilePeakData = array.new<string>()  // Format: "5,8;12,15;18,21"
```

#### Example Encoding

```pinescript
Profile with 3 peaks:
  Peak 1: startRow=5, endRow=8
  Peak 2: startRow=12, endRow=15
  Peak 3: startRow=18, endRow=21

Encoded: "5,8;12,15;18,21"
```

#### Algorithms

**String Encoding:**
```pinescript
encodePeaks(array<int> starts, array<int> ends) =>
    string result = ""
    for i = 0 to array.size(starts) - 1
        if i > 0
            result += ";"
        result += str.tostring(array.get(starts, i))
        result += ","
        result += str.tostring(array.get(ends, i))
    result
```

**String Parsing:**
```pinescript
decodePeaks(string encoded) =>
    array<int> starts = array.new<int>()
    array<int> ends = array.new<int>()

    // Pine Script does not have built-in split() function
    // Must implement manual parsing (complex and error-prone)

    // Pseudo-code (actual implementation is non-trivial):
    // 1. Split by ";" to get peak pairs
    // 2. Split each pair by "," to get start/end
    // 3. Convert strings to integers

    [starts, ends]
```

#### Performance Analysis

**Time Complexity:**
- **Add Profile:** O(P × log(MAX_DIGITS)) for string conversion
- **Get Profile Peaks:** O(P × L) where L = average string length
- **String Parsing:** O(S) where S = total string length (complex)

**Space Complexity:**
- **String Storage:** ~15 bytes per peak (e.g., "123,456;")
- **50 profiles × 5 peaks × 15 bytes** = 3750 bytes
- **More than Option A but less than Option B**

#### Advantages ✅

1. **Flexible:** No hard limit on peak count
2. **Compact for Few Peaks:** Efficient for 2-3 peaks
3. **Human Readable:** Easy debugging

#### Disadvantages ⚠️

1. **Parsing Complexity:** Pine Script lacks built-in string manipulation
2. **Performance:** String operations slower than integer array access
3. **Error Prone:** Manual parsing implementation risks bugs
4. **Type Safety:** No compile-time validation of format
5. **Debugging Difficulty:** Harder to inspect encoded data

---

### Option D: Separate Parallel Arrays for Each Peak Slot

#### Architecture Overview

Create separate arrays for each possible peak slot (e.g., peak1_start, peak1_end, peak2_start, etc.).

#### Data Structures

```pinescript
// Profile boundaries
var array<int> profileStartBars = array.new<int>()
var array<int> profileEndBars = array.new<int>()

// Peak slot 1
var array<int> peak1_startRows = array.new<int>()
var array<int> peak1_endRows = array.new<int>()

// Peak slot 2
var array<int> peak2_startRows = array.new<int>()
var array<int> peak2_endRows = array.new<int>()

// ... continue for all 10 slots
// Peak slot 10
var array<int> peak10_startRows = array.new<int>()
var array<int> peak10_endRows = array.new<int>()

// Total: 22 separate arrays (2 boundaries + 20 peak arrays)
```

#### Algorithms

**Adding Profile:**
```pinescript
storeHistoricalProfile(int startBar, int endBar, array<int> peakStarts, array<int> peakEnds) =>
    array.push(profileStartBars, startBar)
    array.push(profileEndBars, endBar)

    // Must handle each slot explicitly
    if array.size(peakStarts) >= 1
        array.push(peak1_startRows, array.get(peakStarts, 0))
        array.push(peak1_endRows, array.get(peakEnds, 0))
    else
        array.push(peak1_startRows, -1)
        array.push(peak1_endRows, -1)

    if array.size(peakStarts) >= 2
        array.push(peak2_startRows, array.get(peakStarts, 1))
        array.push(peak2_endRows, array.get(peakEnds, 1))
    else
        array.push(peak2_startRows, -1)
        array.push(peak2_endRows, -1)

    // ... repeat for all 10 slots (verbose and error-prone)

    void
```

#### Performance Analysis

**Time Complexity:**
- **Add Profile:** O(MAX_PEAKS) with 10× code duplication
- **Get Profile Peaks:** O(MAX_PEAKS) with complex slot checking

**Space Complexity:**
- Same as Option B: ~4.4 KB
- But with 22 separate array variables

#### Advantages ✅

1. **Simple Data Model:** Each slot is explicit
2. **Constant Time Access:** Direct array index

#### Disadvantages ⚠️

1. **Extreme Code Duplication:** 10× repeated logic for each slot
2. **Unmaintainable:** Changes require updating 10 code blocks
3. **Error Prone:** Easy to miss a slot in updates
4. **Inflexible:** Adding slots requires significant code changes
5. **Memory Waste:** Same 50% overhead as Option B
6. **Variable Explosion:** 22 separate arrays clutters namespace

---

## Recommendation: Option A

### Decision Rationale

**Option A (Flattened Array with Index Mapping)** is the recommended architecture based on the following criteria:

| Criteria | Option A | Option B | Option C | Option D |
|----------|----------|----------|----------|----------|
| **Pine Script Compliance** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Memory Efficiency** | ✅ Optimal | ⚠️ 50% waste | ✅ Good | ⚠️ 50% waste |
| **Implementation Complexity** | ✅ Moderate | ✅ Simple | ❌ High | ❌ Very High |
| **Maintainability** | ✅ High | ✅ High | ⚠️ Medium | ❌ Low |
| **Performance** | ✅ Fast | ✅ Fast | ❌ Slow | ✅ Fast |
| **Flexibility** | ✅ Dynamic | ❌ Fixed 10 | ✅ Dynamic | ❌ Fixed 10 |
| **Code Clarity** | ✅ Clear | ✅ Clear | ⚠️ Medium | ❌ Cluttered |

### Key Decision Factors

1. **Memory Efficiency:** Option A uses ~700 integers vs. Option B's 1100 integers (36% reduction)
2. **Flexibility:** Handles variable peak counts without waste or artificial limits
3. **Maintainability:** Well-documented helper functions encapsulate complexity
4. **Performance:** All operations are O(P) or better, acceptable for rendering
5. **Pine Script Native:** Uses only standard array operations, no workarounds

### Trade-offs Accepted

1. **Index Management Complexity:** Mitigated by encapsulating logic in helper functions
2. **FIFO Cleanup Cost:** O(N) index adjustment is acceptable (happens once per profile reset)
3. **No Random Access:** Sequential iteration is sufficient for rendering use case

---

## Implementation Plan

### Phase 1: Data Structure Refactoring

**File:** `/src/au-mktStructureVP-FULL.pine`
**Lines to Replace:** 606-610, 514-543, 873-917

**Step 1.1: Replace Nested Arrays**
```pinescript
// REMOVE (lines 606-610):
var array<int> historicalStartBars = array.new<int>()
var array<int> historicalEndBars = array.new<int>()
var array<array<int>> historicalPeakStarts = array.new<array<int>>()  // ❌
var array<array<int>> historicalPeakEnds = array.new<array<int>>()    // ❌

// ADD:
// ══════════════════════════════════════════════════════════════════
// HISTORICAL PROFILE STORAGE - FLATTENED ARRAY ARCHITECTURE
// ══════════════════════════════════════════════════════════════════
// Profile boundary tracking (50 profiles)
var array<int> profileStartBars = array.new<int>()        // Length: N profiles
var array<int> profileEndBars = array.new<int>()          // Length: N profiles

// Peak metadata tracking (50 profiles)
var array<int> profilePeakCounts = array.new<int>()       // Length: N profiles
var array<int> profilePeakStartIndex = array.new<int>()   // Length: N profiles

// Flattened peak data storage (all peaks across all profiles)
var array<int> allPeakStartRows = array.new<int>()        // Length: Total peaks
var array<int> allPeakEndRows = array.new<int>()          // Length: Total peaks

const int MAX_HISTORICAL_PROFILES = 50
```

### Phase 2: Helper Function Implementation

**Step 2.1: Add Storage Function**
```pinescript
// @function Store completed profile with peaks using flattened array architecture
// @param startBar Profile start bar index
// @param endBar Profile end bar index
// @param peakStarts Array of peak start rows for current profile
// @param peakEnds Array of peak end rows for current profile
// @returns void
storeHistoricalProfile(int startBar, int endBar, array<int> peakStarts, array<int> peakEnds) =>
    // Store profile boundaries
    array.push(profileStartBars, startBar)
    array.push(profileEndBars, endBar)

    // Store peak count
    int peakCount = array.size(peakStarts)
    array.push(profilePeakCounts, peakCount)

    // Store starting index for this profile's peaks in flattened arrays
    int totalPeaksSoFar = array.size(allPeakStartRows)
    array.push(profilePeakStartIndex, totalPeaksSoFar)

    // Append all peaks to flattened arrays
    for i = 0 to peakCount - 1
        array.push(allPeakStartRows, array.get(peakStarts, i))
        array.push(allPeakEndRows, array.get(peakEnds, i))

    void
```

**Step 2.2: Add Retrieval Function**
```pinescript
// @function Get peaks for a specific historical profile
// @param profileIdx Index of profile (0 to N-1)
// @returns [array<int>, array<int>] Tuple of (peak starts, peak ends)
getProfilePeaks(int profileIdx) =>
    // Validate profile index
    if profileIdx < 0 or profileIdx >= array.size(profilePeakCounts)
        [array.new<int>(), array.new<int>()]
    else
        // Get peak count for this profile
        int peakCount = array.get(profilePeakCounts, profileIdx)

        // Get starting index in flattened arrays
        int startIdx = array.get(profilePeakStartIndex, profileIdx)

        // Extract peaks for this profile
        array<int> starts = array.new<int>()
        array<int> ends = array.new<int>()

        for i = 0 to peakCount - 1
            int globalIdx = startIdx + i
            array.push(starts, array.get(allPeakStartRows, globalIdx))
            array.push(ends, array.get(allPeakEndRows, globalIdx))

        [starts, ends]
```

**Step 2.3: Add FIFO Cleanup Function**
```pinescript
// @function Remove oldest profile when limit exceeded (FIFO)
// @returns void
cleanupOldestProfile() =>
    if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
        // Remove profile boundaries
        array.shift(profileStartBars)
        array.shift(profileEndBars)

        // Get peak count of removed profile
        int removedPeakCount = array.shift(profilePeakCounts)

        // Remove starting index entry
        array.shift(profilePeakStartIndex)

        // Remove peaks from flattened arrays (from front)
        for i = 0 to removedPeakCount - 1
            array.shift(allPeakStartRows)
            array.shift(allPeakEndRows)

        // Adjust all remaining profile indices
        // (decrement each by removedPeakCount since we removed peaks from front)
        for i = 0 to array.size(profilePeakStartIndex) - 1
            int currentIdx = array.get(profilePeakStartIndex, i)
            array.set(profilePeakStartIndex, i, currentIdx - removedPeakCount)

    void
```

### Phase 3: Integration with Existing Code

**Step 3.1: Replace Profile Capture Logic (lines 514-543)**
```pinescript
if prfReset
    if not na(resetBar)
        startBar := resetBar - delay
        endBar   := bar_index - 1 - delay

        // ══════════════════════════════════════════════════════════════════
        // HISTORICAL PROFILE CAPTURE - USING FLATTENED ARRAY ARCHITECTURE
        // ══════════════════════════════════════════════════════════════════

        if array.size(currentPeakStarts) > 0
            // Store profile with flattened array architecture
            storeHistoricalProfile(startBar, endBar, currentPeakStarts, currentPeakEnds)

            // FIFO cleanup if exceeding limit
            cleanupOldestProfile()

        // Clear working arrays for new profile
        array.clear(currentPeakStarts)
        array.clear(currentPeakEnds)

    resetBar := bar_index
    // ... rest of reset logic
```

**Step 3.2: Replace Historical Peak Rendering (lines 873-917)**
```pinescript
if showPeakRect and barstate.islast
    // ══════════════════════════════════════════════════════════════════
    // PHASE 1: RENDER HISTORICAL PROFILE PEAKS (FLATTENED ARCHITECTURE)
    // ══════════════════════════════════════════════════════════════════

    int numHistoricalProfiles = array.size(profileStartBars)

    for profIdx = 0 to numHistoricalProfiles - 1
        // Get profile boundaries
        int histStartBar = array.get(profileStartBars, profIdx)
        int histEndBar = array.get(profileEndBars, profIdx)

        // Get peaks using helper function
        [histPeakStarts, histPeakEnds] = getProfilePeaks(profIdx)

        int numPeaks = array.size(histPeakStarts)

        // Render each peak for this historical profile
        for peakIdx = 0 to numPeaks - 1
            // Check box limit before creating
            if array.size(allBoxes) >= MAX_BOXES - 10
                box.delete(array.shift(allBoxes))

            int startRow = array.get(histPeakStarts, peakIdx)
            int endRow = array.get(histPeakEnds, peakIdx)

            // Get price boundaries (same as before)
            if endRow < flashProf.buckets and startRow < flashProf.buckets
                [topPrice, _] = flashProf.getBktBnds(endRow)
                [_, bottomPrice] = flashProf.getBktBnds(startRow)

                int leftBar = histStartBar
                int rightBar = histEndBar + peakExtensionBars

                box peakBox = box.new(
                  left   = leftBar,
                  top    = topPrice,
                  right  = rightBar,
                  bottom = bottomPrice,
                  border_color = peakRectColor,
                  border_width = peakRectBorderWidth,
                  border_style = line.style_solid,
                  bgcolor      = peakRectFillColor,
                  xloc         = xloc.bar_index,
                  extend       = extend.none)

                array.push(allBoxes, peakBox)

    // ══════════════════════════════════════════════════════════════════
    // PHASE 2: RENDER CURRENT PROFILE PEAKS (UNCHANGED)
    // ══════════════════════════════════════════════════════════════════
    // ... (keep existing current profile rendering code)
```

### Phase 4: Testing & Validation

**Test Cases:**

1. **Single Profile with Variable Peaks**
   - Test 2, 5, and 10 peaks
   - Verify correct storage and retrieval

2. **Multiple Profiles**
   - Add 10 profiles sequentially
   - Verify all peaks render correctly

3. **FIFO Limit Enforcement**
   - Add 51 profiles
   - Verify oldest profile removed
   - Verify remaining 50 profiles intact

4. **Edge Cases**
   - Profile with 0 peaks (should handle gracefully)
   - Profile with 1 peak
   - Maximum profile limit boundary

5. **Performance Validation**
   - Measure array sizes with 50 profiles
   - Verify memory usage < 5 KB

### Phase 5: Documentation

**Code Comments:**
- Add comprehensive inline documentation
- Document data structure layout
- Include complexity analysis comments

**Architecture Diagram:**
- Create visual diagram of flattened array structure
- Document index calculation formulas

---

## Memory Usage Analysis

### Detailed Memory Estimation

**Assumptions:**
- 50 historical profiles (maximum)
- Average 5 peaks per profile
- Integer size: 4 bytes

**Profile Metadata Arrays:**
```
profileStartBars:        50 profiles × 4 bytes = 200 bytes
profileEndBars:          50 profiles × 4 bytes = 200 bytes
profilePeakCounts:       50 profiles × 4 bytes = 200 bytes
profilePeakStartIndex:   50 profiles × 4 bytes = 200 bytes
                         ─────────────────────────────────
                         Total Metadata:          800 bytes
```

**Peak Data Arrays (Flattened):**
```
Total peaks:             50 profiles × 5 peaks = 250 peaks
allPeakStartRows:        250 peaks × 4 bytes = 1000 bytes
allPeakEndRows:          250 peaks × 4 bytes = 1000 bytes
                         ─────────────────────────────────
                         Total Peak Data:        2000 bytes
```

**Total Memory Footprint:**
```
Metadata:                800 bytes
Peak Data:              2000 bytes
                        ──────────
TOTAL:                  2800 bytes (2.7 KB)
```

### Comparison with Other Options

| Option | Memory Usage | Efficiency |
|--------|--------------|------------|
| **A: Flattened (Recommended)** | **2.8 KB** | **100%** (no waste) |
| B: Fixed-Size Slots | 4.4 KB | 64% (50% waste) |
| C: String Encoding | ~3.7 KB | 76% (string overhead) |
| D: Parallel Arrays | 4.4 KB | 64% (50% waste) |

**Memory Savings:** Option A uses **36% less memory** than Options B/D.

---

## Performance Benchmarks

### Operation Complexity Analysis

| Operation | Option A | Option B | Option C | Option D |
|-----------|----------|----------|----------|----------|
| Add Profile | O(P) | O(10) | O(P×D) | O(10) |
| Get Peaks | O(P) | O(10) | O(P×L) | O(10) |
| FIFO Cleanup | O(P+N) | O(10) | O(1) | O(10) |
| Render All | O(N×P) | O(N×10) | O(N×P×L) | O(N×10) |

**Legend:**
- P = Peaks per profile (typically 5)
- N = Number of profiles (typically 50)
- D = Digit count in integers (typically 3)
- L = String length (typically 10 chars)

### Real-World Performance Estimates

**Scenario: Rendering 50 Profiles with 5 Peaks Each**

**Option A (Flattened):**
- Get peaks for 50 profiles: 50 × 5 iterations = 250 operations
- Render 250 peaks total
- **Estimated Time:** ~1-2 ms

**Option B (Fixed-Size):**
- Get peaks for 50 profiles: 50 × 10 iterations = 500 operations (checks 500 slots, finds 250 peaks)
- Render 250 peaks total
- **Estimated Time:** ~2-3 ms (2× Option A due to sentinel checks)

**Option C (String):**
- Parse 50 strings × 5 peaks × 10 chars = complex string operations
- **Estimated Time:** ~5-10 ms (string parsing overhead)

**Option D (Parallel):**
- Same as Option B: 500 operations
- **Estimated Time:** ~2-3 ms

**Winner:** Option A provides best real-world performance for typical usage.

---

## Risk Analysis & Mitigation

### Identified Risks

**Risk 1: Index Arithmetic Errors**
- **Impact:** High (data corruption)
- **Probability:** Medium
- **Mitigation:**
  - Encapsulate all index logic in helper functions
  - Add validation checks in retrieval function
  - Comprehensive unit testing
  - Clear code comments explaining index calculations

**Risk 2: FIFO Cleanup Performance**
- **Impact:** Low (happens infrequently)
- **Probability:** Low
- **Mitigation:**
  - O(N) index adjustment is acceptable (N=50)
  - Happens only once per profile reset (~every 100 bars)
  - Total cost: 50 array updates (~1 µs)

**Risk 3: Array Size Limits**
- **Impact:** Medium (could hit Pine Script limits)
- **Probability:** Low
- **Mitigation:**
  - Total array size: ~700 integers (well below Pine Script limits)
  - Monitor array sizes in production
  - Add defensive checks for maximum sizes

**Risk 4: Integration Complexity**
- **Impact:** Medium (implementation effort)
- **Probability:** Medium
- **Mitigation:**
  - Phased implementation approach
  - Thorough testing at each phase
  - Keep existing code as reference during migration
  - Comprehensive documentation

### Validation Strategy

**Phase 1 Validation:**
- Unit test each helper function independently
- Verify correct index calculations with sample data
- Check boundary conditions (0 peaks, 10 peaks)

**Phase 2 Validation:**
- Integration test with real profile data
- Compare rendered output before/after refactoring
- Verify FIFO cleanup maintains data integrity

**Phase 3 Validation:**
- Performance testing with 50 profiles
- Memory usage monitoring
- Visual inspection of rendered peaks

---

## Migration Path

### Backward Compatibility

**Issue:** Existing code uses nested arrays (non-functional).
**Impact:** No backward compatibility concerns since current code doesn't work.

### Migration Steps

1. **Backup Current Code**
   - Create branch: `feature/flattened-peak-storage`
   - Preserve current non-working implementation for reference

2. **Implement New Architecture**
   - Add new data structures (Phase 1)
   - Implement helper functions (Phase 2)
   - Update integration points (Phase 3)

3. **Testing & Validation**
   - Run comprehensive test suite
   - Visual validation on charts
   - Performance benchmarks

4. **Deployment**
   - Merge to main branch
   - Update version number
   - Document changes in changelog

### Rollback Plan

**If Issues Arise:**
- Revert to previous commit (nested arrays removed, basic functionality preserved)
- Disable peak rectangle rendering temporarily
- Fix issues and redeploy

---

## Alternative Considerations

### Future Enhancements

**1. Compressed Index Storage**
If memory becomes critical:
- Store index offsets as deltas instead of absolute values
- Reduces index array size by ~75%

**2. Peak Merging**
Combine adjacent peaks with similar volume:
- Reduces peak count by ~30%
- Maintains visual accuracy

**3. Sampling for Very Old Profiles**
Keep only key peaks for profiles >30 bars old:
- Every other peak
- Only peaks >60% of max volume

### Pine Script Version Compatibility

**Current:** Pine Script v6
**Future:** If Pine Script v7+ adds nested arrays:
- Keep flattened architecture (proven, efficient)
- Or migrate to nested arrays if significant benefits
- Current design will remain functional

---

## Conclusion

**Option A (Flattened Array with Index Mapping)** provides the optimal balance of:

✅ **Compliance** - Fully compatible with Pine Script v6
✅ **Efficiency** - 36% less memory than alternatives
✅ **Performance** - Fast O(P) operations for typical use
✅ **Maintainability** - Clear, well-documented code structure
✅ **Flexibility** - Handles variable peak counts naturally

The implementation plan provides a clear, phased approach with comprehensive testing and validation strategies. Memory usage remains well within Pine Script limits, and performance is excellent for the rendering use case.

---

## Appendix A: Code Structure Overview

```
Historical Profile Storage (Flattened Architecture)
├── Data Structures (5 arrays)
│   ├── profileStartBars        [50 profiles]
│   ├── profileEndBars          [50 profiles]
│   ├── profilePeakCounts       [50 profiles]
│   ├── profilePeakStartIndex   [50 profiles]
│   ├── allPeakStartRows        [~250 peaks]
│   └── allPeakEndRows          [~250 peaks]
│
├── Helper Functions (3 functions)
│   ├── storeHistoricalProfile()    - Add new profile
│   ├── getProfilePeaks()           - Retrieve profile peaks
│   └── cleanupOldestProfile()      - FIFO management
│
└── Integration Points (2 locations)
    ├── Profile capture on anchor change
    └── Historical peak rendering
```

---

## Appendix B: Pine Script Compatibility Reference

**Supported Array Operations:**
- `array.new<int>()` ✅
- `array.push()` ✅
- `array.get()` ✅
- `array.set()` ✅
- `array.shift()` ✅
- `array.size()` ✅
- `array.clear()` ✅

**NOT Supported:**
- `array.new<array<int>>()` ❌
- `array<array<type>>` ❌
- Any nested array structures ❌

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-11 | System Architecture Designer | Initial comprehensive design document |

---

**End of Document**
