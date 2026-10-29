---
name: hem-4band-eq-engine
description: >
  HEM 4-Band EQ engine — stepped 4-band mastering equaliser with HF shelf,
  upper mid bell, lower mid bell, and LF shelf. Load before building or
  reviewing the mastering EQ processor.
tags: [dsp, audio, hem, eq, mastering, biquad, cpp23]
authors: [HybridElite]
---

# HEM 4-Band EQ Engine

> Processor id: `hc.eq4band` (ASSUMED — confirm when registered)
> Relationship to `Eq6`: separate processor; the 4-Band EQ is a mastering unit
> with stepped frequencies; `Eq6` is the 6-band per-stem parametric EQ.
> Category: `"EQ"`

---

## What It Is

A 4-band mastering equaliser in the spirit of classic stepped mastering EQ hardware.
It is **not** a clone of any specific hardware unit.

Bands (in order, low to high):

| Band | Type | Frequencies (ASSUMED — stepped) | Gain Range |
|---|---|---|---|
| Band 1 — Low Shelf | `FilterType::LowShelf` | 30, 40, 50, 60, 80, 100, 120 Hz | ±16 dB in 2 dB steps |
| Band 2 — Low Mid Bell | `FilterType::Bell` | 100, 150, 200, 300, 400, 600, 800 Hz | ±16 dB in 2 dB steps |
| Band 3 — High Mid Bell | `FilterType::Bell` | 1k, 1.5k, 2k, 3k, 4k, 6k, 8k Hz | ±16 dB in 2 dB steps |
| Band 4 — High Shelf | `FilterType::HighShelf` | 8k, 10k, 12k, 15k, 16k, 18k, 20k Hz | ±16 dB in 2 dB steps |

**UNKNOWN:** Exact stepped frequency values and Q/bandwidth for each band.
**ASSUMED:** Gain steps of 2 dB. Mark all as `// ASSUMED` / `// UNKNOWN` in `config.hpp`.

---

## Differences from `Eq6`

| | `Eq6` (per-stem parametric) | HEM 4-Band EQ (mastering) |
|---|---|---|
| Bands | 6 (HPF, LowShelf, Bell×2, HiShelf, LPF) | 4 (LowShelf, Bell×2, HiShelf) |
| Frequencies | Continuous drag | Stepped (click-stop) |
| Gain | Continuous drag | Stepped (2 dB increments `// ASSUMED`) |
| Use | Per-stem repair | Mastering bus |
| Bypass | Per-band `active_[]` flag | Same pattern |

---

## Implementation with Existing Primitives

Uses `Biquad` from `include/hc/dsp/biquad.hpp` — double precision, Transposed Direct Form II.
Uses `designBiquad()` with the appropriate `FilterType` per band.

```cpp
class Eq4BandProcessor final : public IProcessor {
public:
    static constexpr std::size_t kBands     = 4;
    // Per band: freq index + gain index = 2 params × 4 bands + output trim = 9 params
    static constexpr std::size_t kNumParams = kBands * 2 + 1; // ASSUMED layout

    void prepare(double sr, std::uint32_t) override {
        sr_ = sr;
        for (auto& p : p_) p.prepare(sr, 10.0); // fast snap for stepped params
        updateCoeffs(true);
        reset();
    }

    void process(AudioBlock& b) noexcept override {
        if (bypassed()) return;
        updateCoeffs(false);
        for (std::uint32_t c = 0; c < b.numChannels; ++c)
            for (std::size_t band = 0; band < kBands; ++band)
                if (active_[band]) f_[c][band].process(b.ch[c], b.numFrames);
        // output trim
        const float g = simd::db_to_gain(p_[kBands * 2].current());
        if (g != 1.0f)
            for (std::uint32_t c = 0; c < b.numChannels; ++c)
                simd::scale(b.ch[c], g, b.numFrames);
    }

private:
    void updateCoeffs(bool force) noexcept {
        // For each band: read freq AtomicParam (maps to stepped Hz table)
        //                read gain AtomicParam (maps to stepped dB table)
        // Recompute Biquad only when value changed (same pattern as Eq6)
    }

    double sr_ = 48000.0;
    Biquad f_[2][kBands];    // [channel][band], double precision
    bool active_[kBands]{};
    float last_freq_[kBands]{-1e9f};
    float last_gain_[kBands]{-1e9f};
    std::array<AtomicParam, kNumParams> p_;
};
```

