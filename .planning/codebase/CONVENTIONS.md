# Coding Conventions

**Analysis Date:** 2026-04-13

## Overview

This is a single-file ESPHome YAML project (`mini-rotary-screen.yaml`) targeting an ESP32 with a ST7789V SPI display. All configuration, logic, and inline C++ live in one file. Conventions are informal but consistent throughout.

## Naming Conventions

**ESPHome component IDs (YAML `id:` fields):**
- `snake_case` for all IDs
- Examples: `backlight`, `bl_output`, `wake_screen`, `selected_control`, `editing_mode`, `sleep_ticks`, `enc_last_dir`, `enc_last_ms`

**Global variables:**
- `snake_case`, named after the domain concept they represent
- Boolean flags use `_active`, `_mode`, `_sleeping` suffixes: `heating_active`, `editing_mode`, `was_sleeping`
- Counters use `_ticks` suffix: `sleep_ticks`
- Encoder state uses `enc_` prefix: `enc_last_dir`, `enc_last_ms`

**Substitution variables:**
- `SCREAMING_SNAKE_CASE` for substitution keys
- Examples: `DIM_TIMEOUT_TICKS`, `SLEEP_TIMEOUT_TICKS`

**Font IDs:**
- `font_` prefix followed by size descriptor: `font_tiny`, `font_small`, `font_large`, `font_xl`

**Named entities (Home Assistant `name:` fields):**
- Title case with spaces: `"Display Backlight"`, `"Rotary Encoder"`, `"Encoder Push"`, `"Key0"`

**MQTT topics:**
- Lowercase with `/` hierarchy: `home/heater/current_temp`, `home/hotwater/temp`
- Set commands appended with `/set`: `home/heater/target_temp/set`

**Display string labels:**
- ALL CAPS for section headers and static labels: `"HEATER CONTROLLER"`, `"TARGET"`, `"OUTSIDE"`, `"HOT WATER TEMP"`, `"HEATING"`, `"IDLE"`
- Lowercase for instructional hints: `"push to edit"`, `"rotate to switch screen"`

## Code Organization

The YAML file is organized into named sections separated by `# ── Section Name ──` banner comments. Order:

1. `esphome:` - Device identity and boot action
2. `substitutions:` - Compile-time constants
3. `esp32:` - Board and framework config
4. `logger:` / `api:` / `ota:` / `wifi:` / `captive_portal:` - Infrastructure
5. `mqtt:` - Broker config and all topic subscriptions
6. `spi:` - Bus config
7. `output:` / `light:` - Backlight hardware and ESPHome light entity
8. `font:` - All font definitions
9. `globals:` - All global variables
10. `interval:` - Sleep/dim tick timer
11. `script:` - Reusable action blocks (`wake_screen`)
12. `display:` - Full display rendering lambda
13. `binary_sensor:` - Button inputs
14. `sensor:` - Rotary encoder

Display rendering is entirely contained in a single `lambda:` block inside `display:`. Screen content is selected with `if (id(selected_control) == N)` branches, not separate pages or components.

## Lambda/C++ Style

**Formatting:**
- Lambda bodies use `|-` block scalar (literal block, strip trailing newlines)
- Indentation: 2-space YAML nesting; C++ code inside uses standard 2-space indentation
- Opening braces on same line as control statement (K&R style)

**Variable declarations:**
- Local buffer for `snprintf`: always `char buf[16];`
- Loop variables: `int s`, `int i` (short names for tight loops)
- Local coordinate aliases: `int cx = 120, cy = 148, R = 90;`

**Type usage:**
- Float literals use `f` suffix: `0.5f`, `1.0f`, `5.0f`, `35.0f`
- Integer casts explicit when mixing: `(int)(R * cosf(a))`, `(uint32_t)id(enc_last_ms)`
- `std::min` / `std::max` used for clamping (not ternary chains)
- `snprintf` for all formatted number output (not `sprintf`)

**Color literals:**
- Constructed inline as `Color(R, G, B)` with hex byte literals: `Color(0xE0, 0x55, 0x20)`
- Named colors assigned to local `Color` variables when reused within a block: `Color arc_col`, `Color dot_col`, `Color hw_temp_col`

**Control flow in display lambda:**
- Early `return;` used to short-circuit sleep screen rendering before drawing the active UI
- Screen content split by `if / else if / else` on `id(selected_control)`

**Bounce filtering (encoder):**
- Manual timestamp comparison using `millis()` stored in global `enc_last_ms` (cast to `int`)
- Direction stored in `enc_last_dir` (1, -1, or 0)
- Bounce window: 80ms

**MQTT publish in lambda:**
- `payload: !lambda` tag used to return `std::string` from formatted buffer

**Logging:**
- `ESP_LOGI("main", ...)` used for state change logging (e.g., toggling `editing_mode`)

## Substitution Variables

Substitutions are defined at the top of the file under `substitutions:` and referenced with `$VARIABLE_NAME` syntax in lambda bodies.

```yaml
substitutions:
  DIM_TIMEOUT_TICKS:   "150"   # 150 × 200ms = 30s
  SLEEP_TIMEOUT_TICKS: "300"   # 300 × 200ms = 60s
```

Inline comments document the derived real-world duration (ticks × interval). Used in:
- `interval:` lambda to decide dim vs. sleep state
- `script:` lambda to detect if screen was sleeping
- `display:` lambda for the sleep screen branch

## Documentation Style

**Section banners:**
- Each logical section opens with a `# ── Section Name ──────...` comment line using em-dash box-drawing
- These appear as YAML comments immediately before the component key

**Inline comments in lambdas:**
- `// ── Sub-section label ──` style used to break up the display lambda into named visual zones (e.g., `// ── Top bar ──`, `// ── Screen 0: Heater control ──`)
- Brief explanatory notes appear on the same line or preceding line for non-obvious values

**Substitution comments:**
- Inline `# N × Xms = Ys` comments document the timing math next to raw tick counts

**README:**
- Minimal (single heading, no body content)

**No JSDoc/docstrings** - not applicable to YAML/ESPHome context.
