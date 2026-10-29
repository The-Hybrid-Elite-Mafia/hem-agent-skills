---
name: hem-hybrid-core-dsp
description: >
  HEM Hybrid Core 2 Elite — full architecture reference: audio-thread law,
  C++20/C++23 feature map, lock-free primitives, IProcessor contract, SIMD
  kernels, and DSP building blocks. Load this before working on any engine
  in the HEM Studio Suite.
tags: [dsp, audio, cpp, cpp20, cpp23, hem, real-time, no-juce]
authors: [HybridElite]
---

# HEM Hybrid Core 2 Elite — Architecture Skill

> Repo: `The-Hybrid-Elite-Mafia/HEM-Hybrid-Core-2-Elite`
> Standard: **C++23** (`CMAKE_CXX_STANDARD 23`, MSVC `/std:c++latest`)
> Target: Windows x64, AVX2 + FMA, Visual Studio 2022 / MinGW-w64
> No JUCE. No Qt. No third-party libraries.

---

## 1. What This Codebase Is

A 13-stem repair, mixing, and mastering studio written from scratch in C++23.
Only the C++ standard library plus Windows (WASAPI audio, Direct3D 11 graphics).

All DSP engines live in `include/hc/` as header-only or header + one `.cpp`.
Single CMake `INTERFACE` library (`hybrid_core`) that every executable links.

---

## 2. The Absolute Law — Audio Thread

| Rule | Reason |
|---|---|
| Zero allocations on the audio thread | `malloc`/`new` can take a mutex — detected by overriding `operator new` in tests |
| Zero OS I/O | File reads, sockets, `printf` can block |
| Zero mutexes | Priority inversion; audio thread runs at MMCSS Pro Audio |
| Zero unbounded spins without `_mm_pause()` | Burns CPU, defeats SMT on Alder Lake+ |
| All audio-path buffers pre-allocated in `prepare()` | Called once on the UI thread before playback |
| All parameter changes via `AtomicParam::set()` | One-pole smoothed on audio thread; no zipper noise |

`prepare()` = non-RT. `process()` = RT. Never call RT code from non-RT or vice versa.

---

## 3. Central Config — `include/hc/core/config.hpp`

All compile-time constants and every `// ASSUMED` comment live here only.

```cpp
namespace hc {
  inline constexpr std::size_t kNumStems     = 13;    // hard architectural constant
  inline constexpr std::size_t kNumChannels  = 2;     // planar stereo
  inline constexpr std::size_t kMaxBlock     = 4096;  // max frames per callback
  inline constexpr std::size_t kSimdAlign    = 64;    // cache line = AVX-512 register
  inline constexpr std::size_t kCacheLine    = 64;
  inline constexpr std::size_t kMaxChainSlots = 16;   // max processors per chain
}
```

---

## 4. C++20 Features — Actively Used

### `std::span` — non-owning views for parameter descriptors
```cpp
struct ProcessorInfo {
    std::span<const ParamDesc>        params;
    std::span<const FaceplateControl> controls;
};
```

### `std::numbers::pi` — replaces `M_PI`
```cpp
const double w0 = 2.0 * std::numbers::pi * f0 / fs;
```

### `std::atomic::wait()` / `notify_all()` — OS-free worker parking
```cpp
wake_.wait(w, std::memory_order_acquire);  // futex / WaitOnAddress
wake_.notify_all();
```
Workers spin ~20 µs (`_mm_pause()`) then park. One non-blocking kernel call per callback.

### `std::atomic<float>` — lock-free parameter exchange
```cpp
std::atomic<float> target_{0.0f};
void set(float v) noexcept { target_.store(v, std::memory_order_relaxed); }
```
C++20 guarantees lock-free on x86-64.

---

## 5. C++23 Features — Standard Set, Ready to Adopt

The codebase compiles with C++23 but does not yet use these. They are the natural next step:

### `std::expected<T, E>` — error returns without exceptions
```cpp
#include <expected>
std::expected<wav::Audio, std::string> loadStem(std::string_view path);
```
Use for any IO / non-RT factory. The audio engine never throws (RT law).

### Deducing `this` — const/non-const overload collapse
```cpp
auto& value(this auto& self) noexcept { return self.current_; }
```

### `std::print` / `std::println` — type-safe replaces `printf`
```cpp
#include <print>
std::println("[PASS] {} {:.6g} {}", name, value, unit);
```

### `std::flat_map` — cache-friendly registry lookup
```cpp
std::flat_map<std::string_view, Factory> registry_;
```

### `std::views::zip` — cleaner stereo channel iteration
```cpp
for (auto [l, r] : std::views::zip(leftBuf, rightBuf))
    l = r = 0.5f * (l + r);
```

---

## 6. Lock-Free Primitives — `include/hc/core/lockfree.hpp`

Three and only three ways data crosses a thread boundary:

| Class | Use | Writer | Reader |
|---|---|---|---|
| `SpscRing<T, N>` | Commands, sample taps, retired objects | `push()` / `push_n()` | `pop()` / `pop_n()` |
| `TripleBuffer<T>` | Meter snapshots → GPU UI | `write_buffer()` + `publish()` | `update()` + `read_buffer()` |
| `AtomicParam` | Knob value → audio smoother | `set(float)` | `next()` / `advance(n)` |

