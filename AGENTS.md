# AGENTS.md — HeldController (WLED fork) AI Coding Agent Reference

This repository is **HeldController**: a fork of WLED that is the LED controller for the "Held"
art installation. Read the **Held fork** section first. Everything after it is the upstream WLED
reference, which still applies to the WLED code this fork is built on.

WLED is C++ firmware for ESP32/ESP8266 microcontrollers controlling addressable LEDs,
with a web UI (HTML/JS/CSS). Built with PlatformIO (Arduino framework) and Node.js tooling.

See also: `.github/copilot-instructions.md`, `.github/agent-build.instructions.md`,
`docs/cpp.instructions.md`, `docs/web.instructions.md`, `docs/cicd.instructions.md`,
`docs/hardening.instructions.md`, `docs/securecode.instructions.md`.

Always reference these instructions - including coding guidelines in `docs/*.instructions.md` - first and fallback to search or bash commands only when you encounter unexpected information that does not match the info here.

Design documentation and AI agent temporary notes can be placed into `docs/` sub-folders, like `docs/architecture/`, `docs/design/`, `docs/requirements/`, `docs/testing/`, `docs/temp/`. `docs/temp/` is listed in `.gitignore` so its the preferred location for notes that should not become part of a PR.

## Held fork

User-facing documentation for the fork is in `docs/held/README.md`: setup, the control API, how the engine works, and how to add a layer. This section holds what an agent needs on top of that: project context, invariants, design decisions with their reasons, and environment lessons.

### Project context

- **Held** is a ~750 × 500 mm 3D-printed anatomical heart, suspended from rigging. The owner did the modelling in Autodesk Fusion.
- Six "missing pieces" hang on thin chains. Pushing a piece back in trips a hall sensor. When all six are held in at once, recorded stories about being heartbroken and then held by community play. Six is enough that it takes at least two people.
- The **audio runs on a separate ESP32**, owned by another artist. This firmware only handles LEDs.
- **Audio brain link (implemented 2026-11-08):** the audio "brain" ESP32 reads the six pocket sensors and sends plain-text lines (`BEAT`, `COUNT`, `STORY END`) over a UART. The spec is in the sibling repo `../held-electronics` (`docs/Held_Lighting_Serial_Messages.md`, a proposal by Ellen Juhlin). The brain sketch there **doesn't send the messages yet**. Our side is `wled00/held/held_link.{h,cpp}`, with the UART on GPIO32 (RX) / GPIO33 (TX). Answers to the proposal's open questions are in `docs/held/README.md` → Audio link. We asked for `STORY START` and for `HELLO` → `COUNT` resync. If the protocol changes, update `held_link.cpp`, `tools/held/link_test.cpp` and that README section together.
- **LED hardware:** WS2812 strips in roughly circular loops inside the heart. Default is one strip of 120 LEDs at 60 LED/m; a second, roughly orthogonal loop may be added.
- **Status (2026-11-06):** first flashed to real hardware, a TinyPICO bench board with 60 LEDs. It booted, reported 60 LEDs on the Held effect, and accepted `{"held":...}` over serial. The heart's own wiring and controller are not set up yet. The engine was verified natively against the browser engine. The UI was tested in headless Chrome, in simulator mode and against a mock controller.

### How the user likes to work

- They want everything tunable, both from code/API and from the web UI, so they can explore the visual design space. Exercise judgment and add natural extensions. Ask before substantial deviations, and flag any spec defaults you changed, with the reason.
- They prefer the **lub-dub** waveform (now the default).
- New parameters should come with sensible defaults, help text, and a UI control.

### Architecture

| Part | Where | Notes |
|---|---|---|
| Param tables | `wled00/held/held_params.h` (X-macros) ↔ `wled00/data/held/schema.js` | Same order, keys, types, limits and defaults; `npm test` enforces it. Event groups must have `enabled` as param 0. |
| Engine | `wled00/held/held_core.{h,cpp}` ↔ `wled00/data/held/engine.js` | Two implementations kept **operation-for-operation identical**. |
| Expressions | `wled00/held/held_expr.{h,cpp}` ↔ `wled00/data/held/expr.js` | Same grammar, bytecode and function semantics. |
| WLED glue | `wled00/held/held_wled.cpp`, `held.h`, `held_input.cpp` | Effect, JSON API, `/held.json`, engine mutex, the audio link UART. |
| Audio link | `wled00/held/held_link.{h,cpp}` | Portable protocol handler (no Arduino), driven with bytes by `held_wled.cpp`. Firmware-only: the JS engine has no counterpart, but it has the engine features the link uses. |
| UI | `wled00/data/held/{index.htm,app.js,view3d.js,view3d_gl.js,held.css}` | Inlined into `wled00/html_held.h` by `tools/cdata.js`. The Held UI is served at `/`; stock WLED moved to `/classic`. |
| Heart model | `tools/held/heart/build_heart.py` → `../istari-arts-site/src/static/held/heart_model.js` | Built from the printed chunk STLs with headless Blender; hosted, never in the firmware. |
| Tooling | `tools/held-test.js`, `tools/held/{parity.js,parity_main.cpp,kymo.js,devserver.js}` | See Testing below. |

