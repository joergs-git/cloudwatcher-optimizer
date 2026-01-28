# AAG CloudWatcher Solo - K-Factor Optimizer

An interactive tool for optimizing sky temperature correction parameters for the **Lunatico Astronomia AAG CloudWatcher** cloud detection system. This tool helps astronomers achieve consistent clear-sky detection across all seasons without manual threshold adjustments.

![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)
![License](https://img.shields.io/badge/License-MIT-green.svg)
![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey.svg)

<img width="600" height="504" alt="Bildschirmfoto 2026-01-28 um 10 30 28" src="https://github.com/user-attachments/assets/eae60ab0-8f55-48e2-964c-a33b0c68d463" />

<img width="849" height="529" alt="Bildschirmfoto 2026-01-28 um 10 28 48" src="https://github.com/user-attachments/assets/f5113fc3-9095-4ae8-a271-5ac9b120ab5f" />


## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Installation](#installation)
- [Usage](#usage)
- [Understanding the Physics](#understanding-the-physics)
- [The Correction Model](#the-correction-model)
- [K-Factor Reference](#k-factor-reference)
- [Climate Presets](#climate-presets)
- [Validation & Fine-Tuning](#validation--fine-tuning)
- [FAQ](#faq)
- [References](#references)

---

## The Problem

The AAG CloudWatcher uses an infrared sensor to measure the effective sky temperature. The basic approach to determine cloud cover is:

```
Tsky = Ts - Ta
```

Where:
- **Tsky** = Corrected Sky Temperature (°C)
- **Ts** = Infrared Sky Measured Temperature (°C)  
- **Ta** = Ambient Temperature (°C)

**However, this simple approach has a significant limitation:** The relationship between IR sensor readings and actual sky conditions is not constant throughout the year. As the Lunatico manual states:

> *"This simple approach requires frequent changes to the limits - that is, the resulting cloud detection temperature does not remain the same as the weather changes through the year."*

### What Happens in Practice

| Season | Ambient Temp | Clear Sky Tsky | Problem |
|--------|-------------|----------------|---------|
| Winter | -5°C | -26°C | ✓ Works fine |
| Spring | 10°C | -20°C | Threshold needs adjustment |
| Summer | 25°C | -15°C | May trigger false "cloudy" alerts |

This temperature drift means you either need to:
1. Manually adjust thresholds seasonally (tedious)
2. Accept false alerts during temperature extremes
3. **Use the polynomial correction model (this tool!)**

---

## The Solution

The CloudWatcher includes a sophisticated polynomial correction model with 7 adjustable parameters (K1-K7). When properly calibrated, this model compensates for atmospheric effects and produces **consistent Tsky values regardless of ambient temperature**.

This tool:
1. **Simulates** the correction model across your local temperature range
2. **Optimizes** K-factors using differential evolution algorithms
3. **Visualizes** the improvement in real-time
4. **Validates** that cloud detection thresholds work year-round

### Results Example

| Configuration | Tsky Range (Clear Sky) | Std. Deviation |
|--------------|------------------------|----------------|
| Simple (Ts-Ta) | 12.7°C | 4.0°C |
| Lunatico Default | 12.1°C | 3.8°C |
| **Optimized** | **2.4°C** | **0.6°C** |

**~80% reduction in seasonal drift!**

---

## Installation

### Requirements

- Python 3.8 or higher
- NumPy
- SciPy
- Matplotlib

### Setup

```bash
# Clone the repository
git clone https://github.com/yourusername/cloudwatcher-optimizer.git
cd cloudwatcher-optimizer

# Install dependencies
pip install numpy scipy matplotlib

# Run the tool
python cloudwatcher_optimizer_interactive.py
```

---

## Usage

### Interactive GUI Mode (Default)

```bash
python cloudwatcher_optimizer_interactive.py
```

This opens an interactive window with:

#### Left Panel - Climate Settings
- **Min Temp (°C)**: Coldest expected temperature at your location
- **Max Temp (°C)**: Warmest expected temperature at your location
- **Humidity Factor**: Relative atmospheric humidity (0.7 = desert/dry, 1.0 = normal, 1.3 = coastal/humid)

#### Right Panel - K-Factor Sliders
Adjust your current CloudWatcher K-factor settings (K1-K7) to see their effect in real-time.

#### Buttons
- **OPTIMIZE**: Run the optimization algorithm (~30-60 seconds)
- **Reset**: Return to default K-factor values
- **Save Plot**: Export the analysis as PNG to your Desktop

#### Plots
1. **Clear Sky** - Shows corrected Tsky across temperature range (should be horizontal!)
2. **All Sky Conditions** - Shows separation between clear/cloudy/overcast
3. **Correction Factor** - Shows how Td varies with temperature
4. **Comparison** - Bar chart comparing configurations

### Command Line Mode

```bash
python cloudwatcher_optimizer_interactive.py --cli
```

Follow the prompts to enter your climate profile and current K-factors.

---

## Understanding the Physics

### How the IR Sensor Works

The CloudWatcher's infrared sensor measures thermal radiation in the **8-15 µm wavelength range**. This radiation comes from several sources:

1. **Cloud radiation** - Clouds emit IR at temperatures close to ambient
2. **Atmospheric radiation** - Water vapor and CO₂ emit IR
3. **Ground-reflected radiation** - Thermal radiation reflected by terrain

### Clear Sky vs. Cloudy Sky

| Condition | What the Sensor "Sees" | Typical Ts |
|-----------|----------------------|------------|
| Clear | Cold stratosphere (-40 to -60°C) | -20 to -30°C |
| Thin Clouds (Cirrus) | High ice clouds | -10 to -15°C |
| Cloudy | Mid-level clouds | -5 to -10°C |
| Overcast | Low clouds near ambient | -2 to -5°C below Ta |

### The Water Vapor Problem

As ambient temperature increases, the atmosphere holds more water vapor. This additional water vapor:
- Increases atmospheric IR emission
- Makes clear skies appear "warmer" to the sensor
- Reduces the temperature difference between clear and cloudy conditions

**This is why summer clear skies read warmer than winter clear skies, even though both are cloudless.**

### The Correction Model's Purpose

The K-factor polynomial model compensates for this effect by applying a temperature-dependent correction:

```
Td (correction) increases faster than linearly with temperature
     ↓
Compensates for increased atmospheric IR emission
     ↓
Tsky remains constant for clear sky conditions
```

---

## The Correction Model

### Mathematical Formula

From the Lunatico documentation:

```
Td = (K1/100) × (Ta - K2/10) + (K3/100) × exp(K4/1000 × Ta)^(K5/100) + T67

Tsky = Ts - Td
```

Where:
- **Td** = Correction value (°C)
- **Ta** = Ambient temperature (°C)
- **Ts** = IR sensor reading (°C)
- **Tsky** = Corrected sky temperature (°C)

### T67 Cold Weather Factor

```
If |K2/10 - Ta| < 1:
    T67 = sign(K6) × sign(Ta - K2/10) × |K2/10 - Ta|
Else:
    T67 = (K6/10) × sign(Ta - K2/10) × (log₁₀|K2/10 - Ta| + K7/100)
```

### Model Components

| Component | Formula Part | Purpose |
|-----------|-------------|---------|
| **Linear** | `(K1/100) × (Ta - K2/10)` | Base correction, proportional to temperature |
| **Exponential** | `(K3/100) × exp(...)^(...)` | Non-linear correction for high temperatures |
| **Cold Weather** | `T67` | Additional adjustment for extreme cold |

---

## K-Factor Reference

### Parameter Descriptions

| Parameter | Range | Default | Description |
|-----------|-------|---------|-------------|
| **K1** | 0-100 | 33 | Linear coefficient - primary correction strength |
| **K2** | -100 to 150 | 0 | Temperature offset × 10 (pivot point for T67) |
| **K3** | 0-50 | 0 | Exponential coefficient - high-temp correction |
| **K4** | 50-200 | 100 | Exponential rate × 1000 |
| **K5** | 50-200 | 100 | Exponential power × 100 |
| **K6** | -30 to 30 | 0 | Cold weather factor magnitude |
| **K7** | -30 to 30 | 0 | Cold weather logarithmic adjustment |

### Special Configurations

**Simple Mode (K1=100, others=0):**
```
Tsky = Ts - Ta
```
Basic subtraction, no polynomial correction.

**Default (K1=33, others=0):**
```
Tsky = Ts - 0.33 × Ta
```
Linear correction only.

**Lunatico Recommended (K1=33, K3=8, K4=100, K5=100):**
> *"This combination is more nonlinear for ambient temperatures above 30°C."*

Adds exponential term for hot climates.

---

## Climate Presets

The tool includes several preset climate profiles:

| Preset | Temp Range | Humidity | Typical Use |
|--------|-----------|----------|-------------|
| **North Germany** | -10 to 30°C | 1.0 | Central/Northern Europe |
| **South Germany (Alpine)** | -15 to 28°C | 0.95 | Mountain regions |
| **Mediterranean** | 0 to 35°C | 0.9 | Southern Europe |
| **Scandinavia** | -25 to 25°C | 0.85 | Nordic countries |
| **UK / Ireland** | -5 to 28°C | 1.1 | Atlantic maritime climate |
| **Continental US** | -20 to 35°C | 0.95 | Midwest, Great Plains |
| **Southwest US (Desert)** | -5 to 42°C | 0.7 | Arizona, Nevada, etc. |
| **Australia (Temperate)** | 0 to 40°C | 0.85 | Southeastern Australia |

### Humidity Factor Guide

| Value | Climate Type | Examples |
|-------|-------------|----------|
| 0.7 | Very dry | Desert, high altitude |
| 0.85 | Dry | Continental interior |
| 1.0 | Normal | Temperate zones |
| 1.1 | Humid | Coastal areas |
| 1.3 | Very humid | Tropical, monsoon |

---

## Validation & Fine-Tuning

### After Applying New K-Factors

1. **Enter the optimized values** in your CloudWatcher software (Setup → Device → K factors)

2. **Wait for a clear night** with varying temperatures (ideally spanning at least 10°C)

3. **Observe the Sky Temperature graph** throughout the night:
   - ✅ **Horizontal line** = Correct calibration
   - 📈 **Rising with temperature** = Increase K1 by 5
   - 📉 **Falling with temperature** = Decrease K1 by 5

4. **Check threshold behavior:**
   - Clear sky should consistently read below -13°C
   - Clouds should push readings above -11°C
   - Overcast should read above 0°C

### Recommended Thresholds

These thresholds work well with properly calibrated K-factors:

| Condition | Threshold | Description |
|-----------|-----------|-------------|
| **Clear** | < -13°C | Safe for imaging |
| **Cloudy** | -13 to -11°C | Thin clouds, caution |
| **Overcast** | > -11°C | Close observatory |

### Seasonal Validation

For best results, validate your calibration across multiple seasons:

| Season | Check Point |
|--------|------------|
| **Winter** | Cold clear night (-5 to 0°C) |
| **Spring** | Mild clear night (10-15°C) |
| **Summer** | Warm clear night (20-25°C) |
| **Autumn** | Transitional conditions |

---

## FAQ

### Q: Why not just use the simple Ts - Ta formula?

The simple formula assumes a constant relationship between ambient temperature and clear-sky IR readings. In reality, atmospheric water vapor (which increases with temperature) adds significant IR emission that varies non-linearly with temperature.

### Q: How accurate is the simulation?

The tool uses an empirical model based on typical IR sensor behavior. Your actual sensor may vary slightly. The optimization provides an excellent starting point, but field validation is recommended.

### Q: Can I use this for other cloud sensors?

This tool is specifically designed for the AAG CloudWatcher's polynomial model. Other sensors may use different correction algorithms.

### Q: Why does optimization take so long?

The differential evolution algorithm tests thousands of K-factor combinations to find the global optimum. This typically takes 30-60 seconds on modern hardware.

### Q: What if my optimized values look very different from defaults?

This is normal! The optimal K-factors depend heavily on your local climate. A site in Arizona will have very different optimal values than one in Scotland.

### Q: Should I re-optimize every year?

Generally, no. The K-factors compensate for predictable atmospheric physics. However, if you move your observatory or notice increased drift, re-optimization may help.

---

## References

### Lunatico Documentation

- [AAG CloudWatcher Manual](https://lunaticoastro.com/aagcw/enhelp/)
- [Appendix 6: Sky Temperature Correction Model](https://lunaticoastro.com/aagcw/enhelp/)
- [Appendix 7: Tuning The Sky Temperature Model](https://lunaticoastro.com/aagcw/enhelp/)

### Scientific Background

- Infrared atmospheric window: 8-14 µm
- Stefan-Boltzmann law for thermal radiation
- Atmospheric water vapor absorption spectra

### Related Projects

- [Connecting CloudWatcher to Astroshell Dome](https://indilib.org/)

---

## License

MIT License

---

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

---

## Acknowledgments

- **Lunatico Astronomia** for the excellent CloudWatcher hardware and documentation
- The amateur astronomy community for sharing calibration experiences
- SciPy developers for the differential evolution optimization algorithm

---

*Clear skies! 🌟*
