# Changelog

All notable changes to the Market Structure Volume Profile (au-MSVP) indicator will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.3] - 2025-01-11

### Added
- Rectangle backward-shift feature for improved visual alignment
  - Historical rectangles now start at the previous profile's start position
  - Current rectangle starts at the last completed historical profile's start
  - Edge case handling: first profile and empty history fallback to original start positions

### Changed
- Modified historical rectangle positioning logic (line 962)
- Modified current rectangle positioning logic (line 1022)
- Improved rectangle-to-profile visual correlation

### Technical Details
- Implemented using Pine Script single-line ternary operators for syntax compliance
- Historical: `profIdx > 0 ? array.get(profileStartBars, profIdx - 1) : histStartBar`
- Current: `array.size(profileStartBars) > 0 ? array.get(profileStartBars, array.size(profileStartBars) - 1) : startBar`
- Zero impact on volume profile rendering (isolated to box.new() calls only)

### Notes
- This version represents incremental improvement toward exact rectangle alignment
- Further refinement planned based on user testing and feedback

## [1.0.2] - 2025-01-11

### Added
- Extended peak rectangles to historical volume profiles
- Peak rectangles now render across all past profiles (up to 50)
- Automatic capture of peak data on anchor changes
- FIFO memory management for efficient historical tracking

### Changed
- Improved peak rectangle memory management
- Optimized historical profile storage

## [1.0.1] - 2025-01-11

### Added
- Peak rectangle highlighting feature
- Rectangles automatically identify high-volume zones (≥50% of max volume)
- Fully customizable colors, border width, and extension distance
- Optimized threshold calculation for improved performance

### Changed
- Enhanced visual feedback for high-volume areas
- Improved peak detection algorithm

## [1.0.0] - 2025-01-10

### Added
- Initial release of Market Structure Volume Profile indicator
- Event-based profile anchoring (Swing/Structure/Delta modes)
- Statistical volume distribution engine with PDF models
- Dynamic buy/sell volume estimation
- Flexible display modes (Up/Down, Total, Delta)
- POC, VAH/VAL, VWAP, and StdDev calculations
- 13 integrated alerts for key level crosses
- Non-repainting structure/delta modes
- Dynamic row sizing based on price range

### Features
- High-resolution intra-bar data analysis
- T4-Skew probability density function for volume allocation
- Wick-based buy/sell pressure estimation
- Customizable visual settings
- Max 100 profiles for optimal performance
- Integration with Austrian Trading Machine Pine libraries

---

## Version References

- **[1.0.3]** - Rectangle backward-shift alignment improvement
- **[1.0.2]** - Extended historical peak rectangles
- **[1.0.1]** - Added peak rectangle highlighting
- **[1.0.0]** - Initial release with core features
