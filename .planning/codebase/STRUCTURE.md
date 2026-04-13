# Codebase Structure

**Analysis Date:** 2026-04-13

## Overview

Minimal single-device ESPHome project. All firmware logic lives in one YAML file. Supporting files are documentation only. There is no build output, no package manifest, and no separate source directory — ESPHome compiles the YAML directly.

## Directory Layout

```
MiniRotaryDisplay/
├── mini-rotary-screen.yaml   # All firmware config and logic
├── README.md                 # Project description (one line)
├── Docs/                     # Reference images only
│   ├── dev_kit_pinout.jpg    # ESP32 dev kit pinout diagram
│   ├── nedladdning.png       # Download/setup screenshot
│   └── S436247eb839a4920abb1f491f42d6afdd.avif  # Product photo
└── .planning/                # GSD planning documents (not firmware)
    └── codebase/
        ├── ARCHITECTURE.md
        └── STRUCTURE.md
```

No `secrets.yaml` is committed (referenced via `!secret`). It must exist alongside `mini-rotary-screen.yaml` at compile time.

## Config Organization

`mini-rotary-screen.yaml` is organized top-to-bottom in the following sections (annotated with line ranges):

| Section | Lines | Purpose |
|---|---|---|
| `esphome` + `on_boot` | 1–9 | Device name, boot action (backlight on at 100%) |
| `substitutions` | 11–13 | Compile-time constants for dim/sleep timeouts |
| `esp32` | 15–18 | Board and framework (ESP-IDF) |
| `logger` | 20 | Serial logging, default config |
| `api` | 22–24 | Home Assistant API with encryption key |
| `ota` | 26–28 | OTA update with secret password |
| `wifi` + `captive_portal` | 30–37 | Network with fallback AP |
| `mqtt` + `on_message` | 39–61 | Broker connection and 6 inbound topic handlers |
| `spi` | 63–66 | VSPI bus pin definitions |
| `output` + `light` | 68–81 | LEDC PWM output and monochromatic backlight |
| `font` | 83–96 | Four Roboto font sizes |
| `globals` | 98–142 | All mutable firmware state |
| `interval` | 144–161 | 200 ms tick: sleep/dim state machine |
| `script` | 163–175 | `wake_screen` script |
| `display` | 177–356 | ST7789V display, 50 ms render loop, full UI lambda |
| `binary_sensor` | 358–398 | Key0 and Encoder push button handlers |
| `sensor` | 400–450 | Rotary encoder with CW/CCW handlers |

## Key Globals

All globals are declared in `mini-rotary-screen.yaml` lines 99–142.

**Navigation:**

| ID | Type | Initial | Purpose |
|---|---|---|---|
| `selected_control` | int | 0 | Active page index (0=Heater, 1=Temperatures, 2=Hot Water) |
| `num_controls` | int | 3 | Total number of pages (effectively a constant) |
| `editing_mode` | bool | false | When true, encoder rotation edits `target_temp` instead of switching pages |

**Sensor data (MQTT-sourced):**

| ID | Type | Initial | MQTT Topic |
|---|---|---|---|
| `current_temp` | float | 0.0 | `home/heater/current_temp` |
| `heating_active` | bool | false | `home/heater/heating_active` |
| `hw_temp` | float | 58.0 | `home/hotwater/temp` |
| `hw_minutes` | float | 45.0 | `home/hotwater/minutes` |
| `outside_temp` | float | 5.0 | `home/temperatures/outside` |
| `inside_temp` | float | 21.0 | `home/temperatures/inside` |

**User setpoint:**

| ID | Type | Initial | Range | Published to |
|---|---|---|---|---|
| `target_temp` | float | 18.0 | 5.0–30.0, step 0.5 | `home/heater/target_temp/set` |

**Power management:**

| ID | Type | Initial | Purpose |
|---|---|---|---|
| `sleep_ticks` | int | 0 | Counts 200 ms ticks since last activity; drives dim/sleep logic |
| `was_sleeping` | bool | false | True if device was dimmed/sleeping before last input; used to suppress first-event action |

**Encoder debounce:**

| ID | Type | Initial | Purpose |
|---|---|---|---|
| `enc_last_dir` | int | 0 | Direction of last rotation (1=CW, -1=CCW, 0=none) |
| `enc_last_ms` | int | 0 | `millis()` timestamp of last rotation; events reversing direction within 80 ms are discarded |

## Key Automations & Scripts

### `interval` — Sleep/Dim Tick (line 144)

Fires every **200 ms**. Increments `sleep_ticks` then evaluates two thresholds:
- `>= DIM_TIMEOUT_TICKS` (150 ticks = 30 s): sets backlight to 15% over 1 s transition
- `>= SLEEP_TIMEOUT_TICKS` (300 ticks = 60 s): display lambda takes over (renders sleep screen); backlight stays dim

### `script: wake_screen` (line 165)

Executed on every physical input (encoder rotate, encoder push, Key0 push).
1. Records whether `sleep_ticks` was above the dim threshold into `was_sleeping`
2. Resets `sleep_ticks` to 0
3. If `was_sleeping` was true: restores backlight to 100% over 200 ms transition

Callers check `was_sleeping` after calling this script to decide whether to suppress the event's primary action (first interaction only wakes, does not navigate or edit).

### `display` lambda — Render Loop (line 192)

Runs every **50 ms**. Single monolithic lambda that:
- Branches on `sleep_ticks >= SLEEP_TIMEOUT_TICKS` for sleep screen early return
- Otherwise renders: top bar, page-indicator dots, then dispatches to one of three page blocks based on `selected_control`

### `binary_sensor: Encoder Push` — on_press (line 369)

Sequence:
1. `wake_screen`
2. If `was_sleeping` → return
3. If `selected_control != 0` → return
4. Toggle `editing_mode`
5. If `editing_mode` just became false → `mqtt.publish` `target_temp` to `home/heater/target_temp/set`

### `sensor: Rotary Encoder` — on_clockwise / on_anticlockwise (line 411)

Each direction handler:
1. `wake_screen`
2. Debounce: discard if direction reversed within 80 ms
3. If `was_sleeping` → return
4. If `editing_mode`: adjust `target_temp` ±0.5 (clamped)
5. Else: advance `selected_control` ±1 (wrapping mod `num_controls`)

### `on_boot` (line 4)

Single action at boot (priority -100, runs after all components are ready): turns backlight on at 100% brightness.

## Secrets Required

The following keys must exist in a `secrets.yaml` file co-located with the YAML at compile/flash time:

- `ota_rotaryencoder_1_password`
- `wifi_ssid`
- `wifi_password`
- `mqtt_broker`
- `mqtt_user`
- `mqtt_password`
