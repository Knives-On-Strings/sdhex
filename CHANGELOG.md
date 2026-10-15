# SDHex Changelog

All notable changes to SDHex will be documented in this file.

## [0.14.0] - 2026-10-02

**Version numbers reset.** SDHex went to 1.0 early in development, long before
it was a 1.0 product. Releases 1.0.0 through 1.13.1 below were pre-release
builds; numbering continues from 0.14.0 until there is a real 1.0. If you have
1.13.1 installed, this installer replaces it. On macOS, also remove any other
1.x copy of the AU (for example one in `~/Library/Audio/Plug-Ins/Components`):
macOS loads the highest-numbered AU it finds, so a stray 1.x copy would win.

### Added

- **SDS-3, SDS-7 and SDE factory kits**, alongside the SDS-V, SDS-8 and SDS-9
  kits. The SDS-7 and SDE kits fill all twelve pads.

### Changed

- **The kit browser and the voice picker list models by release year**: SDS-3,
  Clap Trap, SDS-V, SDS-7, SDS-8, Digital Clap Trap, SDS-9, SDE. User kits
  follow the factory kits, A–Z.
- **The top bar's save button is a disk icon** instead of the word "SAVE".
- **Loading a kit or drum preset always gives the same result.** A setting the
  file doesn't mention now loads at its default instead of keeping whatever the
  pad had before.

### Fixed

- **Kits and drum presets kept only 13 of each pad's 24 settings.** Filter
  envelope, velocity-to-filter, waveform, sub mix, noise pitch, FM algorithm,
  the burst settings and the digital-oscillator settings were dropped on save
  and ignored on load — including the noise pitch and sub mix in the factory
  SDS-7 drum presets. All 24 are now saved and loaded.
- **Intermittent AU state-restore failure.** Restoring state told the host a
  preset had changed, which made macOS re-read every parameter while the restore
  was still applying them, so some values were lost. Seen in pluginval about one
  run in four; none in 48 runs since the fix.
- **Knob value readouts for skewed parameters** (FM Speed, Filter Env, Digital
  Pitch and others) showed the wrong number. The sound was always right; only
  the displayed value was off.

## [1.13.1] - 2026-09-30

### Changed

- **New instances start at the declared parameter defaults, not the SDS-V kit.**
  The constructor used to load the SDS-V factory kit, so a new instance sat at
  the kit's values rather than the defaults it reports to the host — and hosts
  require a freshly initialised plugin to match its reported defaults. A new
  instance is now twelve identical SDS-V Bass pads until a kit is loaded. Factory
  kits are still installed and load from the kit browser; saved DAW sessions
  restore exactly as before. The top bar starts at "Init" rather than naming a
  kit that was never loaded.

### Fixed

- **clap-validator `param-default-values`** — the last clap-validator failure.
  SDHex now passes with 0 failures (33 passed, 1 warning, 10 skipped; the
  warning, `state-invalid-random`, comes from the JUCE CLAP wrapper, whose
  state load cannot report failure). Regression-tested by checking every one of
  the 290 parameters against its declared default on a fresh instance.

## [1.13.0] - 2026-09-29

Minor: the plugin now exposes a different set of outputs to the host.

### Changed

- **Per-pad outputs are stereo.** Each of the 12 pad outputs is now a stereo
  pair carrying the pad's post-pan signal, alongside the stereo main — 26
  output channels, up from 14. Pad outputs remain pre-master: they are not
  affected by master volume or by the passive bus-loading compensation, both of
  which only apply to the main mix. Pad buses are stereo or disabled; any other
  layout is rejected.

### Fixed

- **Pad audio went to the wrong outputs whenever a host disabled any pad bus.**
  Routing computed each pad's channel as `2 + padIndex`, which is only correct
  while every bus is enabled — JUCE packs the enabled buses' channels into the
  buffer contiguously. Hosts that activate outputs selectively, such as Cubase
  (main output only until the user enables more), shifted every pad above a
  disabled bus onto its neighbour's output, silently: route "Pad 4" to a tom
  track, hear the kick. Destinations are now resolved from each bus's actual
  enabled state once per block, into fixed arrays, so nothing allocates on the
  audio thread. The oversampled and 1x render paths now share one routing
  block rather than two copies that could drift apart.

  Regression-tested by `Tests/Processor/MultiOutputTests.cpp`. Its key case —
  main plus Pad 4 enabled, trigger Pad 1, assert Pad 4 stays silent — was
  checked against the old indexing reinstated, and fails on it.

## [1.12.0] - 2026-09-18

Minor rather than patch: this adds a feature, not a fix.

### Added

- **Telemetry and issue reporting, via the shared KosNet library.** SDHex is the
  first plugin wired to `../kos-net` rather than carrying its own copy —
  XCent, Stabmatic and Virage each had one, and the three were literally the
  same files differing only in macro prefix, namespace and logging header.

  **Off by default, and it stays off until asked.** With the feature compiled
  out, KosNet is a set of no-op stubs, no socket-opening code exists in the
  binary, and every piece of privacy UI is absent from the DOM rather than
  disabled. Enable with `-DSDHEX_NETWORK_ENABLED=ON` plus a form secret; the
  secret is not stored in this repo and CMake warns without one.

  What it sends when enabled: anonymous session start/end, plugin format, host
  name and OS, keyed to a random per-install id. Never audio, never presets,
  never file paths.

  - Processor: identity and log sink registered from the constructor;
    `session_start` fires in `prepareToPlay` below the scanner guard, so a
    pluginval or auval scan is not counted as a session, and is latched so a
    sample-rate change extends one session rather than starting another;
    `session_end` fires only if a session was actually opened.
  - Editor: `getPrivacyState`, `setTelemetryEnabled` and `reportIssue` native
    functions. `opt_out` fires before the preference flips — the last permitted
    use of the consent being withdrawn, and what makes the opt-out rate
    measurable.
  - UI: first-run consent panel that asks once, a permanent Settings → Privacy
    toggle, a Report-a-bug entry point, and a copyable-URL fallback panel for
    when the backend ping fails.

### Fixed

