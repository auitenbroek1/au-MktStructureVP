# Rectangle Backward-Shift Concept Analysis

## Test Scenario Overview

This document analyzes the **rectangle backward-shift concept** where each rectangle should start at the bar where the previous profile started, effectively shifting rectangles backward by one profile period.

---

## Current Implementation Analysis

### Key Code Sections

**Historical Profile Storage (Lines 461-468):**
```pinescript
var array<int> profileStartBars = array.new<int>()
var array<int> profileEndBars = array.new<int>()
```

**Historical Peak Rendering (Lines 925-977):**
```pinescript
for profIdx = 0 to numHistoricalProfiles - 1
    int histStartBar = array.get(profileStartBars, profIdx)
    int histEndBar = array.get(profileEndBars, profIdx)
    // ...
    int leftBar = histStartBar  // Currently uses profile's own start
    int rightBar = histEndBar + peakExtensionBars
```

**Current Profile Rendering (Lines 1006-1061):**
```pinescript
// Time coordinates
int leftBar = startBar  // Currently uses current profile's start
int rightBar = bar_index + peakExtensionBars
```

---

## Visual Example: BEFORE vs AFTER Shift

### BEFORE SHIFT (Current Implementation)

```
Timeline:        0    50   100  150  200  250  300  350  400  450  500
                 |----|----|----|----|----|----|----|----|----|----|

Profile 0:       [===================]
  Start: bar 0
  End:   bar 99
  Rectangle 0:   [==========================]
  Rect Start: bar 0   (uses own start)
  Rect End:   bar 149 (end + 50 bars extension)

Profile 1:                 [===================]
  Start: bar 100
  End:   bar 199
  Rectangle 1:             [==========================]
  Rect Start: bar 100  (uses own start)
  Rect End:   bar 249  (end + 50 bars extension)

Profile 2:                           [===================]
  Start: bar 200
  End:   bar 299
  Rectangle 2:                       [==========================]
  Rect Start: bar 200  (uses own start)
  Rect End:   bar 349  (end + 50 bars extension)

Current Profile:                               [===================]
  Start: bar 300
  End:   bar 350 (developing)
  Rectangle C:                                 [==========================]
  Rect Start: bar 300  (uses own start)
  Rect End:   bar 400  (current + 50 bars extension)
```

**Alignment Analysis:**
- ✅ Each rectangle starts where its profile starts
- ❌ No backward shift - rectangles align with their own profiles
- Result: Rectangles are synchronous with profiles

---

### AFTER SHIFT (Proposed Implementation)

```
Timeline:        0    50   100  150  200  250  300  350  400  450  500
                 |----|----|----|----|----|----|----|----|----|----|

Profile 0:       [===================]
  Start: bar 0
  End:   bar 99
  Rectangle 0:   ❌ SKIP (no previous profile)
  OR:            [==========================]  (Option 2: use bar 0)

Profile 1:                 [===================]
  Start: bar 100
  End:   bar 199
  Rectangle 1:   [==========================]
  Rect Start: bar 0    (uses Profile 0 start - SHIFTED)
  Rect End:   bar 249  (end + 50 bars extension)

Profile 2:                           [===================]
  Start: bar 200
  End:   bar 299
  Rectangle 2:             [==========================]
  Rect Start: bar 100  (uses Profile 1 start - SHIFTED)
  Rect End:   bar 349  (end + 50 bars extension)

Current Profile:                               [===================]
  Start: bar 300
  End:   bar 350 (developing)
  Rectangle C:                       [==========================]
  Rect Start: bar 200  (uses Profile 2 start - SHIFTED)
  Rect End:   bar 400  (current + 50 bars extension)
```

**Alignment Analysis:**
- ✅ Each rectangle starts where the previous profile started
- ✅ Backward shift creates temporal lag
- ✅ Rectangle 1 covers Profile 0 and part of Profile 1
- ✅ Rectangle 2 covers Profile 1 and part of Profile 2
- ✅ Current rectangle covers Profile 2 and current profile

---

## Mathematical Formula for Shifted leftBar

### Historical Profiles (Already Captured)

For historical profile at index `profIdx` (0-based):

```pinescript
// Current implementation (NO SHIFT):
int leftBar = array.get(profileStartBars, profIdx)

// Proposed implementation (WITH SHIFT):
int leftBar = profIdx == 0
    ? array.get(profileStartBars, 0)  // Option 2: Use own start for first profile
    : array.get(profileStartBars, profIdx - 1)  // Use previous profile's start
```

**Concrete Example (Historical):**
```
profIdx = 0: leftBar = profileStartBars[0] = bar 0   (no previous)
profIdx = 1: leftBar = profileStartBars[0] = bar 0   (shifted to Profile 0)
profIdx = 2: leftBar = profileStartBars[1] = bar 100 (shifted to Profile 1)
profIdx = 3: leftBar = profileStartBars[2] = bar 200 (shifted to Profile 2)
```

