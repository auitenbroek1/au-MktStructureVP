# Market Structure Volume Profile (au-MSVP)

![Version](https://img.shields.io/badge/version-1.0.2-blue)
![Pine Script](https://img.shields.io/badge/Pine%20Script-v6-brightgreen)
![License](https://img.shields.io/badge/license-MPL--2.0-orange)

Advanced TradingView indicator that visualizes volume profiles dynamically anchored to market structure events, providing precise analysis of value establishment during critical market phases.

## 📊 Overview

The Market Structure Volume Profile (MSVP) indicator revolutionizes traditional volume profile analysis by anchoring profiles to **market structure events** rather than fixed time intervals. It combines high-resolution intra-bar data with advanced statistical modeling to show exactly where institutional value is being established.

### Key Features

- **Event-Based Profile Anchoring** - Profiles reset on swing changes, structure breaks, or delta shifts
- **Statistical Profile Engine** - Uses PDF (Probability Density Function) models for accurate volume distribution
- **Advanced Buy/Sell Analysis** - Dynamic volume estimator analyzes wicks and trends
- **Flexible Display Modes** - View as Up/Down, Total Volume, or Delta
- **Peak Rectangle Highlighting** - Automatically draws rectangles around high-volume peaks
- **Dynamic Row Sizing** - Automatically adjusts profile resolution based on price range
- **13 Built-in Alerts** - Real-time notifications for key level crosses
- **Non-Repainting** - Structure/Delta modes plot with confirmed data

## 🎯 Profile Anchoring Methods

### 1. Swing Mode
Profiles reset when the impulse baseline (derived from intra-bar delta) changes. The baseline adjusts on confirmed price pivots:
- **Price High** → Baseline moves to lower of previous level or peak delta
- **Price Low** → Baseline moves to higher of previous level or trough delta

### 2. Structure Mode
Profiles reset immediately when market structure breaks are confirmed (e.g., new HH or LL based on pivot sequences).

### 3. Delta Mode
Profiles reset when cumulative delta's market structure breaks (new HH or LL in the delta itself).

## 📈 Statistical Models

### Allocation Model
- **PDF (Probability Density Function)**: Distributes volume proportionally using T4-Skew distribution
- **Classic**: Assigns all volume to close price

### Volume Estimator
- **Dynamic**: Analyzes candle wicks and recent trends to estimate buy/sell pressure
- **Classic**: Classifies volume based on candle color

## 🔧 Installation

1. Open TradingView and navigate to the Pine Editor
2. Create a new indicator
3. Copy the contents of `/src/au-mktStructureVP.pine`
4. Click "Save" and then "Add to Chart"

## ⚙️ Configuration

### Basic Settings
- **Profile Anchor**: Choose between Swing, Structure, or Delta
- **Intra-Bar Timeframe**: Use custom or auto-selected lower timeframe
- **Row Number**: Adjust profile resolution (default: 24)
- **Value Area**: Set percentage for VA calculation (default: 70%)

### Advanced Settings
- **Allot Model**: PDF or Classic volume distribution
- **Price Estimator**: T4-Skew or other statistical models
- **Volume Estimator**: Dynamic or Classic buy/sell split
- **Pivot Settings**: Configure left/right bars and tolerance

### Display Options
- **Volume Display**: Up/Down, Total, or Delta
- **Profile Width**: Adjust visual width (1-100%)
- **Side**: Display on left or right
- **Colors**: Customize buy/sell volume colors
- **Peak Rectangles**: Enable/disable highlighting of high-volume peaks
  - Customizable border color and fill transparency
  - Adjustable border width (1-4 pixels)
  - Configurable extension bars (10-200 bars forward)

## 📊 Indicator Components

The indicator displays several key levels:

- **POC** (Point of Control): Price level with highest volume
- **VAH/VAL** (Value Area High/Low): Bounds of the value area
- **VWAP**: Volume-Weighted Average Price
- **StdDev Bands**: ±1 Standard Deviation from VWAP

## 🚨 Alerts

13 pre-configured alerts for:
- Profile reset events
- Price crossing POC
- Price crossing VA bands
- Price crossing VWAP
- Price crossing StdDev bands

## 📚 Dependencies

This indicator uses the following Pine Script libraries:
- `AustrianTradingMachine/LibTmFr/1` - Timeframe utilities
- `AustrianTradingMachine/LibBrSt/1` - Bar statistics
- `AustrianTradingMachine/LibPvot/1` - Pivot detection
- `AustrianTradingMachine/LibVPrf/1` - Volume profile calculations

## 🔍 Usage Examples

### Trading Applications
1. **Value Area Trading**: Enter when price returns to VA after expansion
2. **POC Reversals**: Watch for reactions at high-volume nodes
3. **Structure Confirmation**: Combine with market structure breaks
4. **Delta Divergence**: Spot when price and delta disagree

### Best Practices
- Use higher timeframes (4H+) for swing trading
- Combine with trend analysis for directional bias
- Monitor VWAP for intraday institutional levels
- Wait for profile confirmation before entering trades

## ⚠️ Important Notes

### Data Behavior
- **Real-time bars update dynamically** as new intra-bar data arrives
- Structure/Delta modes use non-repainting plotted lines
- Swing mode plots retroactively from pivot origin
- All anchor events require confirmation (Pivot Right Bars)

### Performance Considerations
- Higher row counts increase precision but impact performance
- Lower intra-bar timeframes provide more accuracy
- Max 100 profiles displayed to optimize rendering
- Adjust settings based on available data resolution

## 📖 Version History

See [CHANGELOG.md](changelog/CHANGELOG.md) for detailed version history.

**Current Version: 1.0.2** (2025-01-11)
- Extended peak rectangles to historical volume profiles
- Peak rectangles now render across all past profiles (up to 50)
- Automatic capture of peak data on anchor changes
- FIFO memory management for efficient historical tracking

**Version 1.0.1** (2025-01-11)
- Added peak rectangle highlighting feature
- Rectangles automatically identify high-volume zones (≥50% of max volume)
- Fully customizable colors, border width, and extension distance
- Optimized threshold calculation for improved performance

**Version 1.0.0** (2025-01-10)
- Initial release
- Event-based profile anchoring (Swing/Structure/Delta)
- Statistical volume distribution engine
- Dynamic buy/sell volume estimation
- Flexible display modes
- 13 integrated alerts

## 🤝 Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## 📄 License

This Pine Script code is subject to the terms of the Mozilla Public License 2.0.
See [https://mozilla.org/MPL/2.0/](https://mozilla.org/MPL/2.0/)

## ⚖️ Disclaimer

**For Informational/Educational Use Only**

This indicator is provided for informational and educational purposes only. It does not constitute financial, investment, or trading advice. All trading decisions are made at your own risk. Past performance does not guarantee future results.

---

**Author**: © AustrianTradingMachine
**Repository**: https://github.com/auitenbroek1/au-MktStructureVP
**TradingView**: Market Structure Volume Profile (au-MSVP)
