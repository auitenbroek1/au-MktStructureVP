# Architecture Analysis: Flattened Array Index Corruption

## Executive Summary

**ROOT CAUSE IDENTIFIED**: The FIFO cleanup logic does NOT adjust `profilePeakStartIdx` values after removing elements from flattened arrays, causing all remaining profile indices to point to incorrect positions.

**Impact**: Array out-of-bounds errors (accessing index 216 when size is 209) due to stale index references.

---

## Architecture Overview

### Data Structure Design
The system uses a **flattened array architecture** with two layers:

**Layer 1: Profile Metadata Arrays**
- `profileStartBars[]` - Start bar index for each profile
- `profileEndBars[]` - End bar index for each profile
- `profilePeakCounts[]` - Number of peaks in each profile
- `profilePeakStartIdx[]` - **Starting index into flattened arrays**

**Layer 2: Flattened Peak Data Arrays**
- `allPeakStartRows[]` - Concatenated peak start rows for ALL profiles
- `allPeakEndRows[]` - Concatenated peak end rows for ALL profiles

### Current Storage Logic (Lines 528-542)

```pine
// Store profile metadata
array.push(profileStartBars, startBar)
array.push(profileEndBars, endBar)
array.push(profilePeakCounts, peakCount)
array.push(profilePeakStartIdx, array.size(allPeakStartRows))  // ← Current size = start index

// Append peaks to flattened arrays
for i = 0 to peakCount - 1
    array.push(allPeakStartRows, array.get(currentPeakStarts, i))
    array.push(allPeakEndRows, array.get(currentPeakEnds, i))
```

### Current FIFO Cleanup Logic (Lines 544-562)

```pine
if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
    // Get oldest profile's peak count
    int oldestPeakCount = array.shift(profilePeakCounts)

    // Remove oldest profile metadata
    array.shift(profileStartBars)
    array.shift(profileEndBars)
    array.shift(profilePeakStartIdx)  // ← REMOVES oldest index but doesn't adjust remaining!

    // Remove oldest peaks from flattened arrays
    for i = 0 to oldestPeakCount - 1
        array.shift(allPeakStartRows)  // ← Shifts all remaining elements left
        array.shift(allPeakEndRows)
```

### Retrieval Logic (Lines 884-898)

```pine
for profIdx = 0 to numHistoricalProfiles - 1
    int peakStartIndex = array.get(profilePeakStartIdx, profIdx)
    int peakCount = array.get(profilePeakCounts, profIdx)

    for peakIdx = 0 to peakCount - 1
        // Uses STALE indices → out-of-bounds access!
        int startRow = array.get(allPeakStartRows, peakStartIndex + peakIdx)
        int endRow = array.get(allPeakEndRows, peakStartIndex + peakIdx)
```

---

## Detailed Scenario Trace

### Initial State (50 Profiles)

```
Profile 0: startIdx=0,  peakCount=3  (peaks at flattened indices 0,1,2)
Profile 1: startIdx=3,  peakCount=5  (peaks at flattened indices 3,4,5,6,7)
Profile 2: startIdx=8,  peakCount=2  (peaks at flattened indices 8,9)
Profile 3: startIdx=10, peakCount=4  (peaks at flattened indices 10,11,12,13)
...
Profile 49: startIdx=200, peakCount=9 (peaks at flattened indices 200-208)

Metadata Arrays (size=50):
  profilePeakStartIdx = [0, 3, 8, 10, ..., 200]
  profilePeakCounts   = [3, 5, 2, 4, ..., 9]

Flattened Arrays (size=209):
  allPeakStartRows = [row0, row1, row2, row3, ..., row208]
  allPeakEndRows   = [row0, row1, row2, row3, ..., row208]
```

### Add Profile 51 (Triggers FIFO Cleanup)

**Step 1: Store new profile (Profile 51) BEFORE cleanup**
```
Profile 51: startIdx=209, peakCount=7 (peaks at indices 209-215)

Flattened Arrays (size=216):
  allPeakStartRows = [row0, row1, ..., row208, row209, ..., row215]
```

