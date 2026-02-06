# CLAUDE.md - CloudWatcher K-Factor Optimizer

> **Project Context File for AI-Assisted Development**
> This file contains all relevant knowledge, formulas, physics, and project state for the AAG CloudWatcher K-Factor Optimizer project.

---

## Project Overview

**Purpose**: Interactive tool for optimizing sky temperature correction parameters (K1-K7) for the Lunatico Astronomia AAG CloudWatcher cloud detection system.

**Goal**: Achieve consistent clear-sky detection across all seasons without manual threshold adjustments by mathematically optimizing the polynomial correction model.

**Languages**: 
- Python (scipy/matplotlib) - Full optimization with differential evolution
- JavaScript/HTML - Browser-based interactive simulation

---

## File Structure

```
cloudwatcher-optimizer/
├── CLAUDE.md                              # This file - AI context
├── README.md                              # User documentation
├── cloudwatcher_optimizer_interactive.py  # Python GUI tool
├── index.html                             # JavaScript/HTML version
└── (future: cloudwatcher_optimizer.js)    # Extracted JS module
```

---

## Core Physics

### The Fundamental Problem

The AAG CloudWatcher uses an infrared thermopile sensor (8-15 µm wavelength) to measure effective sky temperature. The naive approach:

```
Tsky = Ts - Ta
```

**fails** because atmospheric IR emission varies non-linearly with temperature due to water vapor content.

### Why Simple Subtraction Doesn't Work

| Ambient Temp | Water Vapor | Atmospheric IR | Clear Sky Ts | Simple Tsky |
|--------------|-------------|----------------|--------------|-------------|
| -5°C (winter) | Low | Low | -30°C | -25°C |
| 10°C (spring) | Medium | Medium | -22°C | -32°C |
| 25°C (summer) | High | High | -10°C | -35°C |

The **10°C drift** in Tsky values means fixed thresholds (-13°C for clear) fail seasonally.

### Atmospheric Physics

1. **Stefan-Boltzmann Law**: Thermal radiation ∝ T⁴
2. **Water Vapor**: Concentration doubles approximately every 10°C
3. **IR Window**: 8-14 µm is partially transparent, but water vapor absorbs/emits
4. **Clear Sky**: Sensor "sees" cold stratosphere (-40 to -60°C)
5. **Clouds**: Sensor "sees" cloud base (near ambient temperature)

---

## The Correction Model (Lunatico)

### Main Formula

```
Td = (K1/100) × (Ta - K2/10) + (K3/100) × exp(K4/1000 × Ta)^(K5/100) + T67

Tsky = Ts - Td
```

### T67 Cold Weather Factor

```javascript
if (Math.abs(K2/10 - Ta) < 1) {
    T67 = Math.sign(K6) * Math.sign(Ta - K2/10) * Math.abs(K2/10 - Ta);
} else {
    T67 = (K6/10) * Math.sign(Ta - K2/10) * (Math.log10(Math.abs(K2/10 - Ta)) + K7/100);
}
```

### Component Breakdown

| Component | Formula | Purpose | Dominant Range |
|-----------|---------|---------|----------------|
| **Linear** | `(K1/100) × (Ta - K2/10)` | Base correction | All temperatures |
| **Exponential** | `(K3/100) × exp(K4/1000 × Ta)^(K5/100)` | High-temp boost | Ta > 20°C |
| **T67** | Logarithmic cold factor | Cold-temp adjustment | Ta < K2/10 |

### K-Factor Ranges and Defaults

| Parameter | Valid Range | Default | Lunatico Rec. | Typical Optimized |
|-----------|-------------|---------|---------------|-------------------|
| K1 | 0-100 | 33 | 33 | 55-70 |
| K2 | -100 to 150 | 0 | 0 | 70-100 |
| K3 | 0-50 | 0 | 8 | 0-5 |
| K4 | 50-200 | 100 | 100 | 50-80 |
| K5 | 50-200 | 100 | 100 | 80-120 |
| K6 | -30 to 30 | 0 | 0 | 15-25 |
| K7 | -30 to 30 | 0 | 0 | -5 to 5 |

---

## IR Temperature Model (Empirical)

### Clear Sky IR Reading Estimation

```python
def get_ir_temperature(Ta, sky_condition='clear', humidity_factor=1.0):
    if sky_condition == 'clear':
        base_delta = 25.0 * (2.0 - humidity_factor)
        humidity_effect = 0.15 * humidity_factor
        delta = base_delta + humidity_effect * Ta
        
        if Ta > 15:
            delta += 0.08 * humidity_factor * (Ta - 15) ** 1.5
        
        Ts = Ta - delta
    # ... other conditions
    return Ts
```

