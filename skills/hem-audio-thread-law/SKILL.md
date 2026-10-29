---
name: hem-audio-thread-law
description: >
  The HEM real-time audio thread contract — zero allocation, zero locks, zero
  OS I/O. Load this skill before writing or reviewing any code that runs on the
  audio callback thread, worker threads, or any function called from process().
tags: [dsp, audio, real-time, cpp, hem, safety]
authors: [HybridElite]
---

# HEM Audio Thread Law

The audio thread is sacred. Break these rules and the app stutters, glitches,
or crashes — often only under load or on stage, never in the test you ran.

---

## The Seven Rules

### 1. Zero Allocations
Never call `new`, `delete`, `malloc`, `free`, `std::vector::push_back` (when it
grows), `std::string` construction from literals, or any STL that may heap-allocate
on the audio thread or any thread it calls.

**Enforcement:** The test suite overrides `operator new` globally and counts every
allocation. Any allocation during `StemEngine::process()` is a test failure.

```cpp
// WRONG — vector may reallocate
void process(AudioBlock& b) noexcept override {
    std::vector<float> tmp(b.numFrames); // ALLOCATION — BUG
}

// RIGHT — pre-allocate in prepare()
AlignedBuffer<float> scratch_;
void prepare(double sr, std::uint32_t maxBlock) override {
    scratch_.resize(maxBlock); // allocation here is fine
}
void process(AudioBlock& b) noexcept override {
    float* tmp = scratch_.data(); // no allocation
}
```

### 2. Zero Mutexes / Locks
`std::mutex::lock()` can block the audio thread behind a UI or loader thread
indefinitely — causing a dropout. Use the three approved lock-free primitives
from `lockfree.hpp` instead: `SpscRing`, `TripleBuffer`, `AtomicParam`.

```cpp
// WRONG
std::mutex m_;
void process(AudioBlock& b) noexcept override {
    std::lock_guard g(m_); // CAN BLOCK — BUG
}

// RIGHT
AtomicParam threshold_{-18.0f, -60.0f, 0.0f};
void process(AudioBlock& b) noexcept override {
    float thr = threshold_.next(); // lock-free, always returns immediately
}
```

### 3. Zero OS I/O
No `fopen`, `fread`, `fwrite`, `printf`, `fprintf`, `std::cout`, socket calls,
registry reads, or any syscall that can block on the audio thread.

`std::fprintf(stderr, ...)` seems harmless — it is not. It can take an internal
`FILE` lock and block behind another thread's `printf`.

### 4. Zero Exceptions
Never throw from the audio thread. The audio callback is declared `noexcept`.
If an exception escapes it, `std::terminate` is called — the app dies silently.

All `process()` methods must be `noexcept`. All RT-called functions must be `noexcept`.

### 5. Zero Unbounded Spins Without `_mm_pause()`
If you must spin-wait (e.g. in the WorkerPool conductor loop), every iteration
must call `_mm_pause()` (x86) or `std::this_thread::yield()` (portable fallback).
Without it you burn a full core, heat the chip, and starve sibling hyperThreads.

```cpp
for (int i = 0; i < 4000; ++i) {
    _mm_pause();   // required
    if (done()) break;
}
```

### 6. Flush Denormals at Thread Entry
Decaying IIR tails produce subnormal floats. On x86, subnormal operations cost
~100× normal. Set FTZ + DAZ at the start of every real-time thread:

```cpp
#include "hc/dsp/simd.hpp"

// In the audio callback and every worker thread:
simd::ScopedFlushDenormals ftz; // set on construction, restored on destruction
```

The WorkerPool stem jobs already do this (`stemJob` calls `ScopedFlushDenormals`).
Any new real-time thread must also do it.

### 7. Prepare Owns All State — Process Owns Nothing
`prepare(sr, maxBlock)` is the only place allowed to allocate, resize, or
reconfigure. `process()` reads state that `prepare()` set up and `AtomicParam`
values that the UI thread wrote.

```cpp
void prepare(double sr, std::uint32_t maxBlock) override {
    // ALL of these happen here, never in process():
    buf_.resize(maxBlock);
    att_ = std::exp(-1.0 / (sr * attackMs_ * 0.001));
    rel_ = std::exp(-1.0 / (sr * releaseMs_ * 0.001));
}
```

---

## Thread Map

| Thread | Priority | What it does |
|---|---|---|
| WASAPI render / ASIO callback | MMCSS Pro Audio | Calls `StemEngine::process()` |
| DSP workers (×N) | MMCSS Pro Audio | Run individual stem chains |
| Analysis thread | Normal | Reads `SampleTap` rings, computes spectrum FFTs |
| UI / loader thread | Normal | `prepare()`, chain publish, `collectGarbage()` |

Data crosses thread boundaries only through `SpscRing`, `TripleBuffer`, or `AtomicParam`.

---

## Approved Cross-Thread Patterns

### Parameter from UI to audio
```cpp
// UI thread:
processor.param(kThreshold).set(-24.0f);

// Audio thread (inside process()):
float thr = p_[kThreshold].next();         // smoothed, zipper-free
// or per block:
float thr = p_[kThreshold].advance(n);    // O(1), block-rate update
```

### Meter from audio to UI
```cpp
// Audio thread (inside process()):
TripleBuffer<StemMeterSnapshot>& tb = stem.snapshot;
StemMeterSnapshot& w = tb.write_buffer();
w.peakDb[0] = simd::gain_to_db(peak);
tb.publish();   // wait-free atomic exchange

// UI / GPU thread:
if (tb.update()) {
    const StemMeterSnapshot& r = tb.read_buffer(); // always latest complete frame
}
```

### Command from UI to audio
```cpp
// UI thread:
SpscRing<MyCmd, 64> cmdRing_;
cmdRing_.push(MyCmd{...});   // returns false if full — caller retries later

// Audio thread (inside process()):
MyCmd cmd;
while (cmdRing_.pop(cmd)) applyCmd(cmd);
```

### Retired object from audio to UI
```cpp
// Audio thread: swap old chain to graveyard ring
graveyard_.push(oldChain);

// UI thread (call periodically):
engine.collectGarbage(); // deletes chains from graveyard
```

---

## Test: Verifying No Allocations

The test runner in `tests/test_main.cpp` instruments `operator new` globally:

```cpp
static std::atomic<bool> g_countAllocs{false};
static std::atomic<long> g_allocs{0};
void* operator new(std::size_t n) {
    if (g_countAllocs.load()) g_allocs.fetch_add(1);
    ...
}
```

Before any `process()` call in a test, set `g_countAllocs = true`, run the
block, then assert `g_allocs == 0`. Any new engine must pass this check.
