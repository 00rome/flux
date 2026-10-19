# Flux — input buffer for Geometry Dash (Geode)

**Mod ID** `flux.input-buffer` · **Developer** Rome · **GD** 2.2081 (Windows) · **Geode SDK** 5.11.2

> **Build status: NOT yet built or tested in-game.** This source was written against the real
> 2.2081 bindings and SDK v5.11.2 headers (every hook and API name was checked), and the
> buffering logic has passing unit tests (`tests/test_buffer.cpp`). It has **not** been compiled
> into a `.geode` yet. Follow "Build" below to produce `Flux.geode`.

## What it does

* **Input buffer (jump/click button).** Presses are forwarded to the game instantly (no added
  latency). A *release* that comes sooner than N frames (1 frame = 1/60 s of gameplay time) after its
  press is held back until the click has lasted N frames. This keeps very short taps from slipping past
  orbs/pads and the like. Long holds are untouched. If you press again while a release is held back, the
  release is sent first, so the game always sees release → press in order.
* **Not applied** to ship, wave, swing and robot (there the hold length *is* the movement), and
  not to platformer left/right buttons.
* **0–20 frames**, default **5**. **Enable toggle.** Both **persist** across restarts.
* **Y** opens/closes the menu (pauses the level when opened mid-run). The key is never consumed, and it
  is ignored while a text field is focused or when Ctrl/Alt/Shift is held.
* Menu: cyan glass panel, FLUX branding, −/+ stepper, 0–20 slider, enable toggle,
  "Every frame counts." tagline, "Press Y to close" hint.

### Interpretation note
"Queue an input for N frames so the game can process it when appropriate" is ambiguous. A literal
delay of every input would only add lag. This mod implements a *minimum click duration* buffer
(above) using only `GJBaseGameLayer::handleButton` and `update`, which are the verified hook
points. It does not read engine internals it cannot verify. If you'd prefer another interpretation
(e.g. replaying an early press until an orb consumes it), that needs in-game testing to tune.

## Build

### Option A: GitHub Actions (recommended, clean Windows machine)
1. Create an empty GitHub repo and push this folder to it.
2. Open the **Actions** tab → **Build Flux** → *Run workflow* (it also runs on every push).
3. When it finishes, download the artifact **geode-build-win** → it contains `flux.input-buffer.geode`.
4. Put the `.geode` file into `%LOCALAPPDATA%\GeometryDash\geode\mods` (or install it from Geode's
   mod menu) and restart the game.

### Option B: local Windows build
Install the Geode CLI and SDK, then run `.\build-local.ps1` from this folder in PowerShell.

## Why your earlier build failed (diagnosis)

Errors like `SequenceTriggerState is undefined` and `EffectManagerState is undefined` come from the
**generated bindings headers**, not from your mod code. Findings from the actual sources:

* SDK `v5.11.2` fetches `geode-sdk/bindings` with `GIT_TAG "main"` and `NO_CACHE` (see the SDK's
  root `CMakeLists.txt`), so every fresh configure downloads whatever `main` is *today*.
* The types **are** defined in the 2.2081 bindings (`SequenceTriggerState` in `Extras.bro`,
  `EffectManagerState` in `GeometryDash.bro`).
* The bindings **code generator changed after the 5.11.2 tag** (commits on 2026-10-16 to
  2026-10-18 modify `codegen/src/PredeclareGen.cpp` and `BindingGen.cpp`: the forward
  declaration / dependency generation). That is exactly the part that emits `depends(...)` ordering.
* That makes an SDK-vs-bindings version mismatch the most likely root cause. Disabling unity builds or
  precompiled headers cannot fix it, which matches what you saw.

**Fix applied:** bindings are pinned to `1605f500d053f8de122bf5c7dc6d17c24866e02e` (the last
bindings commit before the SDK tag) both in the CI workflow (`bindings-ref`) and in
`build-local.ps1` (`GEODE_BINDINGS_REPO_PATH`). This diagnosis is evidence-based but
**unconfirmed by an actual compile**; if the pinned build still fails, send me the first error line.

## Layout
```
mod.json  CMakeLists.txt  build-local.ps1  .github/workflows/build.yml
src/BufferLogic.hpp   pure scheduling logic (unit-tested)
src/Settings.hpp      persistence (Geode saved values)
src/Menu.hpp/.cpp     glass settings popup
src/main.cpp          hooks (handleButton, update, resetLevel) + Y hotkey
tests/test_buffer.cpp g++ -std=c++20 -Isrc tests/test_buffer.cpp && ./a.out
```