---

### Current/Developing Profile

For the current profile being drawn in real-time:

```pinescript
// Current implementation (NO SHIFT):
int leftBar = startBar  // Uses current profile's own start

// Proposed implementation (WITH SHIFT):
int numHistoricalProfiles = array.size(profileStartBars)
int leftBar = numHistoricalProfiles == 0
    ? startBar  // First profile ever - use own start
    : array.get(profileStartBars, numHistoricalProfiles - 1)  // Use last completed profile's start
```

**Concrete Example (Current Profile):**
```
Scenario 1: No historical profiles yet
  leftBar = startBar = bar 0

Scenario 2: 1 historical profile (bars 0-99)
  leftBar = profileStartBars[0] = bar 0
  Current profile starts at bar 100
  Rectangle extends from bar 0 to bar 150

Scenario 3: 3 historical profiles (0-99, 100-199, 200-299)
  leftBar = profileStartBars[2] = bar 200
  Current profile starts at bar 300
  Rectangle extends from bar 200 to bar 350
```

---

## Edge Case Analysis: profIdx = 0 (First Profile)

### Option 1: Skip First Rectangle

```pinescript
if profIdx == 0
    continue  // Skip rendering rectangle for first profile
```

**Pros:**
- Clean logic - no special handling needed
- Avoids ambiguity about where first rectangle should start

