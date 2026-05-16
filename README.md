# WheelSlipTactile — Physics-Based Tyre Feedback for SimHub

A real-time NCalcScript tactile feedback model for **Assetto Corsa EVO** on BST-1 / BST-2 tactile transducer rigs. Converts raw tyre slip telemetry into per-corner intensity and frequency signals that communicate grip state, load transfer, and tyre condition through haptic feedback — not vibration effects, but a model of what the tyre is physically doing.

Designed and prompted by **Dmitriy Podkopaev** ([@ddpodkopaev](https://github.com/ddpodkopaev)).

---

## What it does

Most SimHub wheel-slip effects map slip angle directly to amplitude with a fixed threshold. This model treats grip state as a multi-dimensional signal:

- **Lateral amplitude** follows a bell curve centred at the grip limit — rising as you approach the limit, peaking there, then fading as the tyre transitions into a slide. A separate slide residual signal takes over past the limit, so you always feel what the tyre is doing.
- **Longitudinal amplitude** rises non-linearly toward lockup / wheelspin, weighted separately from the lateral path.
- **Frequency** is derived from slip state, load, speed, tyre compliance (pressure), and temperature — all additive components mapped to the transducer's physical operating range.
- **Harmonics** are pre-computed geometric series above the fundamental, ready for ShakeIt Custom Effect intensity fields.

The two files are kept deliberately separate: `RigConfig_BST.ini` holds hardware frequency parameters (specific to the BST-1/BST-2 transducer pair); `WheelSlipTactile.ini` holds physics and feel parameters (portable across rigs that share the same frequency profile).

An example SimHub profile for Assetto Corsa EVO includes two harmonic layers blended with effects such as RPM and road rumble.

The exposed `Disp_*` layers are demonstrated in two `.simhubdash` overlays: one tracks pedal position against longitudinal tyre slip, the other slip angle. Both include reference lines for estimated peak grip at the current setup, road, F1, and wet compounds.

---

## Requirements

- [SimHub](https://www.simhubdash.com/) with NCalcScripts and ShakeIt enabled
- Assetto Corsa EVO (uses AC EVO shared memory property names)
- A BST-1 / BST-2 transducer rig, or any rig using a compatible frequency profile in `RigConfig_BST.ini`

---

## Installation

1. Copy both files to `C:\Program Files (x86)\SimHub\NCalcScripts\`
2. In SimHub → NCalcScripts, enable both scripts
3. Set up ShakeIt Custom Effects per wheel using the templates below

---

## ShakeIt Custom Effect Setup

Each corner uses one primary effect plus up to three harmonic duplicates. All computation is resolved inside the script — ShakeIt fields are trivial expressions.

**FL example** (substitute FL → FR / RL / RR for other corners):

| Effect | Intensity field | Frequency field |
|---|---|---|
| Primary | `[ES.PrimaryAmp_FL] * 100` | `[ES.Freq_FL]` |
| Harmonic 1 | `[ES.PrimaryAmp_FL] * 65` | `[ES.HarmFreq_FL_1]` |
| Harmonic 2 | `[ES.PrimaryAmp_FL] * 42` | `[ES.HarmFreq_FL_2]` |
| Harmonic 3 | `[ES.PrimaryAmp_FL] * 27` | `[ES.HarmFreq_FL_3]` |

`[ES.XXX]` is shorthand for `[DataCorePlugin.ExternalScript.XXX]`.

FL/FR route to BST-2 (front/pedals). RL/RR route to BST-1 (rear/seat). Harmonic gain ladder (100 / 65 / 42 / 27) follows a natural rolloff — adjust in the Intensity field if your transducers have a different frequency response profile.

---

## Key Physics Features

### Grip limit bell curve with slide residual
LatAmp uses a quadratic bell: `2·NormSA·(1 − NormSA·0.5)`, peaking at NormSA = 1.0. Past the limit, `SlideAmp` ramps up from zero, giving a continuous signal across the full slip range. Effective slide depth scales with tyre compliance and wear — a soft or worn tyre breaks away over a wider range of degrees.

### Pressure-derived compound detection (CompBlend)
No compound telemetry is available in AC EVO. Absolute tyre pressure is used as a proxy:
- ≤ 2.0 bar → GT3 / slick characteristics (SAPeak 2–4°, SRPeak 0.09–0.13)
- ≥ 2.3 bar → road tyre characteristics (SAPeak 4–6°, SRPeak 0.13–0.19)
- Between → linear blend

Within each compound, a separate `TC` (tyre compliance) variable interpolates soft ↔ stiff bounds from actual pressure relative to configurable `P_Soft` / `P_Stiff` thresholds. The two dimensions — compound identity and within-compound stiffness — are independent and additive.

### Lag compensation (OnsetLead)
Transducer chain lag (amp attack time, mechanical inertia, NCalc update rate, ShakeIt smoothing) delays the vibration onset relative to the actual grip event. `OnsetLead` pre-scales NormSA and NormSR upward before they enter every downstream path simultaneously — bell curve, slide onset, longitudinal ramp, and frequency composite. The derivation is:

```
OnsetLead ≈ 1 / (1 − lag_s × dNormSA/dt)
```

For ~100 ms total lag and a typical attack rate of 2.0 /s, the recommended starting value is **1.25**.

### Frequency composition
Five additive components per wheel:
- `FreqMin` — hardware floor (from `RigConfig_BST.ini`)
- `Load × LoadBand` — chassis pitch rises with corner load
- `Pow(SlipComp, 0.8) × SlipBand` — grip state carries frequency across the transducer notch
- `SpeedHz` — gentle sub-linear pitch rise with speed
- `FreqOff` — compliance offset (pressure) + temperature offset

All outputs are clamped to `[FreqMin, FreqMax]` from the rig hardware config.

### Tyre condition modifiers
Dirt, temperature, and wear each apply an independent amplitude modifier to the lateral and longitudinal paths. Dirt and temperature also apply frequency offsets. The combined floor is ~23% of clean-tyre amplitude (configurable). SlideAmp is deliberately not attenuated by condition — chaotic sliding is not quieted by cold or dirty tyres.

---

## Configuration

All tunable parameters live in **Section 0** of `WheelSlipTactile.ini`. Key groups:

| Section | Parameters |
|---|---|
| Tyre physics | `SAPeak_*_GT3/Road`, `SRPeak_*_GT3/Road`, `P_GT3_Max`, `P_Road_Min` |
| Compliance | `P_Soft`, `P_Stiff`, `SWFreqRange` |
| Tyre condition | `TempOptimal`, `TempRange`, `TempAmpFloor`, `DirtAmpLoss`, `WearAmpFloor` |
| Signal mixing | `LatMix_Front`, `LatMix_Rear` |
| Load sensitivity | `LoadAnchor`, `LoadFloor` |
| Slide character | `SlideDepth` |
| Speed | `SpeedGateKmh`, `SpeedFreqRef`, `SpeedFreqRange` |
| Lag compensation | `OnsetLead` |
| Overlay references | `Disp_SA_F1/Road/Wet`, `Disp_SR_F1/Road/Wet` |

Hardware parameters (frequency range, notch, harmonic multiplier) are in `RigConfig_BST.ini`.

---

## Diagnostics

Every intermediate in the FL computation pipeline is exported with a `Diag_` prefix and visible in the SimHub property browser. Filter by `"Diag_"` while in-game. Key ones to watch during calibration:

| Export | What to check |
|---|---|
| `Diag_NormSA_FL` | Should reach ~1.0 at the grip limit in a hard corner |
| `Diag_NormSA_FL_sc` | Same, post-OnsetLead scaling — compare to verify lag compensation |
| `Diag_SAPeak_rad_FL` | Live pressure-derived peak SA — should shift with compound/pressure |
| `Diag_CompBlend_FL` | 0.0 at GT3 pressures, 1.0 at road pressures |
| `Diag_TC_FL` | ~0.3–0.8 depending on actual tyre pressure |
| `Diag_TyreMod_FL` | Should be ~0.8–1.0 in nominal conditions |
| `Diag_Bell_FL` | Peaks near grip limit, falls to zero when sliding |
| `Diag_SlideAmp_FL_v` | Zero within limit, rises when sliding |

---

## Calibration Procedure

1. **Verify telemetry** — drive slowly and check `Diag_Raw_SA_FL` reads non-zero mid-corner (not -99, not 0.0).
2. **Set pressure thresholds** — confirm `Diag_CompBlend_FL` reads near 0 for your car (GT3: ~1.7–2.0 bar) or near 1 (road: ~2.1–2.5 bar).
3. **Calibrate slip peaks** — take a hard corner at the edge of grip. `Diag_NormSA_FL` should approach 1.0. Adjust `SAPeak_Soft_GT3` / `SAPeak_Stiff_GT3` (or Road equivalents) until it does.
4. **Set OnsetLead** — leave at 1.0 initially. If vibration feels delayed relative to actual loss of grip, increase toward 1.25. Watch `Diag_NormSA_FL_sc` vs `Diag_NormSA_FL` to see the shift.
5. **Tune LatMix** — adjust `LatMix_Front` / `LatMix_Rear` to taste: lower = more brake/traction feel, higher = more cornering feel.

---

## Simhub dash components

| Component | Comments |
|---|---|
| Telemetry.simhubdash | Top chart: throttle/brake pedal position (green/red) plotted against slip ratio front/rear left (cyan/magenta). Horizontal lines mark estimated slip ratio at peak grip for various tyre types: orange lines = wet > road > F1 (dislpayed top to bottom), cyan = current compound. Bottom chart uses the same approach for slip angle. |
| Steering-Gs.simhubdash | Steering input (green) plotted against steering output (lateral G, cyan). Both values are scaled to peak at roughly 110% of typical F1 limits. Under optimum control, the traces track closely. Understeer appears as steering input (green) outpacing lateral G's (cyan); oversteer vice-versa, with countersteer (green) skipping into the opposite of the axis. Can serves as a diagnostic overlay for slip angle telemetry validation. |

---

## File Structure

```
NCalcScripts/
└── RigConfig_BST.ini            ← hardware frequency profile (BST-1 / BST-2)
├── WheelSlipTactile.ini         ← physics model, all tunable params
└── Assetto Corsa EVO.siprofile  ← example simhub profile with 2 harmonics
└── Telemetry.simhubdash         ← wheel slip telemetry visualisations
└── Steering-Gs.simhubdash       ← steering input vs output visualisation
```

Sections inside `WheelSlipTactile.ini`:

| Section | Contents |
|---|---|
| 0 | Tunable parameters (tyre physics, condition, mixing, lag, overlay refs) |
| 0a | Derived constants (TC, CompBlend, slip peaks, condition modifiers, frequency offsets) |
| 1 | Raw telemetry aliases (remap wheel order here if needed) |
| 2 | Load normalisation |
| 3 | Slip normalisation |
| 3a | OnsetLead scaling |
| 4 | Lateral amplitude (bell curve) |
| 4a | Slide residual amplitude |
| 5 | Longitudinal amplitude |
| 6 | Primary amplitude (lat/long mix → final export) |
| 7 | FL full diagnostic chain |
| 8 | Frequency derivation + harmonic exports |
| 9 | Display exports (tyre potential, driver input, car output, diagnostics) |

---

## License

This project is released under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSE).

**You are free to:** use, copy, modify, and redistribute this work — including in commercial contexts — provided you give appropriate credit to the original author.

**If this file is included in any official SimHub profile bundle, coach package, content creator release, or commercial product**, attribution is required. A credit line such as *"Tyre tactile model by Dmitriy Podkopaev"* with a link to this repository satisfies the requirement.

---

## Author

**Dmitriy Podkopaev** - background in automotive engineering (NVH and chassis dynamics).

[GitHub](https://github.com/ddpodkopaev)