Hooks into WLED itself are small and marked `// AI:`, all behind `#ifdef WLED_ENABLE_HELD`:

- `wled.cpp`: `heldSetup()` and `heldLoop()`.
- `wled.h`: the include.
- `json.cpp`: the `"held"` key in `deserializeState`, and the `json_target::held` serve target.
- `wled_server.cpp`: `POST /json/held`, `/` and `/classic`.

**Engine model:**

- **Layers.** Each strip composites, in order: heartbeat, ambient, then events in trigger order (pieceIn, pieceOut, allIn). Values are perceptual 0..1 floats; WLED applies gamma and `bri`.
- **Virtual strip.** Each strip renders into a virtual strip of `round(leds/reps)` pixels with the origin at pixel 0. `tile()` then repeats or mirrors that around the physical strip starting at the origin, so repetitions apply to the whole composite.
- **Level.** `min(pieces, numPieces-1) / (numPieces-1)`, smoothed by `levelSmoothing`.
- **Param values.** A number, a `[level0, level1]` pair (the `ranged` flag is explicit; any FLOAT/INT can be ranged), or an expression string (FLOAT/INT layer params only, at most 24 at once). Colours are `"#RRGGBB"` in JSON.
- **Event-time (`et`) expressions** are re-resolved per event by `resolveEt(group, age)`. The calls sit in `trigger`, `cancel`, `tickEvents`, `render`, `state`/`eventInfo`, and the grace check in `setPieces`. Keep those call sites identical in both engines.
- **Events:**
  - Count up fires `pieceIn`; reaching all pieces fires `allIn` instead (not restarted if already running).
  - Count down fires `pieceOut`, and cancels `allIn` if the count dropped from all pieces.
  - In the last `releaseGrace` seconds of `allIn`, removals are ignored but still change the level.
  - With `allInTrigger` = "Story start", reaching all pieces fires `pieceIn`, and `allIn` waits for the link's `STORY START`.
  - `storyEnd` (from the link's `STORY END`) is a plain timed event.
- **External heartbeat clock.** `setExternalClock(true)` stops the internal BPM phase, and each `beat()` emits one pulse at `lutPeak × pulseWidth`, so the waveform's peak (the lub) is at the origin when the beat arrives. `setPieces(mask, false)` adopts a count without events. The link uses these; the parity check covers them in both engines.
- **Link mapping** (`held_link.cpp`):
  - `BEAT` → `beat()` while the global `beatSync` is on. After `beatTimeout` seconds without a `BEAT`, `poll()` returns to the internal clock.
  - `COUNT s r` → `numPieces = r`, and the lowest `s` bits as the mask. The first `COUNT` after boot is silent.
  - `STORY START` → `allIn` (story mode only). `STORY END` → `storyEnd`. `PING` → `PONG`. Unknown lines are counted and ignored.
- **Firmware.** Segment *i* = Held strip *i* (`WLED_AUTOSEGMENTS` makes one segment per LED output), up to 4 strips; LED counts come from segments. `autoApply` keeps every segment on the Held effect. Engine access goes through a FreeRTOS mutex, and the renderer never waits: it keeps the last frame instead.

### Invariants: read before changing the engine

1. **Change both engines together.** Mirror every change in C++ and JS, in the same order of operations. PRNG draws (`rand01()` / `_rand()`) must happen in the same sequence. In C++, draw into separate statements, because argument evaluation order is unspecified.
2. **C++ number types.**
   - Use the `hf` type (float on the MCU, double for parity).
   - Write non-dyadic constants as `HC(0.3)`, so float builds stay single-precision.
   - Expressions evaluate in `double`, so `t` stays smooth over long uptimes.
3. **Run `npm test` after engine changes.** It includes the native parity check (`tools/held/parity.js`), which compiles the C++ core with MSVC/g++/clang and diffs frames against engine.js (tolerance 1e-4; matches are typically about 5e-8). For new behaviour, add a parity scenario that exercises it. The parity check is skipped if no compiler is found.
4. **Avoid long-uptime float drift.** Noise time wraps at a periodic lattice (`NOISE_TP`), and pan offsets wrap per strip, so float precision on the device doesn't degrade after days of uptime. Don't add raw ever-growing float accumulators.
5. **Keep stock WLED building.** Everything is behind `WLED_ENABLE_HELD` (ESP32-only, set only in `[env:held_esp32]`). Check `pio run -e esp32dev` as well as `pio run -e held_esp32`.
6. **Keep the UI small and self-contained.** No external libraries or CDNs: phones on the controller's AP have no internet. Use classic `<script>` tags, not ES modules, so `index.htm` works from `file://`. Tabs for indentation. The current size is about 24 KB gzipped.
7. **Don't use ArduinoJson `const char*` for dynamic strings.** Assigning a `const char*` stores a pointer, which dangles after the engine lock is released and the async response is sent. Assign `char*` (copied) for engine-owned strings.

### Design decisions, and why

- **Heartbeat speed defaults are 14 → 22 LED/s**, not the originally specified 5 → 10. Beats are `speed × 60 / BPM` LEDs apart; at 5 LED/s and 45 BPM that is 6.7 LEDs, less than the 10-LED waveform, so the beats blur into a texture. This was confirmed with kymographs.
- **Pulse width is the span of one whole beat waveform**, not of a single bump.
- **Origin, repetitions and mirroring are per strip, not per layer**, so events originate from the same point as the heartbeat.
- **three.js only as a progressive enhancement.** It is about 170 KB gzipped and needs a CDN, so it is never bundled. The page always starts on the canvas-2D view (`view3d.js`), which works on the controller's AP without internet. `view3d_gl.js` then tries `import('three')` through the import map in `index.htm`. If that works, it swaps in a WebGL view with the same interface and loads the heart model. `?lite` skips the upgrade. Keep the three.js version in the import map and `THREE_BASE` in sync.
- **The heart model is a script, not a `.glb` fetch.** istariarts.org sends no CORS headers. A `<script>` tag loads cross-origin anyway, so `heart_model.js` (`window.HeldHeartModel = {glb: base64}`) works from the hosted demo, from `http://held.local` and from `file://`. `?heart=<url>` overrides the location for testing.
- **The heart model is drawn at true scale**: millimetres, centred on its bounding box, front toward +z. It is never scaled to fit the LEDs. The strip loop is an ellipse (`aspect`, plus `cx`/`cy`/`cz` offsets) with its perimeter fixed by `leds / density`, so the view shows honestly whether a strip fits.
- **Expressions use a custom language rather than a JS engine.** QuickJS is too large for the flash. Elk and mJS are GPL/AGPL, which is unsuitable for this project. Params are evaluated once per frame, not per pixel, so a small bytecode VM is cheap. The JS side intentionally avoids `eval`, so the simulator can never accept something the device can't run.
- **WLED features are compiled out with flags, not deleted.** Alexa, Hue, PixelForge, particle FX and usermods are off. E1.31/Art-Net/DDP are still compiled in. Flash is at about 83.5% of the app partition; disable more if it gets tight.
- **`noisePan` defaults to 1.5 LED/s** so the drift is visible; 0 gives the old in-place look.
- **The default strip doesn't fit in the heart.** 120 LEDs at 60/m is a 2 m loop. Measured from the model, the interior on the mid front/back plane is only about 360 mm wide (217 mm left of the bounding-box centre, 143 mm right). About 84 LEDs at 60/m (1.4 m) with `aspect` 1.55, `cx` −30 and `cy` −40 fits in the front view. Fixing the default needs a hardware decision (strip length or density).

### Testing and verification

- `npm test` runs the schema consistency check, engine and expression tests, the parity check, the native audio-link unit test (`tools/held/link_test.cpp`), and WLED's own `cdata` tests. Native builds share `tools/held/native.js`.
- Test the link on a device without the brain: **Setup → Audio link** in the UI, or `{"held":{"link":"BEAT 1 65.0 349 923"}}` over any JSON transport.
- **Visual checks:**
  - `node tools/held/kymo.js --out k.png --seconds 20 --script '[[2,63]]' --strip '{"leds":120,"reps":2}'` renders a run to a space-time PNG. View it to judge pulse spacing, speed and event timing.
  - For UI checks, drive headless Chrome (`chrome --headless=new --remote-debugging-port=N`) over the DevTools protocol with Node's built-in `WebSocket`: click elements, read the DOM, catch `Runtime.exceptionThrown`, and take screenshots. Do this on a copy of the built page, gunzipped from `wled00/html_held.h`, to catch minifier issues. Also test a phone viewport (390 px).
  - For device mode without hardware, stand up a small Node mock of `/json/held` backed by engine.js. Only a mock has been tested so far, never real firmware.

### Environment lessons (Windows dev machine)

- **Run PlatformIO from PowerShell or cmd, never Git Bash.**
  - The ESP-IDF tool installer rejects MSYS ("MSys/Mingw is not supported"). It leaves empty toolchain packages, and the build later fails with `xtensa-esp32-elf-g++ is not recognized`.
  - In PowerShell, run `Remove-Item Env:MSYSTEM` first.
  - `pio` may not be on PATH: use `%USERPROFILE%\.platformio\penv\Scripts\pio.exe`.
- **Don't pipe `pio` into `head`.** The build blocks once the pipe closes.
- **Don't run `npm test` and a firmware build at the same time.** Both regenerate the `html_*.h` / `js_*.h` headers, which causes `EBUSY` errors.
- **The parity check finds MSVC through `vswhere`** when no g++/clang is on PATH.
- **Backslash escapes in Bash heredocs can be mangled** by some agent shells (`'\n'` became a literal newline). For edits containing escapes, use the editor tool or a script file.
- **Second dev machine (2026-11):**
  - Node is managed by nvm-windows (`NVM_HOME=%LOCALAPPDATA%\nvm`), with 22.x active through the symlink `C:\nvm4w\nodejs`. The old system Node 12 was uninstalled. Shells started before that change may still lack the new PATH; restart them.
  - Blender 5.2 is in `C:\Program Files\Blender Foundation\Blender 5.2`. No Blender MCP server is configured, so it runs headless (`blender -b --factory-startup --python ...`).
  - There is no Chrome; drive headless Edge (`msedge.exe --headless=new`, plus `--use-angle=swiftshader` for WebGL) the same way.
  - PlatformIO comes from the VS Code extension: `%USERPROFILE%\.platformio\penv\Scripts\pio.exe`. Its build hook runs `npm ci` / `npm run build` with the `node` on PATH; an old Node fails with `Cannot find module 'node:fs'`. The first ESP32 build failed once with "Failed to install Python dependencies into penv" (output is hidden). Running the same `uv pip install` by hand showed everything installed, and a retry built fine.
  - Bench board: TinyPICO (ESP32-PICO-D4) on COM7, with 60 × WS2812 on GPIO 23. `platformio_override.ini` (gitignored) defines `[env:held_tinypico]` with `-D DATA_PINS=23 -D PIXEL_COUNTS=60`. These are only first-boot defaults; a saved `cfg.json` wins.
  - To reset device settings: `python -m esptool --port COM7 --baud 460800 erase-region 0x310000 0xF0000` erases the LittleFS partition and keeps the firmware. 921600 baud is unreliable on this board. The pre-Held settings are backed up in `docs/temp/tinypico_{nvs,fs}_*.bin`.
  - Serial JSON works for quick checks: send `{"v":true}` plus a newline at 115200, using pyserial from the penv Python with DTR/RTS low so the board isn't reset.
  - PowerShell 5 strips quotes from JSON passed as an argument to native programs; pass a file instead.

### Public demo

The simulator is hosted at https://istariarts.org/held_simulator.html for teammates. It comes from the sibling repo `../istari-arts-site` (`git@github.com:Tossrock/istari-arts-site.git`): a push to its `main` deploys over scp through `.github/workflows/deploy.yaml`, which lists every deployed file explicitly. To refresh it after UI or engine changes:

1. Run `npm run held:export -- ../istari-arts-site/src/held_simulator.html`.
   - If the heart STLs changed: `blender -b --factory-startup --python tools/held/heart/build_heart.py -- Held_Heart_Pieces ../istari-arts-site/src/static/held/heart_model.js 0.2`. This uses the newest `_vNN` of each chunk. Ratio 0.2 gives about 63k triangles and a 177 KB Draco GLB. The STLs (about 16 MB) are not committed.
2. Commit and push that repo.

Hosted on https, the page can only simulate, because browsers block calls to `http://` controllers (mixed content).

### Open items and next steps

- **Hardware bring-up:**
  - Flash `held_esp32` and join `Held-AP` (password `heldgently`).
  - Configure the LED outputs, then confirm segments map to strips and the origin and direction are right on the physical loop.
  - Check `/json/held` and the live view.
- **Audio link end-to-end:**
  - Not yet tested with real hardware. The brain sketch doesn't send the messages yet, and no board was attached when the link was written.
  - When it does: wire brain GPIO4 → lighting GPIO32, brain GPIO18 ← lighting GPIO33, and GND.
  - Then check Setup → Audio link, and tune the brain's `BEAT` delay by eye.
  - The light's dub currently comes from the waveform shape, not the message's `dub ms`. Matching it exactly is a possible refinement.
- **Sizing:** decide strip density and length for the heart's actual size.

## Build Commands


| Command | Purpose | Timeout |
|---|---|---|
| `npm ci` | Install Node.js deps (required first) | 30s |
| `npm run build` | Build web UI into `wled00/html_*.h` / `wled00/js_*.h` | 30s |
| `npm test` | Run test suite (Node.js built-in `node --test`) | 2 min |
| `npm run dev` | Watch mode — auto-rebuilds web UI on changes | continuous |
| `pio run -e esp32dev` | Build firmware (ESP32, most common target) | 5 min |
| `pio run -e nodemcuv2` | Build firmware (ESP8266) | 5 min |
| `pio run -e held_esp32` | Build the Held firmware (run from PowerShell on Windows) | 2 min |
| `npm run held:sim` | Held UI from source at localhost:8080 (`-- --device <ip>` proxies to a controller) | continuous |
| `npm run held:parity` | Native C++ vs JS engine frame comparison (also part of `npm test`) | 10s |
| `npm run held:export -- <out.html>` | Export the UI as one readable self-contained page (simulator-only when hosted) | 5s |

**Always run `npm ci && npm run build` before `pio run`.** The web UI build generates
required C headers for firmware compilation.

### Running a Single Test

Tests use Node.js built-in test runner (`node:test`). The single test file is
`tools/cdata-test.js`. Run it with:

```bash
npm test                   # runs all tests via `node --test`
node --test tools/cdata-test.js  # run just that file directly
```

There are no C++ unit tests. Firmware is validated by successful compilation across
target environments. Always build after code changes: `pio run -e esp32dev`.

### Common Firmware Environments

`esp32dev`, `nodemcuv2`, `esp8266_2m`, `esp32c3dev`, `esp32s3dev_8MB_opi`, `lolin_s2_mini`

### Recovery / Troubleshooting

```bash
npm run build -- -f              # force web UI rebuild
rm -f wled00/html_*.h wled00/js_*.h && npm run build  # clean + rebuild UI
pio run --target clean           # clean PlatformIO build artifacts
rm -rf node_modules && npm ci    # reinstall Node.js deps
```

## Project Structure

```text
wled00/              # Main firmware source (C++)
  data/              # Web UI source (HTML/JS/CSS) — tabs for indentation
  html_*.h, js_*.h   # Auto-generated (NEVER edit or commit)
  src/               # Sub-modules: fonts, bundled dependencies (ArduinoJSON)
usermods/            # Community usermods (each has library.json + .cpp/.h)
platformio.ini       # Build configuration and environments
pio-scripts/         # PlatformIO build scripts (Python)
tools/               # Node.js build tools (cdata.js) and tests
docs/                # Coding convention docs
.github/workflows/   # CI/CD (GitHub Actions)
```

### Branch / Release Structure

```text
main                # Main development trunk (daily/nightly) 17.0.0-devV5. Target branch for PRs.
16_x                # maintenance for release 16.0.x
0_15_x              # maintenance (bugfixes only) for previous release 0.15.x
(tag) v0.14.4       # old version 0.14.4 (no maintenance)
(tag) v0.13.3       # old version 0.13.3 (no maintenance)
(tag) v0. ... . ... # historical versions 0.12.x and before
```

## C++ Code Style (wled00/, usermods/)

### General

- Follow the existing style in the file you are editing
- Avoid unexplained "magic numbers". For repeated numbers that have the same meaning, prefer named constants (`constexpr`) or C-style `#define` constants.
- If possible, **use `static` for local (C-style) variables and functions** to keep the global namespace clean
- When moving global items to another scope, do not leave comments such as `// lastMqttReconnectAttempt is now private to wled.cpp - see there`. These comments create technical debt for maintenance and will inevitably become out of date. As a rule of thumb: **describe the code as it is, not the code as it was**.
- Describe logic/reasoning behind non-standard patterns and complex solutions.

### Formatting
- **2-space indentation** (no tabs in C++ files)
- K&R brace style preferred (opening brace on same line)
- Single-statement `if` bodies may omit braces: `if (a == b) doStuff(a);`
- Space after keywords (`if (...)`, `for (...)`), no space before function parens (`doStuff(a)`)
- Prefer compact function parameter lists and `if` conditions; avoid putting each parameter or part of a condition on its own line. Wrap very long parameter lists when needed.
- No enforced line-length limit

### Comments
- `//` for inline (always space after), `/* */` for block comments
- Important: AI-generated source code blocks **must be mark with `// AI: below section was generated by an AI` / `// AI: end`**

### Naming Conventions
| Kind | Convention | Examples |
|---|---|---|
| Functions, variables | camelCase | `setRandomColor()`, `effectCurrent` |
| Classes, structs | PascalCase | `BusConfig`, `UsermodTemperature` |
| Macros, constants | UPPER_CASE | `WLED_MAX_USERMODS`, `FX_MODE_STATIC` |
| Private members | _camelCase | `_type`, `_bri`, `_len` |
| Enum values | PascalCase | `PinOwner::BusDigital` |

### Includes
- Include `"wled.h"` as the primary project header
- Project headers first, then platform/Arduino, then third-party
- Platform-conditional includes wrapped in `#ifdef ARDUINO_ARCH_ESP32` / `#ifdef ESP8266`

### Debug Output

- Use `DEBUG_PRINTF()` / `DEBUG_PRINTLN()` / `DEBUG_PRINT()` for developer diagnostics and debug output (compiled out unless -D WLED_DEBUG)
- wled00/wled.h defines these macros. They compile to no output when debug output is disabled, keeping normal serial interfaces clean.
- Do not use direct `Serial.print()`, `Serial.println()`, `Serial.printf()`, or `Serial.write()` calls unless there is a technical justification for not using DEBUG_... macros.

See docs/cpp.instructions.md section Error Handling for more information.

### Types and Const
- Prefer `const &` for read-only function parameters
- Mark getter/query methods `const`; use `static` for methods not accessing instance state
- Prefer `constexpr` over `#define` for compile-time constants when possible
- Use `static_assert` over `#if ... #error`
- Use `uint_fast16_t` / `uint_fast8_t` in hot-path code

### Error Handling
- **No C++ exceptions** — some builds disable them
- Use return codes (`false`, `-1`) and global flags (`errorFlag = ERR_LOW_MEM`)
- Use early returns as guard clauses: `if (!enabled || (strip.isUpdating() && (millis() - last_time < MAX_USERMOD_DELAY))) return;`
- Debug output: `DEBUG_PRINTF()` / `DEBUG_PRINTLN()` (see previous section)

### Strings and Memory
- Use `F("string")` for string constants (saves RAM on ESP8266)
- Use `PSTR()` with `DEBUG_PRINTF_P()` for format strings
- Avoid `String` in hot paths; acceptable in config/setup code
- Use `d_malloc()` (DRAM-preferred) / `p_malloc()` (PSRAM-preferred) for allocation
- No VLAs — use fixed arrays or heap allocation
- Call `reserve()` on strings/vectors to pre-allocate and avoid fragmentation

### ESP32 PSRAM guidelines

- **Check availability**: Use `psramFound() && ESP.getPsramSize() > 0`; never rely on `BOARD_HAS_PSRAM` alone.
- **DMA**: Not possible on ESP32 classic. S3 can do it in hardware (see next bullet), but descriptors must stay in DRAM and bandwidth is poor while the CPU also uses PSRAM. Keeping DMA buffers in DRAM is usually better for performance.
   - On ESP32-S3 with octal PSRAM (`CONFIG_SPIRAM_MODE_OCT`), PSRAM buffers *can* be used with DMA when `CONFIG_SOC_PSRAM_DMA_CAPABLE` is defined.
- **Cache**: PSRAM shares the CPU cache with flash (~32 KB data cache by default). Hot working sets below that run near DRAM speed. Large or random-access buffers run at raw PSRAM speed, and big sweeps can evict cached code and slow down everything.
- **Flash writes**: PSRAM is inaccessible while the flash cache is off. ISRs that run during flash writes must use only DRAM or IRAM data, and they must be declared with `IRAM_ATTR` so instructions don't  need to be fetched from flash while writing.
- **Performance**: Prefer DRAM (or IRAM) for hot-path data that is *frequently* used. Prefer PSRAM for capacity-oriented buffers where slightly slower access times can be tolerated. Small hot working sets (<~32 KB) in PSRAM can run near DRAM speed, especially when accessed sequentially.

#### Background Info:

- PSRAM access is up to 15× slower than DRAM on ESP32 (dual-SPI bus), 3–10× slower than DRAM on ESP32-S3/-S2 with quad-SPI bus. On ESP32-S3 with octal PSRAM (`CONFIG_SPIRAM_MODE_OCT`), the penalty is smaller (~2×) because the 8-line DTR bus can transfer 8 bits in parallel. On ESP32-P4 with hex PSRAM (`CONFIG_SPIRAM_MODE_HEX`), the 16-line bus runs at 200 MHz which brings it on-par with DRAM. These values are rough estimates only.
- PSRAM uses the CPU data cache (typically ~32 KB) that is also shared with flash. Small hot working sets (<~32 KB) run near DRAM speed; larger or random-access buffers fall back to raw PSRAM speed, and big PSRAM sweeps can evict cached flash code and slow down everything else. Keep hot data in DRAM and use PSRAM for large buffers that are accessed sequentially (Espressif's ESP-IDF external RAM guide, "Restrictions").
- Caching read-only files in PSRAM is usually not faster, because flash and PSRAM share the same SPI bus and often run at same speeds. Minor gains are still possible on ESP32-S3 when CONFIG_SOC_PSRAM_DMA_CAPABLE and CONFIG_SPIRAM_MODE_OCT are enabled. Better performance potential is expected on ESP32-P4 with CONFIG_SOC_MEMSPI_FLASH_PSRAM_INDEPENDENT. In special cases, file caching can still help by reducing interrupt contention (for example RMT refill pressure) and preventing visible LED flashing.
- Consider that ESP32 often crashes when the largest available DRAM chunk gets below 10 KB.

### Preprocessor / Feature Flags
- Feature toggling: `WLED_DISABLE_*` and `WLED_ENABLE_*` flags (exact names matter!)
- `WLED_DISABLE_*`: `2D`, `ADALIGHT`, `ALEXA`, `MQTT`, `OTA`, `INFRARED`, `WEBSOCKETS`, etc.
- `WLED_ENABLE_*`: `DMX`, `GIF`, `HUB75MATRIX`, `JSONLIVE`, `WEBSOCKETS`, etc.
- Platform: `ARDUINO_ARCH_ESP32`, `ESP8266`, `CONFIG_IDF_TARGET_ESP32S3`

### Math Functions
- Use `sin8_t()`, `cos8_t()` — NOT `sin8()`, `cos8()` (removed, won't compile)
- Use `sin_approx()` / `cos_approx()` instead of `sinf()` / `cosf()`
- Replace `inoise8` / `inoise16` with `perlin8` / `perlin16`

### Hot-Path Code (Pixel Pipeline)
- Use function attributes: `IRAM_ATTR`, `WLED_O2_ATTR`, `__attribute__((hot))`
- Cache class members to locals before loops
- Pre-compute invariants outside loops; use reciprocals to avoid division
- Unsigned range checks: `if ((uint_fast16_t)(pix - start) < len)`

### ESP32 Tasks
- `delay(1)` in custom FreeRTOS tasks (NOT `yield()`) — feeds IDLE watchdog
- Do not use `delay()` in effects (FX.cpp) or hot pixel path

#### ESP32 Task Synchronization

- Use FreeRTOS mutexes, semaphores or queues when true concurrent access from multiple FreeRTOS tasks is possible, and race-conditions can lead to unexpected behaviour.
- **Avoid `portENTER_CRITICAL()` / `portEXIT_CRITICAL()`**, as these functions stall the complete system and may cause LEDs flickering. Prefer FreeRTOS mutexes, semaphores or queues.
- Don't use `portMAX_DELAY` when waiting to acquire a mutex - this can lock the task indefinitely. Find a reasonable max waiting time, and handle mutex timeouts gracefully.
- **Important**: Not every shared resource needs a mutex. Some synchronization is guaranteed by the overall control flow, for example when function calls are sequenced within the same loop iteration.
- Consider using `std::atomic` or RAII scoped guards as alternatives to mutexes, semaphores or queues.

## Web UI Code Style (wled00/data/)

- **Tab indentation** for HTML, JS, and CSS
- camelCase for JS functions/variables
- Reuse helpers from `common.js` — do not duplicate utilities
- After editing, run `npm run build` to regenerate headers
- **Never edit** `wled00/html_*.h` or `wled00/js_*.h` directly

## Usermods
 
### Source Code Location

* **In-Tree Usermods** live in `usermods/<name>/` with a `.cpp`, optional `.h`, `library.json`, and `readme.md`.  An example is in `usermods/EXAMPLE`
* **Out-Of-Tree Usermods** live in a separate public repository. They use the same pattern as in-tree usermods.

* [Official out-of-tree usermods list](https://kno.wled.ge/advanced/community-usermods/#index)
* [Writing an out-of-tree usermod](https://kno.wled.ge/advanced/custom-features/#writing-a-usermod)

### Usermod Pattern

```cpp
class MyUsermod : public Usermod {
  private:
    bool enabled = false;
    static const char _name[];
  public:
    void setup() override { /* ... */ }                          // runs once at start-up
    void loop() override { /* ... */ }                           // runs once per main loop iteration
    void addToConfig(JsonObject& root) override { /* ... */ }    // create/add persistent settings (usermod settings)
    bool readFromConfig(JsonObject& root) override { /* ... */ } // read from persistent settings (usermod settings UI)
    uint16_t getId() override { return USERMOD_ID_MYMOD; }
    void addToJsonInfo(JsonObject& root) override { /* ... */ }  // Add custom items to the "info" page and to /json/info
    void appendConfigData() override { /* ... */ }               // Customize the settings page: dropdowns, checkboxes, extra text, etc. Buffer size is limited!
};
const char MyUsermod::_name[] PROGMEM = "MyUsermod";
static MyUsermod myUsermod;
REGISTER_USERMOD(myUsermod);
```

refer to detailed examples in `usermods/EXAMPLE/`, `usermods/user_fx/` and [in the user documentation for custom features](https://kno.wled.ge/advanced/custom-features/).

- Activate via `custom_usermods = ` in platformio build config. The `usermod_v2_` prefix or `_v2` suffix can be omitted.
- Base new usermods on `usermods/EXAMPLE/` (never edit the example directly)
- Store repeated strings as `static const char[] PROGMEM`
- Add usermod IDs to `wled00/const.h` **only when a unique ID is required** (see below)

### Pin ownership via `pinManager`
- Before performing any operation on I/O pins, the usermod must allocate its pins from the `pinManager`.
- I/O pins are allocated via `PinManager::allocatePin(byte gpio, bool output, PinOwner tag)` (or `PinManager::allocateMultiplePins()`), and returned via `PinManager::deallocatePin()` when re-configuring pin numbers.
- If the usermod does not register a unique usermod ID, use `PinOwner::UM_Unspecified` as PinOwner tag for generic pin use.
- `PinManager::allocatePin()` will return an error code in case that a pin is already assigned to another WLED function.
- You can use `-1` = `255` for unconfigured / unassigned pin functions.
- Check for valid pin numbers with `PinManager::isPinOk(byte gpio, bool output)`; this function knows which GPI(O) pins are possible on your specific MCU.

#### I2C (Wire) and SPI busses

- PIN numbers for I2C and SPI busses are configured globally in the 'Global I2C & SPI' part of the usermod settings page.
- Usually, a usermod will not start the I2C or SPI units explicitly, since they are already initialised when WLED starts.
- Don't call `Wire.begin()` in your `Usermod::setup()`, as this may break other usermods that need the I2C bus.
- Don't allocate `Wire` I2C pins to the usermod - it would cause a pin conflict, as global I2C pins are already allocated to `PinOwner::HW_I2C`.
- Use this pattern to check for an incomplete I2C setup: `if (i2c_scl<0 || i2c_sda<0) {enabled = false; return;}`.

### Usermod IDs

A unique ID (registered in `wled00/const.h` and overriding `getId()`) is **only required** when a usermod needs one or more of the following:

1. **Inter-usermod communication** — another usermod or an FX effect calls `UsermodManager::lookup(mod_id)` or `UsermodManager::getUMData(..., mod_id)` to find or request data from this specific usermod.
2. **Pin ownership via `PinManager`** — the usermod allocates GPIO pins through `pinManager`. Pin ownership is tracked by `PinOwner` enum values that map directly to `USERMOD_ID_*` constants (see `wled00/pin_manager.h`). This prevents pin-conflict bugs.
3. **Identification in JSON info** — `UsermodManager::addToJsonInfo` emits each mod's ID into the `"um"` array; a unique ID makes the mod identifiable in that output.

If none of the above apply, the usermod may omit `getId()` (or return the default `USERMOD_ID_UNSPECIFIED`) and does **not** need an entry in `const.h`.

### Usermod `loop()`

- Called once per main loop iteration. Usermods should simply `return` when `!enabled`.
- Avoid blocking calls or `delay()` in the usermod `loop()` - delays block the WLED main loop and lead to effect stuttering.
- Frequency of Usermod `loop()` calls varies with system load:
    * up to 2000 times/sec with few LEDs and little background activity,
    * between 20 and 300 times/second during high workload from effects and other usermods,
    * (worst case) down to 1-3 times/sec during FS activity or when serving lots of network API requests.

### See Also
* https://kno.wled.ge/advanced/custom-features/#usermods
* https://kno.wled.ge/advanced/community-usermods/#index
* generic instructions for Debug Output, as per previous section in this file.