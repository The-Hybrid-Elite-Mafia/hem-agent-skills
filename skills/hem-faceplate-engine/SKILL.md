---
name: hem-faceplate-engine
description: >
  HEM faceplate rendering — Direct3D 11 GPU panels, knobs, meters, GR needles,
  and the free-tools-only rule. Load before building or reviewing any Phase 2
  UI code, asset pipeline, or faceplate layout.
tags: [dsp, audio, hem, ui, gpu, direct3d, faceplate, phase2, cpp23]
authors: [HybridElite]
---

# HEM Faceplate Engine

> Phase: **2** — do not start until Steven says GO after Phase 1 CI is green.
> API: Direct3D 11 (Windows SDK — free, no licence cost)
> No JUCE, no Qt, no paid middleware.

---

## The Free-Tools-Only Rule

**Never pay for software, services, plugins, fonts, icon packs, or middleware
to build HEM.** If it costs money, we either find a free alternative or build it ourselves.

| Category | Approved (free) | Forbidden (costs money) |
|---|---|---|
| GPU API | Direct3D 11 (Windows SDK) | Any paid GPU abstraction |
| Immediate-mode UI | Dear ImGui (MIT) | Any paid UI framework |
| Vector art | Inkscape (GPL) | Adobe Illustrator, Affinity |
| Raster art | GIMP (GPL), Krita (GPL) | Adobe Photoshop |
| 3D / knob renders | Blender (GPL) | Cinema 4D, Modo, KeyShot |
| Fonts | Google Fonts (OFL/Apache), OFL fonts | Any font requiring a licence fee |
| Image codecs | stb_image (public domain), libpng (free), WIC (Windows SDK) | Any paid codec |
| Audio analysis | Our own FFT (or KissFFT — BSD) | iZotope MAAT, paid SIMD libs |
| CI / hosting | GitHub Actions free tier | CircleCI paid, self-hosted runners you pay for |
| Code signing | Windows SDK signtool (free toolchain) | Paid EV cert from a reseller (get directly from CA) |

**If you are unsure whether something is free:** check the licence file before
adding it. MIT, BSD-2, BSD-3, Apache 2.0, GPL (if we ship source or LGPL for
dynamic linking), and public domain are all fine. Any licence that charges per
seat, per project, per unit sold, or per year is not allowed.

---

## Rendering Stack

```
Windows HWND
  └─ Direct3D 11 swap chain (DXGI)
       ├─ Dear ImGui (immediate-mode widgets — MIT, free forever)
       │    draws knobs, sliders, meters, VU needles, toggles
       └─ Custom D3D11 draw calls for large faceplate backgrounds / bitmaps
            (optional — use ImGui draw list for most things first)
```

### Why Dear ImGui
- MIT licence — zero cost, no attribution required in binaries
- Single .cpp + .h drop-in, no external build system
- Direct3D 11 backend ships with ImGui: `imgui_impl_dx11.cpp`
- Retained-mode widgets are custom-drawn via `ImDrawList` — full control
- No dynamic allocation on the render thread once the frame budget is set

### Direct3D 11 Setup (no licence, ships with Windows SDK)
```cpp
// Free — part of Windows SDK, already on every Windows machine:
#include <d3d11.h>
#include <dxgi.h>
#pragma comment(lib, "d3d11.lib")
#pragma comment(lib, "dxgi.lib")
```

---

## FaceplateDesc → Renderer Mapping

The DSP layer (`processor.hpp`) already describes every panel's layout:

```cpp
struct FaceplateControl {
    ControlKind kind;   // Knob, Toggle, Meter, Label
    uint32_t    paramId;
    float       x, y;  // normalised 0..1 within the faceplate rect
    float       size;  // normalised radius / height
};

struct FaceplateDesc {
    uint32_t          accentRGBA; // always 0xC0FF3EFF for HEM
    std::span<const FaceplateControl> controls;
};
```

The renderer reads `FaceplateDesc` and turns each `FaceplateControl` into an
ImGui draw call. No hardcoded pixel positions in the renderer — all layout
lives in the processor's static `kFace[]` array.

---

## Knob Rendering

Draw knobs with `ImDrawList::AddArc` + a straight line for the pointer.
No bitmap needed — procedural geometry is free to generate and scales to any DPI.