**Step 2: FIFO cleanup triggered (size=51 > MAX_HISTORICAL_PROFILES=50)**
```pine
// Remove Profile 0 (oldest)
oldestPeakCount = array.shift(profilePeakCounts)  // Returns 3
array.shift(profileStartBars)
array.shift(profileEndBars)
array.shift(profilePeakStartIdx)  // Removes startIdx=0

// Remove 3 peaks from flattened arrays
for i = 0 to 2  // oldestPeakCount - 1
    array.shift(allPeakStartRows)  // Removes indices 0,1,2
    array.shift(allPeakEndRows)
```

### Post-Cleanup State (THE BUG)

```
Flattened Arrays (size=213):
  allPeakStartRows = [row3, row4, row5, ..., row215]
                      ↑ Now at index 0 (was index 3)

Metadata Arrays (size=50):
  profilePeakStartIdx = [3, 8, 10, ..., 200, 209]
                         ↑ STALE! Should be 0, not 3!
  profilePeakCounts   = [5, 2, 4, ..., 9, 7]
```

**The Corruption:**
- Profile 1 (now at profIdx=0): `startIdx=3`, `peakCount=5`
  - Tries to access indices: 3, 4, 5, 6, 7
  - **SHOULD access**: 0, 1, 2, 3, 4 (shifted left by 3)

- Profile 51 (now at profIdx=49): `startIdx=209`, `peakCount=7`
  - Tries to access indices: 209, 210, 211, 212, 213, 214, 215
  - Array size is only 213
  - **ERROR**: Accessing index 214-215 when array size is 213

### Why Error Says "216 accessed, size 209"

After multiple cleanup cycles:
- Accumulated index drift: ~7 indices too high
- Trying to access: `startIdx=209 + peakIdx=7 = 216`
- Actual array size: 209 (after multiple cleanups removed ~7 elements total)

---

## Mathematical Proof of Index Drift

### General Formula

After removing M profiles with total peak count N:

```
Correct Index Adjustment:
  new_startIdx[i] = old_startIdx[i] - N

Current Behavior:
  new_startIdx[i] = old_startIdx[i]  (NO ADJUSTMENT!)

Index Drift = N peaks (cumulative)
```

### Cumulative Effect

Each cleanup cycle:
1. Removes `oldestPeakCount` peaks from flattened arrays
2. All remaining indices shift left by `oldestPeakCount`
3. But `profilePeakStartIdx[]` values remain unchanged
4. Drift accumulates: `total_drift += oldestPeakCount`

After K cleanup cycles:
```
Total Drift = Σ(oldestPeakCount[k]) for k=1 to K
```

---

## Fix Algorithm

### Solution: Adjust All Remaining Indices After Cleanup

```pine
if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
    // Get oldest profile's peak count
    int oldestPeakCount = array.shift(profilePeakCounts)

    // Remove oldest profile metadata
    array.shift(profileStartBars)
    array.shift(profileEndBars)
    array.shift(profilePeakStartIdx)  // Remove oldest index

    // Remove oldest peaks from flattened arrays
    for i = 0 to oldestPeakCount - 1
        if array.size(allPeakStartRows) > 0
            array.shift(allPeakStartRows)
            array.shift(allPeakEndRows)

    // ══════════════════════════════════════════════════════════════════
    // FIX: Adjust all remaining profilePeakStartIdx values
    // ══════════════════════════════════════════════════════════════════
    for i = 0 to array.size(profilePeakStartIdx) - 1
        int currentIdx = array.get(profilePeakStartIdx, i)
        array.set(profilePeakStartIdx, i, currentIdx - oldestPeakCount)

    // Delete associated boxes
    for i = 0 to math.min(oldestPeakCount, array.size(allBoxes)) - 1
        if array.size(allBoxes) > 0
            box.delete(array.shift(allBoxes))
```

### Key Changes

1. **After removing peaks**: Iterate through ALL remaining `profilePeakStartIdx` values
2. **Subtract shift amount**: Each index must be decremented by `oldestPeakCount`
3. **Preserves relative ordering**: All profiles shift left by the same amount

