---
name: hem-phase-gates
description: >
  HEM Digital Console build phases — what to build NOW (Phase 1), what to wait
  for Steven's GO signal before starting (Phases 2–4), and what is permanently
  out of scope. Load before starting any new feature or module.
tags: [hem, project, scope, phases, gates]
authors: [HybridElite]
---

# HEM Phase Gates

These are hard stops. Do not start Phase 2 work during Phase 1.
Do not start Phase 3 work before Phase 2 GO. Wait for Steven's "GO" at each gate.

---

## Phase 1 — CURRENT (Active now)

**What to build:**
- All DSP engine modules (pure C++23, no JUCE, no Qt, no third-party libs)
  - `hem_desk_engine` — 13-stem summing desk + bus compressor
  - `hem_4band_eq_engine` — stepped 4-band mastering EQ
  - `hem_bus_plus_engine` — 2-channel mastering compressor + expansion + colour + D-EQ
- Automated unit tests for every engine (zero-dependency, single binary)
- GitHub Actions Windows runner CI — build and test pass on every push
- Architecture constant `kNumStems = 13` is locked; no runtime stem count
- Every `process()` is `noexcept`; zero allocations on the audio thread
- All unverified defaults in one central `config.hpp`, marked `// ASSUMED` or `// UNKNOWN`

**What to deliver at Phase 1 end:**
- All engines compile on Windows x64 with MSVC (C++23, `/arch:AVX2`)
- All unit tests pass in GitHub Actions
- No flaky tests; no disabled tests
- Steven gives GO for Phase 2

**What NOT to build in Phase 1:**
- No GPU faceplate UI
- No standalone app shell
- No file I/O (WAV load/export)
- No front-end tool engines (Phase 4 items)
- No mic preamp, speaker jack, 48V, Hi-Z, or talkback hardware routing
- No installer

---

## Phase 2 — WAIT for GO (GPU Faceplate UI)

**Gate:** Steven says "GO" after Phase 1 CI is green and all engines are tested.

**What Phase 2 covers:**
- Direct3D 11 GPU faceplate rendering
- Per-engine faceplate panels (knobs, meters, GR needle)
- HEM lime accent `0xC0FF3EFF` applied consistently
- Real-time meter display (TripleBuffer → GPU upload)
- UI thread → audio thread parameter wiring via `AtomicParam`

**Do not start this until GO.**

---

## Phase 3 — WAIT for GO (Standalone App)

**Gate:** Phase 2 GPU UI is complete and reviewed by Steven.

**What Phase 3 covers:**
- Windows standalone app shell
- Load 13 stems (WAV file import)
- Playback engine (WASAPI output)
- WAV export (offline render)
- Windows installer (NSIS or WiX)

**Do not start this until GO.**

---

## Phase 4 — WAIT for GO (Front-End Tool Engines)

**Gate:** Phase 3 standalone app ships.

**What Phase 4 covers:**
- 2-band resonance fixer engine
- Mud / width shuffler engine
- Tap-tempo delay engine
- These occupy the physical front-end tool slots on the console (the slots
  currently reserved but left empty in Phases 1–3)

**Do not start this until GO.**

---

## Permanently Out of Scope (All Phases)

These are never built, regardless of phase:
- Mic preamp routing
- Speaker jack / monitor switching
- 48V phantom power
- Hi-Z input switching
- Talkback hardware
- VST / AAX / AU plugin wrapper
- Linux / macOS builds (Windows x64 only)
- Any JUCE, Qt, or third-party audio framework code
- Any code that names or models SSL, API, Waves, or Soundtoys products

---

## Platform & Compiler Constraints (All Phases)

| Constraint | Value |
|---|---|
| Target OS | Windows x64 only |
| Compiler | MSVC (C++23) |
| SIMD | AVX2 + FMA required; AVX-512 deliberately absent |
| Audio API | WASAPI (Phase 1–3), ASIO optional |
| GPU API | Direct3D 11 (Phase 2+) |
| Third-party libs | None |
| JUCE / Qt | Forbidden |

---

## Quick Reference: Is This Phase 1?

Ask these questions before starting any new module:

1. **Is it a DSP engine or unit test?** → Phase 1 ✓
2. **Does it draw anything on screen?** → Phase 2, wait for GO
3. **Does it load or save files?** → Phase 3, wait for GO
4. **Is it a front-end tool (resonance, mud, delay)?** → Phase 4, wait for GO
5. **Is it mic/speaker/48V/talkback hardware?** → Never build it
