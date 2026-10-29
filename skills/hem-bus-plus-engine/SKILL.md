---
name: hem-bus-plus-engine
description: >
  HEM Bus Plus engine — 2-channel mastering processor: stereo-linked compressor,
  upward expansion, colour/saturation, and a 2-band dynamic EQ. Load before
  building or reviewing the Bus Plus processor chain.
tags: [dsp, audio, hem, mastering, compressor, expansion, eq, cpp23]
authors: [HybridElite]
---

# HEM Bus Plus Engine

> Processor id: `hc.busplus` (ASSUMED — confirm when registered)
> Lives on: master chain in `StemEngine`, or as a standalone mastering insert
> Category: `"Master"`

---

## What It Is

The HEM Bus Plus is a 2-channel (stereo-linked) mastering processor in the
spirit of high-end analogue bus compressor hardware. It is **not** a model
or clone of any specific hardware unit.

Signal path (in order):

```
Input → Compressor → Upward Expander → Colour/Drive → Dynamic EQ (2-band) → Output Trim
```

---

## Section 1 — Stereo-Linked Compressor

Base class: `Compressor` from `include/hc/dsp/dynamics.hpp`.
Algorithm: feed-forward, log-domain, soft knee (Giannoulis / Massberg / Reiss 2012).
Detector: peak of max(|L|, |R|) — stereo linked, not dual-mono.

### Parameters

| Name | Range | Default | Notes |
|---|---|---|---|
| Threshold | −40 to 0 dB | −18 dB | |
| Ratio | 1.5, 2, 4, 10 : 1 | 4 : 1 | `// ASSUMED` — stepped values |
| Attack | 0.1 – 30 ms | 3 ms | `// ASSUMED` |
| Release | 50 – 1200 ms | 300 ms | `// ASSUMED` — or Auto |
| Makeup | 0 – 20 dB | 0 dB | |
| Knee | 0 – 10 dB | 4 dB | `// ASSUMED` |

**UNKNOWN:** Exact auto-release curve. Mark `// UNKNOWN` in `config.hpp` until measured.

---

## Section 2 — Upward Expansion (Drive / Density)

Upward expansion lifts quiet signals toward a floor threshold — the opposite
of a downward gate. This adds perceived density and punch without increasing peaks.

Algorithm: `// ASSUMED` — upward expander with:
- Threshold: the level below which gain is added
- Ratio: expansion ratio (< 1 for upward: each dB below threshold adds gain)
- Attack / Release: same structure as compressor ballistics

```cpp
// ASSUMED implementation shape:
float xDb = gain_to_db(inputPeak);
float below = threshold - xDb;  // positive when signal is below threshold
if (below > 0) gainDb += below * (1.0f - 1.0f / expansionRatio);
```

**UNKNOWN:** Exact ratio values, knee shape, interaction with compressor section.
Mark all defaults as `// ASSUMED` in `config.hpp`.

---

## Section 3 — Colour / Drive

A harmonic saturation stage that adds even-order (primarily 2nd) harmonic
content, controlled by a single Drive knob.

Algorithm: `// ASSUMED` — soft-clip or tube-style waveshaper, e.g.:
```cpp
// Soft clip (placeholder — replace with tuned waveshaper):
float saturate(float x, float drive) noexcept {
    x *= drive;
    return x / (1.0f + std::fabs(x)); // ASSUMED shape
}
```

**UNKNOWN:** Exact transfer curve, frequency-dependent colouring, output compensation.
Mark as `// ASSUMED` / `// UNKNOWN` in `config.hpp` until the character is tuned.

Drive should be `0` by default (unity, no saturation). At max drive the output
must not clip — apply makeup gain automatically or clamp.

---

## Section 4 — 2-Band Dynamic EQ (D-EQ)

A high-shelf and a low-shelf that only act when the band exceeds a threshold.
Controls the high and low end of the stereo bus dynamically.

Uses `Biquad` from `include/hc/dsp/biquad.hpp` for the shelves (double precision).
The detector runs on a copy of the signal — the audio path is the shelf only.

### High Band (D-EQ Hi)
- Type: HighShelf
- Frequency: `// ASSUMED` ~8 kHz
- Threshold: level above which the shelf activates
- Max Cut: maximum dB of reduction
- Direction: cut-only (never boosts dynamically)

### Low Band (D-EQ Lo)
- Type: LowShelf
- Frequency: `// ASSUMED` ~200 Hz
- Same threshold/max-cut structure as hi band

Both bands: no latency, detector uses the same attack/release structure
as `DeEsserProcessor` (0.5 ms attack, 60 ms release `// ASSUMED`).

---

## IProcessor Implementation Notes

```cpp
class BusPlusProcessor final : public IProcessor {
    // Sections — all allocated in prepare(), zero in process()
    Compressor    comp_;
    // expander, colour, deq_hi, deq_lo — to be implemented
    // All AtomicParam for every knob — set from UI, smoothed on audio thread
};
```

Processor id: `"hc.busplus"` — stable, never change after first ship.
Category: `"Master"`.

### `process()` order
1. Check `bypassed()` — if true, pass untouched
2. Flush denormals (`ScopedFlushDenormals ftz;` if not already set by caller)
3. Compressor section
4. Upward expander section
5. Colour / drive section
6. D-EQ hi + lo bands
7. Output trim

---

## Meter Outputs

- Gain reduction dB from compressor → `meterValue()` (feeds the GR needle on faceplate)
- Expander gain dB (optional second meter)
- D-EQ hi and lo activity (optional)

All meters use `std::atomic<float>` with max-hold, taken by the UI thread.

---

## ASSUMED / UNKNOWN Summary

Everything below must live in `config.hpp` until verified:

```cpp
// hem_bus_plus_engine defaults
inline constexpr float kBusPlusDefaultThresholdDb = -18.0f;  // ASSUMED
inline constexpr float kBusPlusDefaultRatio        = 4.0f;   // ASSUMED
inline constexpr float kBusPlusDefaultAttackMs     = 3.0f;   // ASSUMED
inline constexpr float kBusPlusDefaultReleaseMs    = 300.0f; // ASSUMED
// UNKNOWN: auto-release curve
// UNKNOWN: exact colour/drive transfer function
// UNKNOWN: D-EQ crossover frequencies and detector time constants
```