- **`host_version` would have leaked a filesystem path.** The first draft of the
  telemetry wiring used `PluginHostType::getHostPath()` for that field, which
  returns the host executable's path and on macOS routinely contains
  `/Users/<username>/`. The privacy contract forbids sending file paths. Now
  `SystemStats::getOperatingSystemName()`, commented at both call sites since it
  is the easy place to reintroduce.

## [1.11.3] - 2026-09-17

### Fixed

- **Host never told that parameter values changed after a state restore** —
  `setStateInformation()` restored the APVTS state correctly but never called
  `updateHostDisplay()`, so hosts kept serving their cached parameter values.
  After loading a session the host's generic UI, automation lanes and preset
  display all showed the pre-load values. Now calls `updateHostDisplay()` with
  `programChanged` and `parameterInfoChanged` set, which is what JUCE's own
  format wrappers do after loading a program, and what the CLAP wrapper turns
  into `params.rescan(CLAP_PARAM_RESCAN_VALUES)`. Affected VST3 and AU as well
  as CLAP; found by clap-validator's `state-reproducibility-*` tests, which
  pluginval has no equivalent of.

- **Plugin state grew on every save/load cycle** — `getStateInformation()`
  unconditionally appended a `VoicePresets` element, while
  `setStateInformation()` handed the whole tree, that element included, to
  `AudioProcessorValueTreeState::replaceState()`. The element was therefore
  stored inside the APVTS tree, came back out of the next `copyState()`, and
  was written again, so each cycle added another copy and no two consecutive
  saves were byte-identical. `setStateInformation()` now strips it before
  passing the tree to APVTS, and `getStateInformation()` clears any strays
  before adding its own, so states written by earlier builds heal on next save.

- **`One→Three` FM algorithm silently selected `Chain`** — the choice name
  contained a U+2192 arrow. Parameter text crosses the host boundary through
  fixed-size C string buffers and JUCE resolves it back with an exact
  `choices.indexOf(text)`; a miss returns -1, which clamps to index 0. Renamed
  to `One-Three`. All host-visible parameter strings are now ASCII, guarded by
  a test.

- **macOS build was broken** — two faults, neither reachable from the Windows
  toolchain. CMake handed the C++ precompiled header to JUCE's 62 Objective-C++
  sources (`Objective-C was disabled in precompiled file`); those are now
  exempted from the PCH. And `PluginEditor.cpp` requested the WebView2 backend
  and a `windowsLocalAppData` user-data folder unconditionally, which does not
  compile on AppClang; both are now behind `#if JUCE_WINDOWS`, so macOS and
  Linux take JUCE's default WebKit backend.

### Added

- **Multi-platform build and release system**, following the patterns already
  used by XCent and Virage. Paired entry points `scripts/configure.{sh,ps1}`
  and `scripts/build.{sh,ps1}`; per-platform build directories (`build-mac`,
  `build-linux`, `build`) so one checkout serves several machines;
  `SDHEX_MACOS_UNIVERSAL` and `SDHEX_REQUIRE_CLAP` options for shipping builds;
  and `tools/stage_release.py` as a single staging orchestrator for all three
  platforms, which validates with pluginval and clap-validator and fails a
  release on any error. Packaging targets: `SDHex_MacOS_Package` (.pkg in a
  .dmg, with optional signing and notarization), `SDHex_Linux_Package`
  (.tar.gz with install.sh) and `SDHex_Installer` (NSIS).

- **`SDHex_ProcessorTests`**, a second test binary that links the plugin's
  shared code. The existing `SDHex_Tests` is deliberately pure DSP and never
  instantiates the processor, so host-contract behaviour had no coverage at
  all. Covers state round-tripping driven the way a CLAP host drives it, the
  host notification on restore, byte-stable re-saves, and ASCII-only parameter
  text.

### Note

1.11.2's changes are included here. That version was written up in this file
but `project(SDHex VERSION ...)` was never incremented for it, so no build ever
carried the number; 1.11.3 is the first build containing those fixes.

## [1.11.2] - 2026-04-15

### Fixed

- **DigitalClapModel voice stuck active after envelope expiry** — `isActive()` remained
  true long after both LM13700 envelopes expired because the inactivity gate also required
  `!eprom.isPlaying()`. The EPROM player continued running for ~200ms after the envelopes
  hit zero (producing no audible output), wasting CPU and preventing voice stealing.
  Gate now checks only envelope state; `eprom.reset()` is called immediately on deactivation
  to stop the silent playback.

## [1.11.1] - 2026-02-28

### Fixed

- **SDS-9 SUB_MIX on tom voices** — Tom voices (Low/Mid/High Tom) were mixing in
  sub-oscillator output despite the SDS-9 hardware only having sub-oscillator on the
  bass card. Changed `vcoWithSub` → `vcoOutput` in processSample(). This also fixes
  a compilation error from the partially-applied v1.11.0 audit changes.

- **SDS-9 FILTER dead knob travel** — APVTS FILTER range (100–15000 Hz) exceeded the
  CEM3394's effective range (200–12000 Hz), causing dead knob travel at extremes.
  Base cutoff is now clamped to 200–12000 Hz in updateDSPParameters(). FM-modulated
  cutoff was already clamped; this makes the base consistent. Cosmetic fix only —
  audio behavior unchanged (CEM3394 was already clamping internally).

- **SDS-9 VEL_FILTER documented as intentionally unused** — The velocity-to-filter
  parameter (stored in SimmonsDrumModelBase) is used by SDS-V and SDS-8 but not by
  SDS-9, which uses audio-rate Second Skin FM for cutoff modulation instead.
  Added explicit documentation in SDS9Model::updateDSPParameters().

## [1.11.0] - 2026-02-28

### Added

- **Preset filtering by model in VoicePickerModal** — The preset column now only shows
  presets matching the currently selected model family. Previously all presets from all
  models were shown in a flat list. Uses the `engine` field from `getDrumPresetList()` to
  filter via `ENGINE_TO_VOICE_INFO` mapping. "Factory" reset option always visible.

- **Factory defaults moved to JSON** — The hardcoded 23-entry `ModelDefaults` table in
  `PresetManager::applyFactoryDefaults()` has been extracted to `Source/presets/factory-defaults.json`.
  Defaults are parsed from BinaryData once at construction and cached. Editing factory defaults
  no longer requires recompilation — just update the JSON and rebuild BinaryData.

### Changed