### Correctness Verification

**Before Cleanup:**
```
Profile 1: startIdx=3, peakCount=5
Profile 2: startIdx=8, peakCount=2
Profile 3: startIdx=10, peakCount=4
```

**After Cleanup (removed 3 peaks from Profile 0):**
```
Profile 1: startIdx=0 (3-3), peakCount=5 ✓
Profile 2: startIdx=5 (8-3), peakCount=2 ✓
Profile 3: startIdx=7 (10-3), peakCount=4 ✓
```

**Flattened array indices now match:**
```
Indices 0-4: Profile 1's 5 peaks
Indices 5-6: Profile 2's 2 peaks
Indices 7-10: Profile 3's 4 peaks
```

---

## Alternative Solutions

### Option 1: Circular Buffer (More Complex)
Replace shift operations with circular indexing:
- Maintain head/tail pointers
- No actual element removal
- Higher memory usage but O(1) cleanup

### Option 2: Batch Cleanup (Reduces Frequency)
Only cleanup when significantly over limit:
```pine
if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES * 1.5
    // Remove multiple profiles at once
    // Reduces adjustment overhead
```

### Option 3: Separate Index Tracking (Cleaner Architecture)
Maintain a separate mapping array:
```pine
var array<int> profileIndexMapping = array.new<int>()
// Maps logical profile index → flattened array start index
```

---

## Performance Impact

### Current Fix Overhead
- **Time Complexity**: O(N) where N = number of remaining profiles
- **Frequency**: Once per cleanup (every profile addition after limit)
- **Typical N**: ~50 profiles
- **Impact**: Negligible (~50 array operations per cleanup)

### Comparison to Alternative
The index adjustment loop is the **simplest and most efficient** solution:
- No additional memory overhead
- Minimal code changes
- Clear semantic meaning
- Self-documenting fix

---

## Testing Strategy

### Unit Test Scenarios

1. **Basic FIFO Test**
   - Add 51 profiles
   - Verify indices after each cleanup
   - Check no out-of-bounds access

2. **Multiple Cleanup Cycles**
   - Add 100 profiles (triggers 50 cleanups)
   - Verify cumulative drift correction
   - Validate flattened array consistency

3. **Variable Peak Counts**
   - Mix profiles with 1, 5, 10, 20 peaks
   - Verify different shift amounts handled correctly
   - Check edge cases (empty profiles)

4. **Box Rendering Validation**
   - After cleanup, render all historical peaks
   - Verify prices match expected row boundaries
   - No visual artifacts or missing peaks

### Validation Assertions

```pine
// Add after cleanup loop
if array.size(profilePeakStartIdx) > 0
    // Last profile's end index should equal flattened array size
    int lastProfileIdx = array.size(profilePeakStartIdx) - 1
    int lastStartIdx = array.get(profilePeakStartIdx, lastProfileIdx)
    int lastPeakCount = array.get(profilePeakCounts, lastProfileIdx)
    int expectedSize = lastStartIdx + lastPeakCount

    if array.size(allPeakStartRows) != expectedSize
        runtime.error("Index mismatch: expected=" + str.tostring(expectedSize) +
                      " actual=" + str.tostring(array.size(allPeakStartRows)))
```

---

## Recommendation

**IMPLEMENT FIX IMMEDIATELY**

1. Add index adjustment loop to FIFO cleanup (lines 558-560)
2. Add validation assertions for debugging
3. Test with extended historical data (100+ profiles)
4. Monitor for any edge cases during live trading

**Priority**: CRITICAL - Current code will fail consistently after MAX_HISTORICAL_PROFILES limit reached.

---

## Appendix: Code Location Reference

- **Storage Logic**: Lines 528-542
- **FIFO Cleanup**: Lines 544-562
- **Retrieval Logic**: Lines 884-898
- **Array Declarations**: Lines 455-460

**File**: `/Users/aaronuitenbroek/hive-projects/i10iQ-projects/au-MktStructureVP/src/au-mktStructureVP-FULL.pine`
