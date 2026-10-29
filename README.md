# HEM Agent Skills

Agent skills for the **HEM Digital Console** — a C++23 DSP build for Windows x64.
Load these skills into any Claude Code session before working on HEM engine code,
unit tests, CI, or the standalone app.

---

## Install

```bash
npx skills add The-Hybrid-Elite-Mafia/hem-agent-skills
```

This installs all 7 skills at once. Every skill gets its own folder.

---

## Skills in This Repo

| Skill | Load When |
|---|---|
| `hem-hybrid-core-dsp` | Starting any work on the C++23 engine — architecture, primitives, contracts |
| `hem-audio-thread-law` | Writing or reviewing any function that runs on the audio callback thread |
| `hem-branding` | Writing identifiers, UI strings, docs, or any public-facing text |
| `hem-desk-engine` | Working on the 13-stem summing desk, PDC, fader/pan, or bus compressor |
| `hem-bus-plus-engine` | Working on the mastering compressor / expander / colour / D-EQ processor |
| `hem-4band-eq-engine` | Working on the stepped 4-band mastering EQ processor |
| `hem-phase-gates` | Starting any new feature — confirms you are in the right phase |
| `hem-faceplate-engine` | Working on Phase 2 GPU panel UI — D3D11, Dear ImGui, knobs, meters, needles |

---

## Source

These skills document the architecture of the **HEM Hybrid Core 2 Elite** codebase.
They are copied reference material only — the live source code lives in
`The-Hybrid-Elite-Mafia/HEM-Hybrid-Core-2-Elite`.

All unverified working assumptions are marked `// ASSUMED` in the skill text.
All gaps in the specification are marked `// UNKNOWN`.
No skill claims any module sounds identical to hardware.

---

## HEM Branding Quick Reference

- Accent colour: `0xC0FF3EFF` (lime)
- Processor id format: `hc.<shortname>`
- Forbidden names in any identifier or UI string: SSL, API (brand), Waves, Soundtoys
- Phase 1 is active. Wait for Steven's GO before Phase 2.
