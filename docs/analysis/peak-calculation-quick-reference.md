# Peak Rectangle Calculation - Quick Reference Card

## The 50% Rule in 3 Steps

### Step 1: Find Maximum Volume
```
Scan all rows → Find row with highest volume
Example: Row 4 has 330 volume (max)
```

### Step 2: Calculate Threshold
```
Threshold = Max Volume × 0.5
Example: 330 × 0.5 = 165
```

### Step 3: Identify Peak Zones
```
Mark all CONTINUOUS rows where volume ≥ threshold
Example: Rows 3-8 all have volume ≥ 165
```

## Visual Example

```
Price    Volume  Above 165?  Peak Zone
$100.40  170     YES         ← Top
$100.35  210     YES         |
$100.30  300     YES         |
$100.25  270     YES         | Rectangle
$100.20  330     YES (MAX)   | spans
$100.15  220     YES         ← Bottom
$100.10  190     YES         |
$100.05  150     NO          (excluded)
$100.00  80      NO          (excluded)
```

## Key Points

| Aspect | Explanation |
|--------|-------------|
| **What is 50%?** | 50% of the MAXIMUM volume in a single row |
| **NOT 50% of...** | Total profile volume, price range, or time |
| **Rectangle Top** | Upper price boundary of highest row in peak |
| **Rectangle Bottom** | Lower price boundary of lowest row in peak |
| **Multiple Peaks** | Each continuous zone gets separate rectangle |
| **Code Location** | Line 699: threshold calculation |

## Example Calculations

### Profile A: Narrow Peak
```
Max Volume: 500
Threshold: 250
Peak Rows: 12-15 (4 rows)
Rectangle: $99.60-$99.75 (tight zone)
```

### Profile B: Wide Peak
```
Max Volume: 800
Threshold: 400
Peak Rows: 5-20 (16 rows)
Rectangle: $98.25-$99.05 (broad zone)
```

### Profile C: Multiple Peaks
```
Max Volume: 600
Threshold: 300
Peak 1: Rows 3-7 → Rectangle 1: $100.15-$100.35
Peak 2: Rows 15-18 → Rectangle 2: $100.90-$101.05
```

## Common Questions

**Q: Can I change the 50%?**
A: Currently no (hardcoded). Future enhancement possible.

**Q: What if no rows exceed threshold?**
A: No peak rectangles drawn (only profile boundary).

**Q: Does it work with Delta display mode?**
A: Yes! Uses 50% of max absolute delta instead.

**Q: How does this differ from Value Area?**
A: VA = 70% of TOTAL volume (statistical distribution)
   Peaks = Continuous zones ≥50% of MAX volume (concentration zones)

## Code Snippet
```pine
// Line 699: Threshold calculation
float volumeThreshold = maxVol * 0.5

// Lines 728-731: Peak detection
if rowVol >= volumeThreshold and not inPeak
    peakStart := i        // Mark start of peak zone
    inPeak := true
```

## Trading Interpretation

- **Peak = Institutional Footprint**: Heavy trading concentration
- **Multiple Peaks = Price Disagreement**: Different value zones
- **Peak Width = Acceptance Range**: Tight vs. broad consensus
- **Use as Support/Resistance**: Price respects high-volume zones

---

**Full Documentation:** See `peak-rectangle-calculation-explained.md`