### Sky Condition Approximations

| Condition | Formula | Typical Delta |
|-----------|---------|---------------|
| Clear | `Ta - (25 + 0.15×Ta + nonlinear)` | 25-35°C below Ta |
| Thin Clouds | `Ta - 15 - 0.05×Ta` | 15-20°C below Ta |
| Cloudy | `Ta - 8 - 0.03×Ta` | 8-12°C below Ta |
| Overcast | `Ta - 3` | 3°C below Ta |

### Humidity Factor Guide

| Value | Climate | Effect on Delta |
|-------|---------|-----------------|
| 0.7 | Desert/Dry | Larger delta (clearer readings) |
| 1.0 | Normal | Baseline |
| 1.3 | Humid/Coastal | Smaller delta (warmer readings) |

---

## Optimization Algorithm

### Differential Evolution Settings

```python
bounds = [
    (30, 80),     # K1
    (-100, 150),  # K2
    (0, 30),      # K3
    (50, 200),    # K4
    (80, 150),    # K5
    (-25, 25),    # K6
    (-30, 30),    # K7
]

result = differential_evolution(
    objective_function,
    bounds,
    seed=42,
    maxiter=2000,
    tol=1e-10,
    polish=True,
    popsize=15
)
```

### Objective Function

```python
def objective_function(params, Ta_range, humidity_factor, target_tsky=-18.0):
    # Primary: Minimize variance of clear sky Tsky
    variance = np.var(clear_temps)
    
    # Secondary: Keep mean around target (-18°C, safely below -13°C threshold)
    mean_penalty = (mean_clear - target_tsky) ** 2
    
    # Constraint: No clear sky values above -13°C
    threshold_violations = sum(1 for t in clear_temps if t > -13.0)
    
    # Constraint: Clouds must still be detectable (above -11°C)
    separation_penalty = max(0, -11 - cloudy_mean) ** 2
    
    return variance * 10 + mean_penalty * 0.5 + threshold_violations * 100 + separation_penalty * 10
```

### Performance Metrics

- **Objective function calls**: ~29,000
- **Tsky calculations**: ~5.8 million
- **Typical runtime**: 30-60 seconds
- **Target Tsky std deviation**: < 1.0°C
- **Target Tsky range**: < 3.0°C

---

## Standard Thresholds

| Condition | Threshold | Meaning |
|-----------|-----------|---------|
| Clear | < -13°C | Safe for imaging |
| Cloudy | -13°C to -11°C | Thin clouds, caution |
| Overcast | > -11°C | Close observatory |

These thresholds are designed to work with properly calibrated K-factors.

---

## Climate Presets

```python
CLIMATE_PRESETS = {
    'north_germany':    ClimateProfile("North Germany", -10, 30, 10, 1.0),
    'south_germany':    ClimateProfile("South Germany (Alpine)", -15, 28, 8, 0.95),
    'mediterranean':    ClimateProfile("Mediterranean", 0, 35, 18, 0.9),
    'scandinavia':      ClimateProfile("Scandinavia", -25, 25, 5, 0.85),
    'uk_ireland':       ClimateProfile("UK / Ireland", -5, 28, 12, 1.1),
    'continental_us':   ClimateProfile("Continental US (Midwest)", -20, 35, 12, 0.95),
    'southwest_us':     ClimateProfile("Southwest US (Desert)", -5, 42, 20, 0.7),
    'australia':        ClimateProfile("Australia (Temperate)", 0, 40, 18, 0.85),
    'namibia':          ClimateProfile("Namibia (Desert)", 5, 40, 22, 0.65),
}
```

---

## Known Limitations

### What the K-Factor Model CAN Compensate

✅ Systematic temperature-dependent drift (water vapor)
✅ Seasonal variations in clear-sky readings
✅ Day/night temperature swings
✅ Altitude effects (via humidity factor)

### What the K-Factor Model CANNOT Compensate

❌ Wind-induced sensor cooling (stochastic)
❌ Dust/smoke particles in atmosphere
❌ Rapid weather changes
❌ Sensor aging/calibration drift
❌ Local obstructions (trees, buildings)

### Wind Problem (From Forum Discussion)

Wind causes two effects:
1. **Direct cooling**: Air flow cools IR sensor's internal ambient temperature sensor
2. **Atmospheric particles**: Dust/smoke carried by wind (unknowable in real-time)