---

## Stepped Parameter Pattern

Because frequencies and gains are stepped (not continuous), the `AtomicParam`
stores the **index** into a lookup table, not the raw Hz or dB value.

```cpp
// Stepped frequency table for Band 1 low shelf (ASSUMED values):
static constexpr float kLowShelfFreqs[] = {30, 40, 50, 60, 80, 100, 120}; // ASSUMED
static constexpr std::size_t kLowShelfFreqCount = std::size(kLowShelfFreqs);

// In process(): convert index to Hz
float freqHz = kLowShelfFreqs[std::clamp((int)p_[kFreqParam].current(), 0, kLowShelfFreqCount - 1)];
BiquadCoeffs c = designBiquad(FilterType::LowShelf, sr_, freqHz, q, gainDb);
```

Smoothing time for stepped params should be very short (10 ms or `snap()` on
change) — stepped EQs click to position, they don't glide.

---

## Q / Bandwidth

**ASSUMED:** Fixed Q per band type:
- Low shelf / High shelf: `Q = 0.707` (Butterworth shelf)
- Bell bands: `Q = 0.5` to `1.0` — `// UNKNOWN` exact value

Mark Q values as `// ASSUMED` in `config.hpp`:
```cpp
inline constexpr double kEq4BandShelfQ = 0.707;   // ASSUMED
inline constexpr double kEq4BandBellQ  = 0.7;     // ASSUMED — UNKNOWN exact value
```

---

## Faceplate Layout

4 bands across the width, each with a frequency selector and a gain knob.
Output trim knob on the far right.

```cpp
static constexpr FaceplateControl kFace[] = {
    // Band 1 (Low Shelf)
    {ControlKind::Knob, kFreq1, 0.08f, 0.35f, 0.10f},
    {ControlKind::Knob, kGain1, 0.08f, 0.70f, 0.14f},
    // Band 2 (Low Mid)
    {ControlKind::Knob, kFreq2, 0.28f, 0.35f, 0.10f},
    {ControlKind::Knob, kGain2, 0.28f, 0.70f, 0.14f},
    // Band 3 (High Mid)
    {ControlKind::Knob, kFreq3, 0.56f, 0.35f, 0.10f},
    {ControlKind::Knob, kGain3, 0.56f, 0.70f, 0.14f},
    // Band 4 (High Shelf)
    {ControlKind::Knob, kFreq4, 0.76f, 0.35f, 0.10f},
    {ControlKind::Knob, kGain4, 0.76f, 0.70f, 0.14f},
    // Output trim
    {ControlKind::Knob, kOutput, 0.93f, 0.60f, 0.10f},
};
```

---

## ASSUMED / UNKNOWN Summary

```cpp
// hem_4band_eq_engine — all in config.hpp
// ASSUMED: gain steps are 2 dB
// ASSUMED: low shelf frequencies: 30,40,50,60,80,100,120 Hz
// ASSUMED: low mid bell frequencies: 100,150,200,300,400,600,800 Hz
// ASSUMED: high mid bell frequencies: 1k,1.5k,2k,3k,4k,6k,8k Hz
// ASSUMED: high shelf frequencies: 8k,10k,12k,15k,16k,18k,20k Hz
// UNKNOWN: exact Q / bandwidth for bell bands
// UNKNOWN: shelf Q (0.707 assumed, may differ from hardware)
// UNKNOWN: gain range (±16 dB assumed)
```
