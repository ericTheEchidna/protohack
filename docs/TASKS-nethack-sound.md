Protohack / nethack-protohack
Sound support — wire platform backends into Meson build

## Phase N3 — Sound Support

Wire NetHack's sound backend system into the Meson build with sensible
platform defaults. Prerequisite: Phase N1 (Meson build) complete.
Tiles (N2) can be done in parallel with this task.

### Context

Sound backends live in `sound/`:
- `sound/wav/` — WAV file playback via OS primitives (Linux: ALSA or
  libao; macOS: AudioToolbox; Windows: winmm)
- `sound/fmod/` — FMOD Engine (proprietary, cross-platform, high quality)
- `sound/qtsound/` — Qt multimedia backend
- `sound/macsound/` — macOS-native (AudioToolbox directly)
- `sound/windsound/` — Windows-native (winmm / MCI)

Platform default targets:
- **Linux** — `wav` backend (ALSA or libao; no proprietary dependency)
- **macOS** — `macsound` (native AudioToolbox; zero extra deps)
- **Windows** — `windsound` (native winmm; zero extra deps)

FMOD and Qt sound are opt-in, not default. They require external SDKs that
a contributor should not have to install to get a working build.

### Tasks

- [ ] Audit `sound/wav/`, `sound/macsound/`, `sound/windsound/` — identify
      source files, required system libraries, and the NetHack sound API
      they implement (likely a set of function pointers or a `soundprocs`
      struct analogous to `window_procs`)
- [ ] Identify the sound API contract in `include/` — what does a backend
      have to implement? (grep for `soundprocs` or `sndprocs`)
- [ ] Add sound backend selection to `meson.build`:
  - [ ] Meson option `sound` with choices `['wav', 'fmod', 'qt', 'none']`
        and platform-appropriate default
  - [ ] On Linux: `sound/wav/` sources + `dependency('alsa')` or
        `dependency('ao')` (prefer libao as it abstracts ALSA/PulseAudio)
  - [ ] On macOS: `sound/macsound/` sources + AudioToolbox framework link
  - [ ] On Windows: `sound/windsound/` sources + `winmm` lib link
  - [ ] `none` option compiles a no-op stub so the build never fails due
        to missing audio dependencies
- [ ] Wire the selected backend into the main executable target
- [ ] Smoke test on Linux: a NetHack sound event (e.g. hitting a monster)
      produces audible output or at least does not crash with `none` stub
- [ ] Document the `sound` Meson option in a `README.build.md` or inline
      in `meson.build` comments

### Source references

- `sound/wav/`, `sound/macsound/`, `sound/windsound/` — backend sources
- `include/` — grep for `soundprocs`, `sndprocs`, or `sound_procs` to
  find the API contract
- `src/sounds.c` (if it exists) — likely the dispatch layer between
  NetHack core and the backend

### Complexity notes

- Do not add FMOD support in this task. FMOD requires a registered
  developer account and SDK download; it blocks any contributor who
  doesn't have it. Leave it as a documented opt-in.
- libao abstracts ALSA, PulseAudio, and PipeWire on Linux — prefer it
  over raw ALSA so the build works on modern distros where PulseAudio or
  PipeWire is the default.
- The `none` stub must compile and link cleanly. Sound failure at runtime
  should log a warning, not crash. NetHack has historically been playable
  without sound.
- macOS AudioToolbox is a system framework, not a pkg-config dependency.
  Use Meson's `dependency('appleframeworks', modules: ['AudioToolbox'])`.

### Done when

- `meson setup builddir && ninja -C builddir` completes on Linux with the
  `wav` backend selected by default
- `meson setup builddir -Dsound=none` completes and produces a binary
  that runs silently without error
- Platform default backend compiles without requiring any manually
  installed SDK beyond system libraries
- Sound backend selection is documented in meson.build or README.build.md
