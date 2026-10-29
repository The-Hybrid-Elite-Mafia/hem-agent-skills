---
name: hem-desk-engine
description: >
  HEM Desk engine — 13-stem summing console with per-stem chains, plugin delay
  compensation, fader/pan automation, meters, and a stereo-linked bus compressor.
  Load before building, reviewing, or debugging any part of StemEngine or the
  desk compressor path.
tags: [dsp, audio, hem, summing, compressor, mixing, cpp23]
authors: [HybridElite]
---

# HEM Desk Engine

> Implementation: `include/hc/engine/stem_engine.hpp`
> Bus compressor: `include/hc/dsp/dynamics.hpp` → `Compressor`
> Config: `include/hc/core/config.hpp` — `kNumStems = 13`

---

## Architecture

```
Audio callback (conductor thread)
  │
  ├─ Swap pending chains (non-blocking atomic exchange)
  ├─ Recompute PDC if any chain latency changed
  ├─ WorkerPool::run() ──→ 13 stem jobs (parallel)
  │     each stem job:
  │       1. Read source audio @ playhead
  │       2. Run stem chain (Gate → Destroyer → Resonance → EQ → De-esser → Comp → Transient)
  │       3. PDC delay ring (aligns all stems to longest-chain latency)
  │       4. Fader + pan advance (block-rate AtomicParam smoothing)
  │       5. Stem meters (post-chain, pre-fader)
  │       6. Spectrum tap (mid signal → analysis thread)
  ├─ Join all stem jobs
  ├─ Sum 13 stems → master bus (conductor, single-threaded, deterministic order)
  │     uses simd::mac_ramp() — click-free gain ramping across fader moves
  ├─ Master chain (bus compressor + limiter)
  ├─ Master meters + true-peak
  └─ Copy to device output buffer
```

**Summing is deterministic.** Stem 0 is always first. No atomic operations during the sum — the conductor owns the master bus exclusively during the join phase.

---

## The 13-Stem Rule

`kNumStems = 13` is a **hard architectural constant**. Every per-stem array is
`std::array<..., kNumStems>` — the audio thread never allocates and never branches
on a dynamic stem count. Do not make this runtime-configurable.

Stems 0–12 are user stems. Index `kMaster` (= `kNumStems` = 13) is the master bus chain slot.

---

## Per-Stem Chain Slot Order

The audio path through a stem's chain (in order):

```
Gate → Artifact Destroyer → Resonance → EQ → De-esser → Comp → Transient
```

Plugin delay compensation (PDC) is applied after the chain, before summing.

---

## Plugin Delay Compensation (PDC)

Every stem is delay-padded to match the stem with the highest chain latency.
Maximum PDC: `kMaxPdc = 16384` samples per channel.

PDC is recomputed automatically when any chain is swapped in. Each stem has
a simple ring-buffer delay: `pdc[kNumChannels]`, length `pdcLen`, position `pdcPos`.

When adding a processor with latency, return the correct value from `latency()`.
The engine sums all chain latencies per stem to find each stem's total, then
pads the others.

---

## Publishing a Chain (UI → Audio Thread)

```cpp
// UI thread — prepare the chain fully before publishing
auto ch = std::make_unique<ProcessorChain>();
ch->add(std::make_unique<GateProcessor>());
ch->add(std::make_unique<Eq6>());
// DO NOT call ch->prepare() here — publishChain() calls it

bool ok = engine.publishChain(stemIndex, std::move(ch));
// false = previous publish not picked up yet; retry next frame
```

The old chain is sent to a `SpscRing<ProcessorChain*, 64>` graveyard.
Call `engine.collectGarbage()` periodically from the UI thread to `delete` them.

---

## Fader & Pan

Both are `AtomicParam` on each `Stem`:
- `faderDb` — range −96 to +12 dB, default 0
- `pan` — range −1.0 to +1.0, default 0 (centre)

The audio thread calls `advance(n)` per block (O(1) block-rate smoothing),
computes L/R gain as:
```cpp
const float g = muted ? 0 : db_to_gain(faderDb);
blockGain[0] = g * (pan > 0 ? 1 - pan : 1);
blockGain[1] = g * (pan < 0 ? 1 + pan : 1);
```
Then `simd::mac_ramp()` sums each stem into master with a linear ramp from
`lastGain` to `blockGain` — click-free across fader moves.

---

## Mute / Solo

`std::atomic<bool> mute`, `std::atomic<bool> solo` per stem.
Set from any thread. Audio thread reads with `memory_order_relaxed`.

When any solo is active (`anySolo_ == true`), non-soloed stems multiply to zero gain.

---

## Loop

`setLoop(on, start, end)` packs start + end into one `uint64_t` atomic:
- High 32 bits: start frame
- Low 32 bits: end frame

This ensures the audio thread never sees a half-updated loop range.
The wrap is **sample-exact inside a block** — plugin chains run continuously
across the loop point with no gap or click.

---

## Bus Compressor — `Compressor` from `dynamics.hpp`

The desk bus compressor is a `Compressor` instance on the master chain.

Algorithm: feed-forward, stereo-linked, log-domain gain computer.
Soft knee: Giannoulis / Massberg / Reiss 2012.
Ballistics: branching attack/release on the gain-reduction signal in dB.

```cpp
Compressor::Settings s;
s.thresholdDb = -18.0f;
s.ratio        = 3.0f;
s.kneeDb       = 6.0f;
s.attackMs     = 10.0f;
s.releaseMs    = 120.0f;
s.makeupDb     = 0.0f;
```

**ASSUMED values in config.hpp** — exact desk compressor character (attack/release
curves, ratio stepping, auto-gain) requires tuning against the hardware reference.
Mark any hard-coded defaults as `// ASSUMED` in `config.hpp`.

---

## Meters

Each stem has a `LoudnessMeter` (BS.1770-4 K-weighted, integrated + short-term + momentary)
and a `TripleBuffer<StemMeterSnapshot>` for lock-free UI delivery.

`StemMeterSnapshot` fields:
- `loudness` — `LoudnessReading` (M, S, I, LRA)
- `peakDb[kNumChannels]` — block peak in dB
- `faderDb` — current fader position
- `tpDb[kNumChannels]` — true peak (master only)
- `clipped` — `true` if peak ≥ 1.0

Meters are post-chain, pre-fader — they show what the stem IS, not where it sits in the mix.

---

## Seek & Transport

```cpp
engine.play();
engine.stop();
engine.seek(framePosition);   // atomic; applied at start of next callback
engine.requestMeterReset();   // clears integrated loudness and LRA
```

`seek()` uses `seekTarget_.exchange(frame)`. If `UINT64_MAX` is loaded, no seek
is pending. The audio thread checks this at the top of every `process()` call.

---

## What NOT to Build in This Engine

- No mic preamp routing
- No speaker jack / monitor switching
- No 48V phantom power
- No Hi-Z switching
- No talkback

These physical console slots are reserved for Phase 4 front-end tools.