```cpp
// Free, no assets required:
void drawKnob(ImDrawList* dl, ImVec2 centre, float radius,
              float normValue,          // 0..1
              ImU32 trackCol,           // dark ring
              ImU32 accentCol) {        // 0xC0FF3EFF -> IM_COL32(0xC0,0xFF,0x3E,0xFF)
    const float kMinAngle = IM_PI * 0.75f;
    const float kMaxAngle = IM_PI * 2.25f;
    float angle = kMinAngle + normValue * (kMaxAngle - kMinAngle);
    // Track arc (background)
    dl->PathArcTo(centre, radius, kMinAngle, kMaxAngle, 32);
    dl->PathStroke(trackCol, false, 3.0f);
    // Value arc (accent colour)
    dl->PathArcTo(centre, radius, kMinAngle, angle, 32);
    dl->PathStroke(accentCol, false, 3.0f);
    // Pointer line
    dl->AddLine(centre,
        {centre.x + std::cos(angle) * radius * 0.65f,
         centre.y + std::sin(angle) * radius * 0.65f},
        IM_COL32_WHITE, 2.0f);
}
```

---

## Meter (Level Bar) Rendering

```cpp
// Vertical bar meter — no bitmaps, pure ImDrawList:
void drawMeter(ImDrawList* dl, ImVec2 topLeft, ImVec2 size,
               float normLevel,   // 0..1 (0 = silence, 1 = 0 dBFS)
               ImU32 accentCol) {
    // Background
    dl->AddRectFilled(topLeft, {topLeft.x + size.x, topLeft.y + size.y},
                      IM_COL32(30, 30, 30, 255));
    // Level fill (bottom up)
    float fillH = size.y * normLevel;
    dl->AddRectFilled(
        {topLeft.x, topLeft.y + size.y - fillH},
        {topLeft.x + size.x, topLeft.y + size.y},
        normLevel > 0.9f ? IM_COL32(255, 60, 60, 255) : accentCol);
}
```

---

## GR Needle (Gain Reduction)

Used on the bus compressor and Bus Plus panels.
Sweep: 0 dB GR = 12 o'clock; maximum GR = full left.

```cpp
void drawGrNeedle(ImDrawList* dl, ImVec2 pivot, float armLen,
                  float grDb,             // 0 = no GR, negative = GR active
                  float maxGrDb) {        // e.g. -20 dB
    float norm   = std::clamp(grDb / maxGrDb, 0.0f, 1.0f); // 0..1
    float angle  = (-IM_PI * 0.5f) - norm * (IM_PI * 0.75f); // ASSUMED sweep
    dl->AddLine(pivot,
        {pivot.x + std::cos(angle) * armLen,
         pivot.y + std::sin(angle) * armLen},
        IM_COL32_WHITE, 2.0f);
}
```

**ASSUMED:** Needle sweep of 135° (¾π). Adjust in `config.hpp` once the exact
hardware sweep angle is confirmed.

---

## Fonts

Load fonts from the Windows SDK or bundle a free OFL font.
**Do not bundle any font that requires a licence fee.**

Recommended free fonts for HEM panels:
- **Inter** (OFL) — clean, modern, great for labels and values
- **Roboto Mono** (Apache 2.0) — for numeric readouts
- Both available at fonts.google.com — free to embed in commercial software

```cpp
// Load from disk in D3D11 init (not on audio thread):
ImGuiIO& io = ImGui::GetIO();
io.Fonts->AddFontFromFileTTF("fonts/Inter-Regular.ttf", 13.0f);
io.Fonts->Build();
```

---

## HEM Colour Tokens (for ImGui)

```cpp
// Convert HEM lime 0xC0FF3EFF (RGBA) to ImGui ABGR:
static constexpr ImU32 kHemLime    = IM_COL32(0xC0, 0xFF, 0x3E, 0xFF);
static constexpr ImU32 kHemDark    = IM_COL32(0x12, 0x12, 0x12, 0xFF); // panel bg
static constexpr ImU32 kHemMid     = IM_COL32(0x28, 0x28, 0x28, 0xFF); // control bg
static constexpr ImU32 kHemText    = IM_COL32(0xE0, 0xE0, 0xE0, 0xFF); // label text
static constexpr ImU32 kHemClip    = IM_COL32(0xFF, 0x3C, 0x3C, 0xFF); // clip/over
```

Never substitute a different green or neon for `kHemLime` unless Steven says so.

---

## What NOT to Build in Phase 2

- No file I/O dialogs (WAV load/export — Phase 3)
- No transport controls / playback (Phase 3)
- No installer (Phase 3)
- No front-end tool panels (Phase 4)
- No paid font or icon pack

---

## ASSUMED / UNKNOWN Summary

```cpp
// hem_faceplate_engine
// ASSUMED: GR needle sweep is 135° (¾π radians)
// ASSUMED: knob sweep is 270° (1.5π radians) min at ~7 o'clock, max at ~5 o'clock
// UNKNOWN: exact pixel size / DPI scaling target for 1080p vs 4K panels
// UNKNOWN: whether faceplate backgrounds are flat colour, gradient, or bitmap texture
```