- **SDS-V noise HPF: Tom 600 Hz → 800 Hz** — Hardware-accurate "gimmick cap" value.
  Bass and Tom cards both use 100pF capacitors (800 Hz HPF), while Snare uses 470pF (400 Hz).
  Previously Tom was incorrectly set to 600 Hz.

- **SDS-8 CLICK knob color: light blue → royal blue** — Added `royalBlue` (#4488CC) color
  token to `@pfs/ui` theme. SDS-8 CLICK knob now uses this darker blue to visually
  differentiate it from the other parameter knobs on the navy-themed strip.

### Technical

- New color token `royalBlue` in `pfs-ui/themes/tokens.ts`, `Knob.tsx`, and `types.ts`
- `factory-defaults.json` added to `SDHex_PresetData` BinaryData target
- `PresetManager::loadFactoryDefaults()` parses JSON at construction, caches in `std::vector`
- KnobSlot color type updated to include `'royalBlue'` and `'cream'` (was missing)

## [1.10.0] - 2026-02-28

### Added

- **Undo/redo for all parameter changes** — wired JUCE `UndoManager` into APVTS so every knob
  drag, preset load, kit load, factory reset, and model change is undoable.
  - Keyboard shortcuts: `Ctrl+Z` (undo), `Ctrl+Y` / `Ctrl+Shift+Z` (redo)
  - Header bar undo/redo arrow buttons with auto-dimming when stack is empty
  - Preset loads grouped as single undo transactions via `beginNewTransaction()`
  - Undo history is in-memory (cleared on DAW session restore, lost on close)

## [1.9.1] - 2026-02-28

### Fixed

- **"Factory" voice preset was a no-op** — Selecting "Factory" in the Voice Picker now correctly
  resets the voice to model-specific factory defaults. Previously, the UI attempted to load a
  `Factory.json` drum preset file that does not exist, silently failed, and left all parameters
  unchanged — causing each pad to show "Factory" while retaining residual values from previously
  loaded presets or manual tweaks.
  - Root cause: `PresetManager::loadDrumPreset("Factory", ...)` fell through to file I/O path
    with no special handling for the synthetic "Factory" sentinel name
  - Fix: Added `applyFactoryDefaults(padNumber)` which reads the current `MODEL` parameter and
    applies curated, musically appropriate defaults for all 23 model types (pitch, decay, bend,
    balance, click, filter, resonance). All other per-voice params (FM, filter env, sub mix, etc.)
    are reset to APVTS defaults. No new files are required.

## [1.9.0] - 2026-02-28

### Added

- **SDS-7 independent digital EPROM controls** — the Digital (D) section now has its own
  dedicated parameters separate from the Analog (A) section, matching the real SDS-7 hardware:
  - `D_PITCH` — EPROM playback rate (20–6000 Hz), independent from VCO pitch
  - `D_BEND` — EPROM pitch bend depth (0–100%), independent from VCO bend
  - `D_FM_DEPTH` — FM LFO modulation depth on EPROM (0–100%), independent from VCO FM
  - `D_BALANCE` — EPROM level in mix (0–100%), independent from VCO balance (A Level)
  - 48 new APVTS parameters (4 per voice x 12 voices) with relay/attachment bindings
  - React UI: SDS-7 panel D-mode buttons now control digital-specific params
  - DSP: SDS7Model processes digital params independently (separate pitch bend, FM, and level)

## [1.8.1] - 2026-02-24

### Fixed

- **Knob pointer off-center** — `left: calc(50% - 4px)` correctly centers the 8px-wide
  border-triangle without `translateX(-50%)` (which the JS rotate override was removing)
- **Master-knob inset shadow not applying** — moved `.master-knob .knob-body` rule into
  `index.html` inline `<style>` so it wins over `.knob-body.color-tan` (equal specificity
  0,2,0; inline `<style>` loads after linked `components.css`)
- **Clap-A index line off-center and wrong orbit** — `left: calc(50% - 1px)` centers the
  2px line; `transform-origin: center 18px` places rotation pivot at knob center (21,21)
  instead of (22,18); master-knob variant uses `19.5px`

## [1.8.0] - 2026-02-24

### Added

- **SDS-V amplitude envelope attack** — hardware-accurate 0.5ms non-zero attack on
  the tone/amplitude EG (`eg1`), reflecting capacitor charging behavior on the SDS-V
  module board (0.1–1.0ms RC range per service manual)
  - Previously: instant attack (0ms), snapping immediately to target velocity level
  - Now: 0.5ms exponential rise to target velocity (~99% in 0.5ms via 4.6τ coefficient)
  - Only `eg1` (amplitude) is affected; `eg2` (noise) and `eg3` (click, 4.7ms) unchanged
  - No new user parameters — internal DSP change only; backward compatible

## [1.7.3] - 2026-02-23

### Changed

- **MASTER knob inset shadow** — added deeper concave shadow to the MASTER volume
  knob body for a pressed-in bowl appearance
  - Added `.master-knob .knob-body` rule in `components.css` overriding the generic
    `.color-tan` box-shadow with stronger inset values
    (`inset 0 3px 6px rgba(0,0,0,0.45)` top, `inset 0 -2px 4px rgba(255,255,255,0.18)` bottom)
  - Outer drop shadow retained unchanged; only the inset layers are deepened

## [1.7.2] - 2026-02-23

### Fixed

- **Layout broken after switching from Clap-A** — knobs (RES, PITCH, DECAY, FILTER)
  appeared below the hex pad when switching any voice away from Clap-A
  - Root cause: `makeClapSection` appends wrapper divs to the end of the strip DOM
    (after the hex-pad). Unwrapping moved children back before the wrapper's current
    position — which is still after the hex-pad. PITCH/DECAY/FILTER visually overflowed
    below the pad.
  - Fix: after clap-section cleanup, all `knobContainers` are re-inserted before the
    hex-pad in their original order (indices 0–8)

- **Resonance knob gap for SDS-3/4 and SDS-8** — `visibility: hidden` kept the
  knob invisible but preserved its layout space, leaving a blank gap at the top
  of the channel strip
  - Fix: changed to `display: none` to fully collapse the space; reset path updated
    to restore `display: ''` before applying per-model config

## [1.7.1] - 2026-02-23

### Fixed

- **Knob double shadow (regression from v1.6.0)** — `filter: drop-shadow` was
  reintroduced on `.knob` during the v1.6.0 UI rebuild
  - Root cause: same as v1.5.3 — `.knob-body` already has `box-shadow: 0 4px 8px`,
    stacking two shadow systems on every knob; `filter` also creates a CSS stacking
    context causing Clap-A tick marks to cast shadows onto the knob surface
  - Fix: removed `filter: drop-shadow` from `.knob` in `components.css`

## [1.7.0] - 2026-02-23

### Added

- **SDS-7 filter envelope sweep** — hardware-authentic independent filter envelope
  - New `FILT_ENV` parameter per voice (0–600ms, from SDS-7 service guide)
  - On note trigger, filter cutoff starts 2 octaves above FILTER knob position and
    sweeps down exponentially to the resting FILTER value over the set time
  - `FILT_ENV = 0` (default): no sweep, filter stays static (backward compatible)
  - Root cause: SDS-7 service guide confirms hardware has independent filter sweep
    envelope (8–600ms range) separate from the amplitude envelope
  - UI: FILT ENV knob replaces CLICK for SDS-7 model (CLICK is stored but hidden)
  - DSP: `AREnvelope filterSweepEg` in `SDS7Model.h`; updates CEM3372 cutoff per
    sample while sweep is active, returns to static mode after sweep completes

- **About page panel** — accessible via ABOUT button in top bar
  - Plugin name, version (v1.7.0), subtitle
  - Voice engine list with hardware model and year
  - Research sources (Simmons service manuals, synfo.nl, simmonsmuseum.com)
  - Credits and license section
  - Dismissible via close button or clicking outside the modal

### Changed

- **DECAY parameter max: 2000ms → 2200ms** — matches hardware RC maximum
  - Root cause: SDS-9 service manual documents 220kΩ × 10µF = 2.2s RC time constant
  - Extends parameter range by ~9% to fully cover hardware specification
  - DECAY parameter version unchanged (1); range extension is backward compatible
  - Existing presets with DECAY ≤ 2000ms unaffected

### Technical

- `DrumModel::updateParameters()` interface: added `filtEnvMs = 0.0f` optional param
- `SimmonsDrumModelBase`: added `filtEnvMsParam` protected member
- `SDS7Model`: filter sweep AREnvelope triggered on note-on; sweep active while EG > 0
- All 7 DrumModel direct subclasses updated with no-op `filtEnvMs` parameter

## [1.6.0] - 2026-02-23

### Changed

- **3-column voice chooser** — Voice picker modal redesigned from stacked family/type chips to
  a side-by-side 3-column layout: Model | Type | Preset
  - **Model column** (88 px, scrollable): hardware family — SDS-V, SDS-3, SDS-7, SDS-8, SDS-9,
    Clap-A, D.Clap, SDE, Empty
  - **Type column** (68 px, scrollable): drum type abbreviations filtered by selected model:
    - SDS-V: BS · SN · TM · HH · CY  (CY targets Cymbal variant idx 7, rest target Drum idx 0)
    - SDS-9: TM (idx 4) · BS (idx 10) · SN (idx 11) · RM (idx 12)
    - SDS-3/8: BS · SN · TM
    - SDS-7: BS · SN · TM · HH · CY · RM · CP · CL
    - Clap-A / D.Clap: CP; SDE: PC · HH · CY · BS · TM
  - **Preset column** (flex, scrollable): factory/user presets filtered by model + drum type;
    currently selected preset highlighted; drumType tag removed (type column is the filter)
  - `MODEL_FAMILY_TYPES` data structure encodes family → [{abbrev, label, variantIdx, drumTypes}]
  - `getDrumTypeAbbrev()` maps raw drumType strings → 2-letter abbreviation
  - `buildTypeColumn(familyId, autoSelectAbbrev, setParam)` replaces `buildTypeChips()`
  - `updatePresetsForType(familyId, typeEntry)` replaces `updatePresetsForModel(modelIndex)`
- **Channel strip chooser box** — display updated to show context at a glance:
  - Line 1 (accent colour): `"FAMILY - ABBREV"` e.g. `"SDS-V - BS"` or `"SDS-9 - TM"`
  - Line 2 (dim): preset name only (removed "Drum · " / "Cymbal · " variant prefix)
  - `padPresetState` extended with `typeAbbrev` field; derived from `getDrumTypeAbbrev(preset.drumType)`
    when a preset is loaded, or from `getDefaultTypeAbbrevForModelIndex()` as fallback
- `updatePresetDisplay()` is now the canonical updater for both display lines
  (previously `bindVoicePicker.updateDisplay()` handled line 1 independently)

## [1.5.5] - 2026-02-23

### Changed

- **Two-level voice chooser** (Option A) — voice picker modal redesigned from a flat model list
  to a two-row segmented selector:
  - **Level 1 (family chips):** SDS-V · SDS-3 · SDS-7 · SDS-8 · SDS-9 · Clap-A · D.Clap · SDE · Empty
  - **Level 2 (voice type chips):** appears below Level 1 only when selected family has variants:
    - SDS-9 → Tom · Bass · Snare · Rim
    - SDS-V → Drum · Cymbal
  - Single-variant families set model directly with one click (no type row shown)
  - Channel strip preview: line 1 = family name, line 2 = "Type · Preset" when applicable
  - `MODEL_FAMILIES` data structure encodes the family/variant/index mapping in JS
- **`* 9` → `* 12` normaliser fix** — all MODEL parameter denormalizations updated to use
  `MODEL_MAX_IDX = 12` (was 9, which silently mapped indices 10-12 to wrong models)
- **`applyModelConfig` extended** — cases 10 (SDS-9 Bass), 11 (SDS-9 Snare), 12 (SDS-9 Rim)
  configure knob visibility appropriate to each voice's signal path

## [1.5.4] - 2026-02-23

### Fixed (SDS-9 parameter authenticity — from service manual)

- **SDS-9 noise HPF frequency** (`SDS9Model.h`) — Hardware: RC network 100kΩ × 1µF = 1.59 Hz
  (essentially white noise, no tonal shaping). Corrected from 600 Hz → 1.59 Hz.
- **SDS-9 FM_SCALE constant** (`SDS9Model.h`) — Hardware: 5V CV × (12000–200 Hz) = 11800 Hz
  full modulation range. Corrected from 4000.0 → 11800.0.
- **CEM3394 filter cutoff range** (`CEM3394Filter.h`) — Hardware: timing caps give 200–12000 Hz
  (from SDS-9 service manual). Corrected from 100–15000 Hz → 200–12000 Hz.
- **SDS-9 Second Skin jlimit** (`SDS9Model.h`) — Modulated cutoff clamp updated to match
  corrected CEM3394 range: 100–15000 → 200–12000 Hz.

### Added (SDS-9 architecture — from service manual)

- **`SDS9BassModel`** — Service manual accurate bass voice:
  - No noise generator path (balance fixed at 100% tone, as in hardware)
  - 1 kHz sawtooth click burst (not random noise, matching hardware sawtooth generator)
  - No Second Skin filter FM (bass uses separate discrete generator, not CEM3394 FM path)
  - CEM3394VCO + CEM3394Filter signal path, same chips as SDS-9 tom voice
  - Selectable as "SDS-9 Bass" in MODEL dropdown (index 10)

- **`SDS9SnareModel`** — Service manual confirms snare is EPROM+DAC, NOT analog synthesis:
  - EPROM sample playback (one-shot, no ping-pong loop) — architecture mirrors SDSVCymbalModel
  - CEM3372 filter post-playback (hardware specified; distinct from CEM3394 used for toms)
  - Default procedural content: band-passed noise at 4 kHz (snare buzz spectrum, 180 ms)
  - External EPROM loading supported via `loadExternalSample()`
  - Selectable as "SDS-9 Snare" in MODEL dropdown (index 11)

- **`SDS9RimModel`** — Same EPROM+CEM3372 architecture as SDS9SnareModel:
  - Shorter, brighter burst (80 ms, ~7.5 kHz centre) for rimshot character
  - External EPROM loading supported
  - Selectable as "SDS-9 Rim" in MODEL dropdown (index 12)

- **Voice::Model enum** — Added `SDS9BASS = 10`, `SDS9SNR = 11`, `SDS9RIM = 12`
- **NUM_MODELS** updated from 10 → 13

### Fixed (build)

- **`CA3080VCA::reset()`** — Added no-op `reset()` method to CA3080VCA.h; required by
  SDS9SnareModel and SDS9RimModel which call `vca.reset()` in their `reset()` overrides.
  CA3080VCA is stateless so no-op is correct.

### Notes

- Envelope timing (2.2s hardware max): current DECAY param max is 2000ms (within hardware range).
  No parameter range change required; noted for future consideration.
- New MODEL dropdown choices are appended — existing presets (indices 0–9) unaffected.

## [1.5.3] - 2026-02-23

### Fixed

- **Knob double shadow** — Removed `filter: drop-shadow` from `.knob`; `.knob-body`
  already has `box-shadow: 0 4px 8px` for the drop shadow, causing two shadows to stack
  on every knob. Secondary issue: `filter` creates a CSS stacking context which caused
  Clap-A tick marks (`.clap-a-tick`, `z-index:-1`) to bleed into the composite and cast
  tick mark shadows onto the knob surface.

### Documentation

- Marked 4 SDS-V authenticity fixes as complete in todo.md — these were implemented in
  earlier commits but never checked off:
  - Pitch bend down-only (`SDSVModel.h`: `std::min(0.0f, bendParam)`)
  - SSM2044 resonance scaling 10:1 (`SSM2044Filter.h`: `resonanceScaling() = 10.0`)
  - 10ms parameter smoothing for cutoff and pitch CVs (`ParameterSmoother`)
  - Click envelope 4.7ms attack (`eg3.setAttackTime(4.7f, sampleRate)`)

## [1.5.2] - 2026-02-22

### Fixed

- **Clap-A balance knob label** — Changed from "ENS CRASH" to "BALANCE" (clearer purpose)
- **Clap-A section group boxes** — ENSEMBLE and CRASH bordered boxes now render correctly
  - Root cause: `clap-grp-*` CSS classes were applied but had no CSS rules; `.clap-section`
    wrapper creation code was missing despite cleanup code and CSS being present
  - Fix: Create `.clap-section` wrapper divs with proper flex `order` values, matching
    existing cleanup logic and clap-a.css styling

## [1.5.1] - 2026-02-22

### Changed

- **ClapModel: Schematic-accurate Analogue Clap-Trap DSP** - Complete rewrite of the
  analog clap voice engine based on actual Simmons Analogue Clap-Trap circuit schematic
  - **3 RC integrator envelopes** replace rectangular burst pulses — each integrator
    (IC5/1, IC4/5, IC4/4 with C=2.2nF, R=1MO, tau=2.2ms) produces smooth exponential
    decay, triggered sequentially by the speed oscillator counter
  - **Speed oscillator** (IC1/1, IC1/2) drives hit timing — replaces fixed ms gap
    approach with oscillator-clocked sequential triggering (15-120 Hz range)
  - **Wien bridge VCO** (IC7/1, IC7/2) — cubic soft-shaping rounds triangle peaks
    towards sinusoidal character, matching op-amp bandwidth limiting in hardware
  - **Rattle/Tight control** (R66 pot) — new parameter controlling integrator decay
    overlap: Rattle (tau~1ms, distinct hits) to Tight (tau~8ms, merged sound)
  - **Humanizer** now modulates speed oscillator clock rate per trigger (matching
    D7/R6/R14/C4 circuit), not just random VCO phase offsets
  - **Separate VCA models**: CA3080 OTA for clap path only; crash path uses linear
    transistor VCA (TR6) matching schematic topology

- **ClapModel parameter remapping** for schematic accuracy:
  - `bend` -> Speed (burst rate oscillator frequency, was unused)
  - `res` -> Rattle/Tight (integrator overlap, was SVF Q)
  - SVF bandpass Q now internally fixed at 2.0 (hardware had no Q control)
  - `burstCount`/`burstSpacingMs` parameters ignored (hardware has fixed 3 integrators)

### Added

- **ClapBurstEngine.h** — New DSP component modeling the 3-integrator burst pattern
  circuit from the Simmons Analogue Clap-Trap schematic

## [1.5.0] - 2026-02-22

### Added

- **All 8 YM2203C FM Algorithms** - Complete FM algorithm set for SDE model
  - Algorithm 0 (Chain): OP1→OP2→OP3→OP4 (deep metallic, gong)
  - Algorithm 1 (ParallelMod): (OP1+OP2)→OP3→OP4 (thick organ-like)
  - Algorithm 2 (DualMod): OP1→OP3→OP4 + OP2→OP4 (complex harmonics)
  - Algorithm 3 (SplitChain): OP1→OP2→OP4 + OP3→OP4 (shimmering)
  - Algorithm 4 (TwoPairs): (OP1→OP2)+(OP3→OP4) (SDE primary, bells)
  - Algorithm 5 (OneToThree): OP1→(OP2+OP3+OP4) (brass, reed-like)
  - Algorithm 6 (TwoOpPlusTwo): OP1→OP2 + OP3 + OP4 (mixed FM+additive)
  - Algorithm 7 (AllCarriers): OP1+OP2+OP3+OP4 (pure additive, bright)
  - New `FM_ALGO` parameter (0-7) for per-voice algorithm selection

- **19 SDE Factory Presets** - FM percussion sounds for all three modes
  - Bell/Mallet presets: FM-Bell-1/2, FM-Marimba, FM-Xylophone, FM-Vibraphone, FM-Glockenspiel, FM-Kalimba
  - Metallic presets: FM-HiHat, FM-OpenHat, FM-Ride, FM-Crash
  - Clang presets: FM-Gong, FM-Clang, FM-Steelpan, FM-GamelanBell
  - FM Drum presets: FM-Kick, FM-Tom, FM-Timpani
  - PresetManager engine mapping updated for all 9 models

- **SDS-8 Factory Kit & Presets** - Authentic budget 5-channel configuration
  - "SDS-8 Kit" factory kit: Bass, Snare, Low/Mid/High Toms (5 active voices)
  - Remaining 7 voices set to inactive (-60dB)
  - 7 individual SDS-8 drum presets (BD-8-1/2, SN-8-1/2, LT-8-1, MT-8-1, HT-8-1)
  - CEM3372 filter with fixed resonance (~0.1) - historically accurate

- **Velocity-to-Filter Modulation** - Authentic dynamics for SDS-V and SDS-8 models
  - New `VEL_FILTER` parameter (0-100%) controls velocity-scaled filter cutoff
  - Bass drum gets brighter with harder hits (up to +4000 Hz modulation at max depth)
  - Formula: `effectiveCutoff = baseCutoff + velocity * depth * 4000Hz`
  - Implemented in SimmonsDrumModelBase with `calculateVelocityModulatedCutoff()` helper

- **SDS-9 Waveform Selection** - Triangle or sawtooth VCO
  - New `WAVEFORM` parameter (Triangle/Sawtooth)
  - Sawtooth: brighter, more harmonics for cutting toms
  - Triangle: softer, classic SDS-9 character

- **SDS-9 Sub-Oscillator** - Bass reinforcement for kick drums
  - New `SUB_MIX` parameter (0-100%) blends sub-oscillator
  - Square wave one octave below main VCO
  - Adds weight and depth to bass drum sounds

- **SDS-V Pitched Noise** - Snare sizzle character control
  - New `NOISE_PITCH` parameter (100-5000 Hz)
  - SVFilter bandpass on noise path for tunable snare sizzle
  - Lower frequencies = darker snare, higher = brighter crackle

- **Model-Specific Pitch Ranges** - Authentic hardware constraints per voice engine
  - SDS-3/4: 30-3000 Hz (discrete VCO design, narrower range)
  - ClapModel: 200-4000 Hz (VCO frequency limits)
  - DigitalClapModel: 200-4000 Hz (EPROM playback rate)
  - SDSVCymbalModel: 500-4000 Hz (EPROM clock rate)
  - SDEModel: 20-4000 Hz (FM carrier frequency)
  - Base class (SDS-V/7/8/9): 20-6000 Hz (full range)
  - Virtual `getPitchRange()` method allows per-model override
  - Pitch automatically clamped in `updateParameters()` for each model

- **SDS-9 Velocity-Dependent Second Skin FM** - More expressive filter FM dynamics
  - FM depth now scales by hit velocity (harder hits = more Second Skin effect)
  - Soft hits (~30% velocity): 30% of knob's FM depth
  - Hard hits (~100% velocity): 100% of knob's FM depth
  - Creates more dynamic, expressive Second Skin timbres

- **SDS-V Cymbal SWEEP Control** - Time-varying filter for authentic cymbal behavior
  - Bend parameter repurposed as SWEEP control for cymbal model
  - Center (0): Static filter (no sweep)
  - Right (+): Bright-to-dull (filter sweeps down over envelope decay)
  - Left (-): Dull-to-bright (filter sweeps up over envelope decay)
  - ±8000 Hz sweep range, envelope-modulated cutoff
  - Uses SVFilter lowpass for smooth timbral evolution

- **SDS-8 Velocity Saturation Curve** - Authentic compressed dynamics
  - Non-linear velocity curve matching SDS-8's "punchier" character
  - Uses tanh saturation: ~78% output at 50% velocity, ~95% at 75%
  - Reaches full output earlier than linear curve (budget rock-focused design)
  - Consistent with "maximum output occurs at 75mV with 1.5V max input"

- **SDE Per-Operator Detune** - Fine-tuning for complex FM timbres
  - Added `setOperatorDetune(int opIndex, int cents)` API (-400 to +400 cents)
  - Creates beating and shimmer effects between operators
  - Critical for realistic bell and metallic sounds
  - 100 cents = 1 semitone, applied as frequency multiplier: 2^(cents/1200)

- **Empty Voice Slot** - Disable unused voice slots for zero CPU overhead
  - New "Empty" option in MODEL dropdown (10 total models now)
  - Created `EmptyModel.h` - null DrumModel with minimal footprint
  - `processSample()` returns 0.0f, `isActive()` returns false
  - Voice processor skips all DSP processing for empty slots
  - Ideal for authentic 5-channel SDS-8 kits or partial kits

### Changed

- DrumModel interface: `updateParameters()` now accepts `velToFilter`, `waveform`, `subMix`, `noisePitch`
- SimmonsDrumModelBase: Added parameter cache for new model-specific parameters
- SDS9Model: Uses waveformParam and subMixParam for VCO selection and sub-osc mixing
- SDSVModel: Added SVFilter noiseBandpass for pitched noise path

### Fixed

- **Bend direction reversed** - Negative bend now correctly bends pitch DOWN
  - Root cause: `pow(2, bendDepth * eg1Level)` gave upward bend for negative values
  - Fix: Negate bendDepth in `calculateModulatedFrequency()` so negative bend = downward
  - Now matches authentic Simmons behavior: negative bend starts high, falls to target pitch

### Maintenance

- Removed dead voice-browser CSS/JS code (replaced by combined voice-picker modal)
  - Cleaned components.css, index.html inline CSS/JS, sds-3.css, sds-v.css

## [1.4.0] - 2026-02-22

### Added

- **YM2203C.h** - Complete 4-operator FM synthesis engine
  - 20-bit phase accumulators with proper wrapping
  - Per-operator ADSR envelopes with YM2203C-style exponential curves
  - OP1 self-feedback (0-7 levels) for harmonic richness
  - Three algorithm implementations for drum sounds:
    - **Algorithm 0 (Chain):** OP1→OP2→OP3→OP4 for deep metallic sounds
    - **Algorithm 4 (TwoPairs):** OP1→OP2 + OP3→OP4 (SDE primary mode)
    - **Algorithm 7 (AllCarriers):** All operators as carriers for bright additive sounds
- **SDEModel.h** - DrumModel wrapper for YM2203C
  - Balance parameter controls FM mode (0-33% bell, 33-66% metallic, 66-100% clang)
  - Filter controls modulator frequency ratio (1-15x)
  - Resonance controls feedback amount (0-7)
  - Click controls modulation depth
  - Full FM Speed/Depth parameter support
- **SDE voice engine** in Voice.h (Model index 8)
  - Historical: SDE (1986) was "the only FM synth ever made with knobs"
  - Authentic YM2203C FM percussion for bells, marimbas, hi-hats, metallic clang
- **YM2203C Vintage Mode** - Authentic SDE character
  - Internal synthesis at ~8,287 Hz (original chip rate)
  - 13-bit quantization (YM3014 DAC emulation)
  - Linear interpolation upsampling to host sample rate
  - Adds characteristic "crunch" and aliasing artifacts
- **Per-Voice EPROM Assignment** - Independent sample loading for each pad
  - 12 independent EPROMManager instances (one per voice)
  - Per-pad EPROM status indicators in WebView UI
  - `loadCymbalEPROM(voiceIndex, file)` API
  - Visual "ROM" badge on pads with loaded EPROMs
- **Unit Tests** - New test coverage
  - SDS8ModelTests.cpp (8 test cases)
  - SDEModelTests.cpp (11 test cases)
  - VCODistinctionTests.cpp (acoustic differences between models)

### Changed

- Model count: 8 → 9 (SDS-V, SDS-3, SDS-7, SDS-8, SDS-9, Clap, D.Clap, Cymbal, SDE)
- Updated all UI components for 9-model support
- **Model-specific bend knob styles** - Historical accuracy via `getModelConfig()` API + WebView UI
  - `bendStyle: "traditional"` for SDS-3/4/V, Clap, D.Clap (0-100% knob, down-only)
  - `bendStyle: "bipolar"` for SDS-7/8/9, SDE (center=0, left=down, right=up)
  - `bendStyle: "hidden"` for Cymbal (EPROM-based, no pitch bend)
  - WebView UI: Knob rotation and drag behavior adapt to bend style
  - Value display: Traditional shows "0-100%", bipolar shows "-100 to +100"

### Fixed

- **SDSVCymbalModel Nyquist singularity** - Filter cutoff at exactly 0.5 normalized
  caused sin(π) = 0, making all biquad coefficients zero (silent output)
  - Root cause: `fc = 8000/16000 = 0.5` when filter cutoff = EPROM render rate / 2
  - Fix: Clamp fc to [0.05, 0.45] to avoid Nyquist edge case

### Technical Notes

- FM uses phase modulation (not frequency modulation) matching YM2203C behavior
- Inharmonic ratios (7.07, 11.3, 14) essential for metallic/cymbal timbres
- CPU budget: ~80-120 FLOPS/sample for 4-operator FM (cheapest model in SDHex)
- No oversampling required (no nonlinear saturation to alias)
- High-level API (Hz, ms, %) with internal conversion to FM parameters

## [1.3.0] - 2026-02-19

### Added

- **DiscreteVCA.h** - Discrete transistor VCA for SDS-3/4 (1978-1980) authenticity
  - Less precise exponential response than OTA designs (more "organic")
  - Soft polynomial saturation (gentler than OTA's tanh curve)
  - CV feedthrough modeling (subtle transient click on envelope attack)
  - Thermal noise floor for vintage character
  - 60dB dynamic range (vs 80dB for CEM3360)
- **CEM3360VCA.h** - Curtis CEM3360 dual VCA for SDS-9 (1985) "hi-fi" character
  - Precise log conformance over 80dB range (factory-trimmed accuracy)
  - Low distortion (<0.05% THD) - cleaner than OTA designs
  - Linear mode option for proportional CV-to-gain
  - Minimal CV feedthrough (internal crosstalk cancellation)
  - Soft clipping only at extreme levels
- **EPROM UI Integration** - WebView file browser for user-loadable .bin files
  - `browseForEPROM()` native function with juce::FileChooser
  - EPROM button in top bar with load/clear toggle
  - Loading state indicator with CSS animation
  - Status display for filename/errors
- **Dynamic UI Based on Model** - Parameter visibility per model
  - `getModelConfig()` native function returns visibility flags
  - `applyModelConfig()` hides/dims unavailable parameters
  - 8 model configurations supported (SDS-V, SDS-3, SDS-7, SDS-8, SDS-9, Clap, D.Clap, Cymbal)
- **SDS-8 voice engine** - Budget Simmons model (1983-1984)
  - CEM3372 filter (same as SDS-7) but with fixed resonance (~0.1)
  - Standard triangle VCO path (RC LPF → Op-Amp coloring → VCA)
  - No FM capability - simpler, more accessible sound
  - "Similar to SDS-V but softer, more musical saturation" character
- Unit tests for DiscreteVCA and CEM3360VCA

### Changed

- **SDS3Model.h** - Now uses `DiscreteVCA` (shadows base class CA3080VCA)
- **SDS9Model.h** - Now uses `CEM3360VCA` (shadows base class CA3080VCA)
- Model-specific VCAs wired for authentic per-era character

### Technical Notes

- DiscreteVCA models BJT differential pair mismatch with 15% nonlinearity factor
- CEM3360 uses dB-linear mapping: gain = 10^((CV-1) * 80/20)
- VCA selection per model: SDS-3/4 → DiscreteVCA, SDS-V/7/8 → CA3080, SDS-9 → CEM3360
- VCAs shadow base class member via C++ name hiding (no virtual overhead)
- SDS-8 uses CEM3372 filter with k=2.0 (softer than SSM2044's k=4.0)
- SDS-8 ignores user resonance parameter - internal fixed resonance of 0.1

## [1.2.0] - 2026-02-19

### Added

- **SDS-9 voice engine** - New drum model emulating the Simmons SDS-9 (1985) "Second Skin" technology
  - CEM3394 VCO with triangle, sawtooth, and sub-oscillator outputs
  - CEM3394 ZDF filter with k=1.5 saturation (between CEM3372 and Moog)
  - **Second Skin FM**: VCO modulates filter cutoff at audio rate for complex timbres
  - `fastTan()` approximation enables per-sample cutoff updates without CPU overhead
- **SDS-V Cymbal voice engine** - EPROM-based cymbal/hi-hat with authentic SDS-V character
  - Bidirectional (ping-pong) EPROM playback for sustained metallic tones
  - Procedurally generated metallic content (noise → resonant bandpass → 8-bit quantization)
  - User-loadable external EPROM .bin files via `loadCymbalEPROM()` API
  - Authentic 8-bit quantization with zero-order hold reconstruction
- `CEM3394VCO.h` - Multi-waveform bandlimited oscillator
  - Triangle output with polyBLAMP anti-aliasing
  - Sawtooth output with polyBLEP anti-aliasing
  - Sub-oscillator (square wave, one octave below)
- `CEM3394Filter.h` - ZDF 4-pole filter optimized for audio-rate modulation
- `SDS9Model.h` - Complete SDS-9 signal path with Second Skin FM
- `SDSVCymbalModel.h` - EPROM-based cymbal/hi-hat voice engine
- **EPROMPlayer bidirectional mode** - Ping-pong looping for sustained tonal content
  - Forward/reverse playback with smooth direction changes
  - Looping vs one-shot modes
  - Address tracking for visualization
  - `loadSampleData()` methods for external sample loading
- `EPROMManager.h` - Thread-safe user-loadable .bin file support
  - Double-buffered loading (non-blocking audio thread)
  - Async file loading with atomic commit
- `YM2203C.h` - Placeholder stub for future FM synthesis chip emulation
- **VCO distinction tests** - Verify SDS-V vs SDS-3 produce measurably different output
- Unit tests for all new components (TDD approach)
- PluginProcessor: `loadCymbalEPROM()`, `clearCymbalEPROM()`, `hasCymbalEPROM()` API

### Changed

- Voice::Model enum now includes `SDS9` and `CYMBAL` (7 total models)
- MODEL parameter version bumped to 6 (Cymbal option added)
- Model indices updated: 0=SDS-V, 1=SDS-3, 2=SDS-7, 3=SDS-9, 4=Clap, 5=D.Clap, 6=Cymbal
- EPROMPlayer now supports bidirectional/looping configuration and external sample loading
- Tests/CMakeLists.txt updated with new test files

### Technical Notes

- CEM3394 filter uses k=1.5 saturation (CEM3372 = 2.0, SSM2044 = 4.0, Moog = 1.0)
- Second Skin FM depth maps 0-100% to +/- 4000 Hz filter modulation
- fastTan() Taylor series approximation: accurate for |x| < π/4
- EPROMManager uses std::atomic for lock-free thread synchronization
- Cymbal EPROM playback uses bidirectional ping-pong (period = 2*(N-1) samples)
- Procedural cymbal content: white noise → high-Q bandpass → 8-bit quantization at 16kHz
- All 7 voice engines pre-allocated for real-time safety

## [1.1.0] - 2026-02-19

### Added

- **SDS-3/4 voice engine** - New drum model emulating the earliest Simmons drum machines (1978-1980)
  - Classic Moog transistor ladder filter (Huovilainen ZDF, k=1.0 saturation)
  - Warmer, softer character compared to the SDS-V's SSM2044 filter
  - No passband attenuation under resonance (unlike SSM2044)
  - Same triangle VCO + envelope architecture as SDS-V
- `MoogLadderFilter.h` - New DSP component implementing classic Moog 4-pole 24dB/oct ladder
- `SDS3Model.h` - Complete SDS-3/4 signal path implementation
- Model selection dropdown now includes "SDS-3" option (index 1)

### Changed

- Voice::Model enum now includes `SDS3` between `SDSV` and `SDS7`
- MODEL parameter version bumped to 4 (new option added)
- Model indices updated: 0=SDS-V, 1=SDS-3, 2=SDS-7, 3=Clap, 4=D.Clap

### Technical Notes

- Moog ladder uses k=1.0 saturation coefficient (standard transistor differential pair)
- SSM2044 uses k=4.0 (proprietary OTA-like gain cells at lower signal levels)
- SDS-3/4 historically used ARP 4035 variant Moog-style ladder filters
- All 5 voice engines pre-allocated for real-time safety

## [1.0.0] - 2026-02-14

### Added

- Initial release
- SDS-V (1981) voice engine with SSM2044 filter emulation
- SDS-7 (1984) voice engine with CEM3372 filter and FM modulation
- Analog Clap voice engine (Simmons ClapTrap style)
- Digital Clap voice engine (MK3 style)
- 12-voice polyphony with multi-output routing (14 outputs total)
- Per-voice parameters: Pitch, Decay, Bend, Balance, Filter, Resonance, Click, Pan, Volume
- SDS-7 specific: FM Speed, FM Depth parameters
- GM drum map MIDI note assignment
- Preset system with kit and drum preset save/load
- WebView UI with hex pad layout
