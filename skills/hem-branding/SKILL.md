---
name: hem-branding
description: >
  HEM Studio Suite branding, naming, and identity rules. Load before writing
  any identifier, UI string, comment, documentation, or public-facing text in
  the HEM codebase. Violations are not style issues — they are legal exposure.
tags: [hem, branding, naming, identity]
authors: [HybridElite]
---

# HEM Branding & Naming Rules

---

## The Hard Lines

**Never use** the following names, abbreviations, logos, or artwork in code
identifiers, UI strings, comments, documentation, filenames, or any
public-facing output from this codebase:

| Forbidden | Why |
|---|---|
| SSL / Solid State Logic | Hardware brand — not affiliated |
| API (the brand / the product line) | Hardware brand — not affiliated |
| Waves | Software brand — not affiliated |
| Soundtoys | Software brand — not affiliated |
| BiG SiX / Big Six | SSL product name |
| Bus+ / Bus Plus (as a direct reference) | SSL product name |
| Fusion | SSL product name |

Sound is "**in the spirit of**" only. The HEM engines are original designs
inspired by professional console and mastering hardware categories. Never claim
they are the hardware, recreations of it, or "modelled" on a specific product.

---

## HEM Product Names

These are the correct names. Use them exactly:

| Engine / Feature | Correct Name |
|---|---|
| 13-stem summing desk | **HEM Desk** |
| Bus compressor on the desk | **HEM Desk Compressor** |
| Mastering compressor unit | **HEM Bus Plus** |
| 4-band mastering EQ | **HEM 4-Band EQ** |
| STFT artifact repair | **HEM STFT Destroyer** |
| Dynamic resonance suppressor | **HEM Resonance** |
| Transient designer | **HEM Transient** |
| Gate | **HEM Gate** |
| De-esser / hat tamer | **HEM De-esser** |
| Every-hit hi-hat tamer | **HEM Every Hit** |
| Master bus + limiter | **HEM Mastering** |
| Full application | **HEM Hybrid Core 2 Elite** (or **HYBRID CORE 2 ELITE** in UI caps) |

---

## Code Identifier Rules

### Processor `id` strings — stable API, never change once shipped
Format: `hc.<shortname>` — lowercase, no spaces.

```cpp
// Correct
static inline const ProcessorInfo kInfo{"hc.gate", "HEM Gate", "Dynamics", ...};
static inline const ProcessorInfo kInfo{"hc.deess", "HEM De-esser", "Repair", ...};
static inline const ProcessorInfo kInfo{"hc.everyhit", "HEM Every Hit", "Repair", ...};

// Wrong — do not use hardware brand names
static inline const ProcessorInfo kInfo{"ssl.buss", ...};      // FORBIDDEN
static inline const ProcessorInfo kInfo{"waves.ssl", ...};     // FORBIDDEN
```

### C++ namespace
`namespace hc` — all engine code lives here. Sub-namespaces: `hc::simd`, `hc::platform`.

### Header guard / include path
`include/hc/<subsystem>/<Name>.hpp`  
Examples: `include/hc/dsp/biquad.hpp`, `include/hc/engine/processor.hpp`

### File names — PascalCase for class files, snake_case for utility
```
Biquad.hpp       ✓
stem_engine.hpp  ✓
SSLBuss.hpp      ✗ (forbidden brand)
```

---

## UI String Rules

- Use all-caps for screen/panel titles: `HEM STFT DESTROYER`, `HEM MASTERING`
- Use title case for knob labels: `Threshold`, `Attack`, `Release`
- Unit strings are short and lowercase: `"dB"`, `"ms"`, `"Hz"`, `""`
- Never display a hardware brand name on any control, label, or tooltip

---

## Accent Colour

HEM lime: `0xC0FF3EFF` (RGBA)

Use this as the `accentRGBA` in every `FaceplateDesc`. Do not substitute
a different green, yellow, or neon unless explicitly directed.

---

## Comment Rules

- Mark every unverified working assumption with `// ASSUMED` in `config.hpp`
- Mark every gap in the specification with `// UNKNOWN` in `config.hpp`
- Do not scatter `ASSUMED` or `UNKNOWN` notes in DSP headers — they belong in the central config only
- Never claim a DSP module "sounds identical" to hardware or is "more powerful" without measured numbers
- Always state: what is TESTED, what is ASSUMED, what is UNKNOWN

---

## Phase & Scope Rules

- **Phase 1 (current):** DSP engines + unit tests + GitHub Actions Windows CI only
- Do not build mic preamps, speaker jack routing, 48V phantom, Hi-Z switches, or talkback
- Those physical slots are reserved for Phase 4 front-end tools
- Stop after Phase 1. Wait for Steven's "GO" before Phase 2 (GPU faceplate UI)