**Suggested workarounds** (not K-factor related):
- Temporal filtering (moving average when windy)
- Voting logic (N consecutive unsafe readings)
- Wind-conditional delays in automation scripts
- Physical shielding of sensor

---

## Code Architecture

### Python Version

```
CloudWatcherOptimizer (class)
├── __init__()
├── run_interactive()      # Launch matplotlib GUI
├── update_climate()       # Slider callback
├── update_kfactors()      # Slider callback
├── update_plots()         # Refresh all 4 plots
├── run_optimization()     # Trigger differential evolution
├── reset_to_defaults()    # Reset sliders
└── save_plot()            # Export PNG

Helper Functions:
├── calculate_T67()        # Cold weather factor
├── calculate_Td()         # Total correction
├── get_ir_temperature()   # IR sensor model
├── calculate_Tsky()       # Final corrected temp
├── objective_function()   # Optimization target
└── optimize_k_factors()   # Run scipy DE
```

### JavaScript Version

```
Functions:
├── calculateT67(Ta, K2, K6, K7)
├── calculateTd(Ta, kFactors)
├── getIRTemperature(Ta, condition, humidity)
├── calculateTsky(Ta, kFactors, condition, humidity)
├── runOptimization()      # Simplified grid search or similar
├── updatePlots()          # Chart.js or similar
└── exportResults()

UI Elements:
├── Climate sliders (min/max temp, humidity)
├── K-factor sliders (K1-K7)
├── Optimize button
├── Results display box
└── 4 canvas plots
```

---

## Future Development Ideas

### High Priority
- [ ] Import/export K-factor configurations (JSON)
- [ ] Historical data analysis from CloudWatcher logs (.csv)
- [ ] Comparison mode: overlay multiple configurations
- [ ] Print-friendly report generation

### Medium Priority
- [ ] Wind filtering simulation module
- [ ] Multi-sensor support (multiple CloudWatchers)
- [ ] ASCOM/INDI integration for live readings
- [ ] Mobile-responsive web version

### Low Priority / Research
- [ ] Machine learning approach (train on real clear-sky data)
- [ ] Automatic climate detection from coordinates
- [ ] Integration with weather APIs for humidity estimation
- [ ] Long-term drift monitoring and alerts

---

## Testing Guidelines

### Unit Tests to Implement

```python
def test_T67_near_pivot():
    """T67 should use linear formula when |K2/10 - Ta| < 1"""
    
def test_T67_far_from_pivot():
    """T67 should use logarithmic formula when |K2/10 - Ta| >= 1"""

def test_Td_default_values():
    """Default K-factors should produce Td ≈ 0.33 × Ta"""

def test_optimization_reduces_variance():
    """Optimized K-factors should have lower Tsky variance than defaults"""

def test_clear_sky_below_threshold():
    """Optimized clear sky Tsky should always be < -13°C"""

def test_cloudy_above_threshold():
    """Cloudy sky Tsky should be > -11°C for proper detection"""
```

### Manual Validation

1. Set optimized K-factors in CloudWatcher
2. Wait for clear night with 10°C+ temperature swing
3. Sky Temperature graph should be horizontal
4. If rising with temp → increase K1
5. If falling with temp → decrease K1

---

## Reference Links

- [Lunatico CloudWatcher Manual](https://lunaticoastro.com/aagcw/enhelp/)
- [Appendix 6: Sky Temperature Correction Model](https://lunaticoastro.com/aagcw/enhelp/)
- [Appendix 7: Tuning The Sky Temperature Model](https://lunaticoastro.com/aagcw/enhelp/)
- [SciPy Differential Evolution](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.differential_evolution.html)

---

## Glossary

| Term | Definition |
|------|------------|
| **Ta** | Ambient temperature (°C) |
| **Ts** | IR sensor raw reading (°C) |
| **Td** | Correction value (°C) |
| **Tsky** | Corrected sky temperature = Ts - Td |
| **T67** | Cold weather adjustment factor |
| **Thermopile** | IR sensor element measuring thermal radiation |
| **Differential Evolution** | Global optimization algorithm |
| **Humidity Factor** | Multiplier for atmospheric moisture (0.7-1.3) |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-01 | Initial Python version with matplotlib GUI |
| 1.1 | 2024-01 | Added Lunatico reference line, improved layout |
| 1.2 | 2024-01 | JavaScript/HTML version for browser use |

---

## Contact / Attribution

- **Original Tool**: Created with Claude AI assistance
- **Physics Basis**: Lunatico Astronomia documentation
- **License**: MIT

---

*Last updated: January 2024*
