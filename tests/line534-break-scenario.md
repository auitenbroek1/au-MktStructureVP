# Test Scenario: Line 534 Change Breaking Profile Rendering

## Critical Issue: Profile Rendering Uses Wrong Start Bar

### The Problem

Line 534 was changed from using `resetBar - delay` to `prevResetBar`, which breaks profile rendering because:

1. **Line 534** stores historical profile boundaries (not used for rendering)
2. **Line 648** recalculates `startBar` for ACTUAL profile rendering
3. The line 648 calculation is **NEVER executed** when line 534 is changed to `prevResetBar`

---

## Test Scenario: Profile Reset at Bar 200

### Setup Parameters
```pine
delay = 10  // Lookahead delay
bar_index = 200  // Current bar where profile resets
```

### Initial State (Before Bar 200)
```
Previous profile:
  - Started at bar 100
  - resetBar = 100
  - prevResetBar = 100
```

---

## Step-by-Step Execution Trace

### Bar 200: Profile Reset Detected

#### Step 1: Values at Start of Reset Block (Line 533)
```
resetBar = 100        // From previous profile
prevResetBar = 100    // Set during last reset
bar_index = 200       // Current bar
delay = 10
```

#### Step 2: BEFORE CHANGE (Original Code - Line 534)

```pine
Line 534: startBar := resetBar - delay
```

**Calculation:**
```
startBar = resetBar - delay
startBar = 100 - 10
startBar = 90
```

**Result:** `startBar = 90` ✅ CORRECT
- Profile should render from bar 90 (accounting for lookahead delay)

---

#### Step 3: AFTER CHANGE (Modified Code - Line 534)

```pine
Line 534: startBar := prevResetBar
```

**Calculation:**
```
startBar = prevResetBar
startBar = 100
```

**Result:** `startBar = 100` ❌ WRONG
- Profile renders from bar 100 instead of bar 90
- **Missing 10 bars of volume data** (bars 90-99)

---

### Step 4: Historical Storage Block Executes (Lines 541-589)

This stores the PREVIOUS profile (bar 100-199) into historical arrays:
```pine
array.push(profileStartBars, startBar)  // Pushes either 90 or 100
array.push(profileEndBars, endBar)      // Pushes 199
```

**Impact:**
- ✅ BEFORE: Stores `startBar = 90` (includes delay)
- ❌ AFTER: Stores `startBar = 100` (missing 10 bars)

---

### Step 5: Update State Variables (Lines 593-595)

```pine
Line 593: if not na(resetBar)
Line 594:     prevResetBar := resetBar
Line 595: resetBar := bar_index
```

**After execution:**
```
prevResetBar = 100   // Updated from old resetBar
resetBar = 200       // Updated to current bar
```

---

### Step 6: **CRITICAL SECTION** - Profile Rendering (Lines 647-648)

```pine
Line 647: endBar   := bar_index
Line 648: startBar := resetBar - delay
```

#### BEFORE CHANGE (Original Code):

**Line 648 recalculates startBar for rendering:**
```
startBar = resetBar - delay
startBar = 200 - 10
startBar = 190
```

**Profile Rendering (Line 731):**
```pine
barWidth = (endBar - startBar + 1) * profWidth / 100.0
barWidth = (200 - 190 + 1) * profWidth / 100.0
barWidth = 11 * profWidth / 100.0  ✅ CORRECT
```

**Rendering uses bars 190-200** (11 bars with delay accounted)

---

#### AFTER CHANGE (Modified Code):

**Line 534 set startBar = 100, Line 648 NEVER EXECUTES because it's in `if endBar >= startBar` block**

Wait, let me check this more carefully...

**Actually, Line 648 DOES execute in `barstate.islast` block:**

```pine
Line 629: if barstate.islast
Line 647:     endBar   := bar_index
Line 648:     startBar := resetBar - delay
```

**So Line 648 recalculates:**
```
startBar = resetBar - delay
startBar = 200 - 10
startBar = 190
```

**This means Line 534 change ONLY affects historical storage, NOT rendering!**

---

## The Real Break: Historical Profile Storage

### Impact Analysis

The line 534 change breaks **historical profile storage**, not real-time rendering:

#### BEFORE CHANGE:
```
Historical Array Storage (Bar 200):
  profileStartBars[n] = 90   ✅ Includes lookahead delay
  profileEndBars[n] = 199
  Profile spans bars 90-199 (110 bars total)
```

