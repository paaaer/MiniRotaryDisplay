# Architecture

**Analysis Date:** 2026-04-13

## Overview

Single-file ESPHome YAML firmware for an ESP32-based mini display device with a rotary encoder. The device acts as a local home-automation display node: it subscribes to MQTT topics for sensor data, renders that data across three pages on a 240×320 ST7789V LCD, and accepts physical input via a rotary encoder (rotate + push) to navigate pages and edit a target temperature setpoint which it publishes back to MQTT.

## System Architecture

```
[MQTT Broker] ─── subscribe ──► [ESP32 / ESPHome]
                                        │
                  publish ◄──────────── │  (target_temp/set on save)
                                        │
                              ┌─────────┴──────────┐
                              │  Globals (RAM state)│
                              └─────────┬──────────┘
                                        │
                              ┌─────────▼──────────┐
                              │  Display lambda     │
                              │  (renders 3 pages)  │
                              └─────────────────────┘
                                        ▲
                              ┌─────────┴──────────┐
                              │  Input handlers     │
                              │  rotary + buttons   │
                              └─────────────────────┘
```

**Runtime:** ESP-IDF framework on `esp32dev` board.
**Connectivity:** Wi-Fi (with captive-portal fallback AP) + MQTT broker. Home Assistant API with encryption also enabled.
**OTA:** ESPHome OTA with password from secrets.

## Component Model

| Component | ESPHome Platform | GPIO | Role |
|---|---|---|---|
| LCD display | `mipi_spi` / ST7789V | SPI: CLK=18, MOSI=23, CS=5, DC=4, RST=2 | Renders all UI |
| Backlight | `ledc` PWM output + `monochromatic` light | GPIO16 | Brightness/dimming control |
| Rotary encoder | `rotary_encoder` sensor | A=GPIO32, B=GPIO33 | Page navigation and value editing |
| Encoder push button | `gpio` binary_sensor | GPIO25 (PULLUP, inverted) | Toggle edit mode / save setpoint |
| Key0 button | `gpio` binary_sensor | GPIO26 (PULLUP, inverted) | Wake screen only |
| MQTT | `mqtt` component | n/a | Data ingestion and setpoint publishing |

**Fonts:** Roboto from Google Fonts, four sizes: `font_tiny` (10px), `font_small` (14px), `font_large` (26px), `font_xl` (38px).

## State Management

All mutable state lives in ESPHome `globals` (C++ variables in firmware RAM). There is no persistent storage — values reset to `initial_value` on reboot, then are refreshed by incoming MQTT messages.

**Navigation state:**
- `selected_control` (int, 0–2) — which page is active
- `num_controls` (int, 3) — total page count (constant in practice)
- `editing_mode` (bool) — whether encoder rotation edits a value vs. switches pages

**Sensor data (MQTT-fed):**
- `current_temp` (float) — heater current temperature
- `heating_active` (bool) — heater on/off status
- `hw_temp` (float) — hot water temperature
- `hw_minutes` (float) — hot water available minutes
- `outside_temp` (float) — outdoor temperature
- `inside_temp` (float) — indoor temperature

**User setpoint:**
- `target_temp` (float, 5.0–30.0, step 0.5) — editable on Screen 0; published to MQTT on save

**Power/sleep state:**
- `sleep_ticks` (int) — incremented every 200 ms by interval timer; reset to 0 on any input
- `was_sleeping` (bool) — set by `wake_screen` script to flag that the first event after sleep should be consumed (wake-only, no action)

**Encoder debounce state:**
- `enc_last_dir` (int: 1=CW, -1=CCW, 0=none) — last rotation direction
- `enc_last_ms` (int) — timestamp of last rotation event (milliseconds)

**Substitution constants** (compile-time):
- `DIM_TIMEOUT_TICKS` = 150 → 30 s before dimming
- `SLEEP_TIMEOUT_TICKS` = 300 → 60 s before sleep screen

## UI/Display Logic

The display lambda runs every **50 ms** (20 fps). It reads globals and renders one of three exclusive states:

**Sleep screen** (`sleep_ticks >= SLEEP_TIMEOUT_TICKS`):
- Near-black fill
- Faint "HEATER CONTROLLER" and "-- push to wake --" text
- Returns early; no other UI drawn

**Active screen** (all other ticks):
1. **Top bar** (y=0–26): Title "HEATER CONTROLLER" + heating-status dot (orange=heating, green=idle)
2. **Page indicator dots** (y=32): Three dots, active page highlighted blue
3. **Page content** (dispatched by `selected_control`):
   - `0` — Heater Control: circular arc gauge (5–35 °C range), target temp large, current temp + HEATING/IDLE status, edit-mode hint text
   - `1` — Temperature Overview: two cards (Outside / Inside) with xl temperatures
   - `2` — Hot Water: two cards (temp with color-coded threshold, available minutes with color-coded threshold)
4. **Bottom bar** (y=308–320): Context hint text

**Dimming** (managed by 200 ms interval, not display lambda):
- `sleep_ticks >= DIM_TIMEOUT_TICKS` and `< SLEEP_TIMEOUT_TICKS`: backlight set to 15% over 1 s
- `sleep_ticks < DIM_TIMEOUT_TICKS`: backlight set to 100% instantly

**Edit mode visual feedback** (Screen 0 only):
- Arc color changes from blue to amber when `editing_mode = true`
- Target temp text color changes to amber
- Hint text changes from "push to edit" to "< rotate  push=save >"

## Control Flow

**Rotary encoder rotation (CW / CCW):**
```
on_clockwise / on_anticlockwise
  │
  ├─ script.execute: wake_screen   (always)
  │     └─ resets sleep_ticks, sets was_sleeping flag, restores brightness
  │
  ├─ Debounce check: if direction reversed within 80 ms → discard event
  │
  ├─ if was_sleeping → consume event (no further action)
  │
  ├─ if editing_mode == true
  │     └─ Screen 0: increment/decrement target_temp by 0.5 (clamped 5–30)
  │
  └─ if editing_mode == false
        └─ selected_control = (selected_control ± 1) % num_controls
```

**Encoder push button:**
```
on_press
  │
  ├─ script.execute: wake_screen
  │
  ├─ if was_sleeping → return (wake only)
  │
  ├─ if selected_control != 0 → return (push only works on Screen 0)
  │
  ├─ editing_mode = !editing_mode   (toggle)
  │
  └─ if editing_mode just became false (i.e. save action)
        └─ mqtt.publish: "home/heater/target_temp/set" with target_temp value
```

**Key0 button:**
```
on_press
  └─ script.execute: wake_screen   (wake only, no other effect)
```

**MQTT inbound (6 topics):**
Each topic handler directly writes the corresponding global via a lambda `atof()` or boolean parse. No buffering or validation layer.