**Cons:**
- First profile has no visual frame
- User might wonder why first profile is missing rectangle
- Inconsistent visualization (some profiles have rectangles, first doesn't)

**Visual Impact:**
```
Profile 0:       [===================]  ← No rectangle frame
Profile 1:       [==========================]
Profile 2:       [==========================]
```

---

### Option 2: Use profIdx=0 Start (No Shift) ⭐ RECOMMENDED

```pinescript
int leftBar = profIdx == 0
    ? array.get(profileStartBars, 0)
    : array.get(profileStartBars, profIdx - 1)
```

**Pros:**
- All profiles get rectangles (consistent visualization)
- First profile rectangle starts at bar 0 (logical)
- Simple conditional logic
- **Graceful degradation:** shift applies where possible, no shift for first

**Cons:**
- First rectangle not "shifted" (but there's nothing to shift to)
- Slight inconsistency in behavior (but acceptable edge case)

**Visual Impact:**
```
Profile 0:       [==========================]  ← Starts at own position
Profile 1:       [==========================]  ← Starts at Profile 0 position (shifted)
Profile 2:       [==========================]  ← Starts at Profile 1 position (shifted)
```

**Code Implementation:**
```pinescript
// Historical profiles
for profIdx = 0 to numHistoricalProfiles - 1
    // Calculate shifted leftBar
    int leftBar = profIdx == 0
        ? array.get(profileStartBars, 0)  // First profile: use own start
        : array.get(profileStartBars, profIdx - 1)  // Others: shift backward

    int rightBar = array.get(profileEndBars, profIdx) + peakExtensionBars
    // ... rest of rendering

// Current profile
int numHistoricalProfiles = array.size(profileStartBars)
int leftBar = numHistoricalProfiles == 0
    ? startBar  // First profile ever
    : array.get(profileStartBars, numHistoricalProfiles - 1)  // Shift to last completed
```

---

### Option 3: Force to Bar 0

```pinescript
int leftBar = profIdx == 0 ? 0 : array.get(profileStartBars, profIdx - 1)
```

**Pros:**
- First rectangle always starts at absolute beginning
- Maximum historical context for first profile

**Cons:**
- May extend rectangle into pre-data territory (bars before data starts)
- Could create very wide rectangle if Profile 0 doesn't start at bar 0
- Less intuitive - why would first rectangle not match profile's actual start?

**Visual Impact:**
```
Profile 0 starts at bar 50:
Profile 0:            [===================]
Rectangle 0:  [=============================]  ← Extends backward to bar 0
              ← May include "dead space" with no data
```

---

## Verification Test Cases

### Test Case 1: Three Sequential Profiles

**Setup:**
```
Profile 0: bars 0-99    (start: 0,   end: 99)
Profile 1: bars 100-199 (start: 100, end: 199)
Profile 2: bars 200-299 (start: 200, end: 299)
Current:   bars 300-350 (start: 300, end: 350)
```

**Expected Rectangle Coordinates (Option 2 with 50-bar extension):**

| Profile | Rectangle leftBar | Rectangle rightBar | Span |
|---------|------------------|-------------------|------|
| Profile 0 | 0 (own start) | 149 (99 + 50) | 150 bars |
| Profile 1 | 0 (shifted) | 249 (199 + 50) | 250 bars |
| Profile 2 | 100 (shifted) | 349 (299 + 50) | 250 bars |
| Current | 200 (shifted) | 400 (350 + 50) | 201 bars |

**Verification Logic:**
```pinescript
// Profile 0
assert(leftBar == 0)  // Uses own start (edge case)
assert(rightBar == 149)

// Profile 1
assert(leftBar == 0)  // Shifted to Profile 0 start
assert(rightBar == 249)

// Profile 2
assert(leftBar == 100)  // Shifted to Profile 1 start
assert(rightBar == 349)

// Current
assert(leftBar == 200)  // Shifted to Profile 2 start
assert(rightBar == 400)
```

---

### Test Case 2: First Profile Only

**Setup:**
```
Current Profile: bars 0-50 (no historical profiles yet)
```

**Expected Rectangle Coordinates:**

| Profile | Rectangle leftBar | Rectangle rightBar | Shift Applied? |
|---------|------------------|-------------------|----------------|
| Current | 0 (own start) | 100 (50 + 50) | No (nothing to shift to) |

**Verification Logic:**
```pinescript
assert(numHistoricalProfiles == 0)
assert(leftBar == 0)  // Uses own start
assert(rightBar == 100)
```

---

### Test Case 3: Gap Between Profiles

**Setup:**
```
Profile 0: bars 0-99    (start: 0)
Profile 1: bars 200-299 (start: 200)  ← 100-bar gap
Current:   bars 400-450 (start: 400)  ← 100-bar gap
```

**Expected Rectangle Coordinates (Option 2):**

| Profile | Rectangle leftBar | Rectangle rightBar | Visual Impact |
|---------|------------------|-------------------|---------------|
| Profile 0 | 0 | 149 | Normal |
| Profile 1 | 0 (shifted) | 349 | ✅ Rectangle spans gap (0-349) |
| Current | 200 (shifted) | 500 | ✅ Rectangle spans gap (200-500) |

**Key Observation:**
- Backward shift creates wide rectangles that span gaps
- This may be intentional (shows temporal relationship)
- Or may be undesired (rectangle covers empty space)

---

## Implementation Checklist

### Code Changes Required

**Location 1: Historical Profile Rendering (Line ~962)**

```pinescript
// BEFORE:
int leftBar = histStartBar

// AFTER:
int leftBar = profIdx == 0
    ? histStartBar  // First profile: use own start
    : array.get(profileStartBars, profIdx - 1)  // Others: shift backward
```

**Location 2: Current Profile Rendering (Line ~1022)**

```pinescript
// BEFORE:
int leftBar = startBar

// AFTER:
int numHistoricalProfiles = array.size(profileStartBars)
int leftBar = numHistoricalProfiles == 0
    ? startBar  // No historical profiles: use own start
    : array.get(profileStartBars, numHistoricalProfiles - 1)  // Shift to last completed
```

---

## Recommended Implementation

**Use Option 2 (Use profIdx=0 start for first profile)** because:

1. ✅ All profiles get rectangles (consistent visualization)
2. ✅ Graceful handling of edge case (first profile can't shift)
3. ✅ Simple and understandable logic
4. ✅ No gaps in visualization (every profile has frame)
5. ✅ Backward shift applies where meaningful (profIdx ≥ 1)
6. ✅ Current profile correctly shifts to last historical profile

---

## Testing Strategy

### Unit Test Cases

1. **First profile only** (no shift possible)
   - Verify leftBar = profileStartBars[0]

2. **Second profile** (first shift)
   - Verify leftBar = profileStartBars[0] (shifted)

3. **Multiple profiles** (full shift chain)
   - Verify each rectangle starts at previous profile's start

4. **Current profile with history**
   - Verify leftBar = profileStartBars[last]

5. **Current profile without history**
   - Verify leftBar = startBar (no shift)

### Visual Inspection Checklist

- [ ] Rectangle 0 starts at bar 0 (or first profile start)
- [ ] Rectangle 1 starts where Profile 0 started
- [ ] Rectangle N starts where Profile N-1 started
- [ ] Current rectangle starts where last historical profile started
- [ ] All rectangles have consistent rightBar calculation (end + extension)
- [ ] No missing rectangles (all profiles have frames)
- [ ] Rectangles extend correctly into future (extension bars)

---

## Summary

**Current Behavior:**
- Each rectangle starts at its own profile's start bar
- No temporal shift - synchronous alignment

**Proposed Behavior (Option 2):**
- Each rectangle (except first) starts at previous profile's start bar
- Creates backward temporal shift
- First profile uses own start (edge case)
- Visual effect: rectangles "lag behind" their profiles by one period

**Implementation Complexity:**
- Low - requires 2 simple conditional statements
- High readability - clear intent with comments
- Safe - handles all edge cases gracefully