`SpscRing` capacity must be a power of two. Full ring drops the item — never block on it.
`AtomicParam::advance(n)` runs the smoother O(1) for n samples — use for block-rate params (EQ coefficients).

---

## 7. IProcessor Contract — `include/hc/engine/processor.hpp`

```cpp
class IProcessor {
    virtual void prepare(double sampleRate, std::uint32_t maxBlock) = 0; // NON-RT
    virtual void reset() noexcept = 0;                                   // RT
    virtual void process(AudioBlock& block) noexcept = 0;                // RT
    virtual std::uint32_t latency() const noexcept { return 0; }
    virtual std::size_t numParams() const noexcept = 0;
    virtual AtomicParam& param(std::size_t i) noexcept = 0;
    virtual float meterValue() const noexcept { return 0.0f; }
};
```

`AudioBlock`: `ch[0]` = left, `ch[1]` = right, `numFrames` ≤ maxBlock. Never store across `process()`.

### New processor checklist
- [ ] `static constexpr ParamDesc kParams[kNumParams]` with stable `id` strings
- [ ] `static constexpr FaceplateControl kFace[]` (normalised 0..1 coords)
- [ ] `static inline const ProcessorInfo kInfo{...}` (`inline` avoids ODR issues)
- [ ] `prepare()` does all allocs; `process()` does none
- [ ] Check `bypassed()` at the top of `process()` — pass audio untouched when true
- [ ] Keep detector running while bypassed so meters work for threshold-setting
- [ ] Register in `makeBuiltinRegistry()` in `builtin_processors.hpp`

---

## 8. SIMD Kernels — `include/hc/dsp/simd.hpp`

AVX2 (8 floats/op) with scalar fallback. All take plain `float*`.

| Function | Operation |
|---|---|
| `clear(dst, n)` | zero fill |
| `copy(dst, src, n)` | memcpy-equivalent |
| `scale(dst, g, n)` | `dst *= g` |
| `mac(dst, src, g, n)` | `dst += src * g` — summing bus kernel |
| `mac_ramp(dst, src, g0, g1, n)` | linear gain ramp — click-free fader moves |
| `peak_abs(x, n)` | max \|x[i]\| |
| `sum_squares(x, n)` | Σx² in double (loudness) |
| `dot(a, b, n)` | FIR dot product |
| `complex_power(p, re, im, n)` | re² + im² |
| `apply_mask(re, im, g, n)` | spectral gain mask |
| `db_to_gain(db)` | 10^(db/20) |
| `gain_to_db(g)` | 20·log₁₀(g) |

`ScopedFlushDenormals` — set FTZ+DAZ at the top of every real-time thread. Subnormal floats cost 100× on x86.

> AVX-512 is deliberately absent. Intel 12th–14th gen consumer CPUs (including the i7-14700HX) do not expose AVX-512. Do not add an AVX-512 path.

---

## 9. Biquad — `include/hc/dsp/biquad.hpp`

Transposed Direct Form II in **double** precision (single is insufficient at low frequencies — 30 Hz shelf at 96 kHz puts poles within ~1e-3 of the unit circle).

```cpp
BiquadCoeffs c = designBiquad(FilterType::Bell, sr, freq, q, gainDb);
Biquad f;
f.setCoeffs(c);
f.process(io, numFrames);  // in-place float I/O, double internal
```

Types: `Bell`, `LowShelf`, `HighShelf`, `LowPass`, `HighPass`, `BandPass`, `Notch`.
Source: Robert Bristow-Johnson Audio EQ Cookbook.

---

## 10. Compressor — `include/hc/dsp/dynamics.hpp`

Feed-forward, stereo-linked, log-domain. Soft knee: Giannoulis / Massberg / Reiss 2012. Branching attack/release on the gain-reduction signal in dB — no pumping artefacts.

```cpp
Compressor::Settings s{-18, 3, 6, 10, 120, 0}; // thresh, ratio, knee, att, rel, makeup
Compressor comp;
comp.prepare(sr);
comp.setSettings(s);
comp.process(block);
float gr = comp.gainReductionDb();
```

---

## 11. LookaheadLimiter — `include/hc/dsp/dynamics.hpp`

True-peak-aware, brick-wall, zero overshoot. Lookahead: 1.5 ms. Oversampler: 4× polyphase Kaiser-sinc (β=8). The limiter's detector and the BS.1770 true-peak meter share the same oversampler — they always agree.

```cpp
LookaheadLimiter lim;
lim.prepare(sr);
lim.setCeilingDb(-1.0f);
lim.setRelease(80.0f);
lim.process(block);
std::uint32_t lat = lim.latency(); // compensate with PDC
```

---

## 12. Build Quick Reference

```bash
# Windows (Visual Studio 2022)
cmake -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release

# Linux / macOS → Windows cross-compile
./build_windows.sh   # → dist/HybridCore-Setup.exe

# Engine tests (any OS — must end: ALL PASSED)
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target hc_tests
./build/hc_tests
```

AVX2 is mandatory (`/arch:AVX2` / `-mavx2 -mfma`). Scalar fallbacks in `simd.hpp` are for CI runners only.