#### AFTER CHANGE:
```
Historical Array Storage (Bar 200):
  profileStartBars[n] = 100  ❌ Missing lookahead data
  profileEndBars[n] = 199
  Profile spans bars 100-199 (100 bars total)
```

**Missing Data:** 10 bars (bars 90-99) are excluded from historical storage

---

## Visual Demonstration

### Timeline with delay = 10

```
Bar:     90   91   92   ... 99   100  101  ... 199  200
         |------------------------|-------------|    |
         |    MISSING DATA        |  STORED     |    | Reset
         | (bars 90-99)          | (100-199)   |    |
         |                        |             |    |
         ▼                        ▼             ▼    ▼
BEFORE:  [====INCLUDED====][=======STORED======]
AFTER:   [XXX EXCLUDED XXX][=======STORED======]
                           ^
                           startBar moved here
```

### What Gets Stored

**BEFORE (Correct):**
```
Profile stored as bars 90-199:
  - Includes 10 bars of lookahead data (90-99)
  - Volume data is complete
  - POC/VA calculations accurate
```

**AFTER (Broken):**
```
Profile stored as bars 100-199:
  - Missing 10 bars of lookahead data (90-99)
  - Incomplete volume distribution
  - POC/VA calculations inaccurate
```

---

## Impact on Historical Profile Rendering

When user scrolls back to view stored profiles:

### Original Code (Correct):
```pine
// Rendering historical profile from arrays
startBar = profileStartBars[i]  // Returns 90
endBar = profileEndBars[i]      // Returns 199
barWidth = (199 - 90 + 1) * profWidth / 100.0 = 110 bars
```

Profile renders across 110 bars with complete volume data.

### Modified Code (Broken):
```pine
// Rendering historical profile from arrays
startBar = profileStartBars[i]  // Returns 100
endBar = profileEndBars[i]      // Returns 199
barWidth = (199 - 100 + 1) * profWidth / 100.0 = 100 bars
```

Profile renders across only 100 bars, **missing 10 bars** of volume data from the beginning.

---

## Proof of Break

### Concrete Example Values

**Setup:**
```
delay = 10
Profile 1: Bar 0-99 (resetBar = 0, ends at bar 99)
Profile 2: Bar 100-199 (resetBar = 100, ends at bar 199)
Profile 3: Bar 200-299 (resetBar = 200, starts at bar 200)
```

**At Bar 200 (Profile 2 storage, Profile 3 start):**

| Variable | BEFORE Change | AFTER Change | Difference |
|----------|---------------|--------------|------------|
| `resetBar` | 100 | 100 | Same |
| `prevResetBar` | 100 | 100 | Same |
| `delay` | 10 | 10 | Same |
| **Line 534 `startBar`** | **90** | **100** | **-10 bars** |
| `endBar` | 199 | 199 | Same |
| Stored bars | 90-199 (110) | 100-199 (100) | -10 bars |
| **Line 648 `startBar`** | **190** | **190** | Same |
| Real-time render bars | 190-200 (11) | 190-200 (11) | Same |

**Summary:**
- ✅ Real-time rendering: NOT affected (Line 648 recalculates)
- ❌ Historical storage: BROKEN (Line 534 excludes lookahead bars)
- ❌ Historical rendering: Shows incomplete profiles (missing first 10 bars)

---

## Testing Instructions

### Manual Test

1. **Apply the change:**
   ```pine
   Line 534: startBar := prevResetBar  // Modified
   ```

2. **Load chart with visible profiles**

3. **Check current profile:**
   - Should render correctly (Line 648 handles it)

4. **Scroll back to historical profiles:**
   - Profiles will appear shorter at their start
   - Missing first 10 bars of volume data
   - POC/VA may be shifted due to incomplete data

5. **Compare with original:**
   ```pine
   Line 534: startBar := resetBar - delay  // Original
   ```
   - Historical profiles now show complete data
   - Full 110-bar span for each profile

---

## Conclusion

The line 534 change breaks **historical profile storage and rendering** by excluding the lookahead delay period from stored profile boundaries. While real-time profiles render correctly (because Line 648 recalculates `startBar`), historical profiles become incomplete and inaccurate.

**Fix Required:** Keep original calculation at Line 534:
```pine
Line 534: startBar := resetBar - delay
```

This ensures historical profiles store the complete data range including the lookahead delay period.
