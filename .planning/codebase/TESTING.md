# Testing Patterns

**Analysis Date:** 2026-04-13

## Overview

This is an ESPHome firmware project with no automated test suite. There is no test runner, no test files, and no CI pipeline. All verification is done by flashing to physical hardware and observing behavior. This is standard practice for ESPHome-based projects.

## Testing Approach

**Hardware-in-the-loop only.**

The only way to verify behavior is to:
1. Run `esphome compile mini-rotary-screen.yaml` to compile the firmware
2. Flash to the ESP32 device via `esphome run mini-rotary-screen.yaml` (USB) or OTA
3. Observe the physical display and test input devices manually

**OTA (Over-The-Air) updates** are configured via `ota:` with a password stored in ESPHome secrets. This enables iterative testing without USB access after initial flash.

**Fallback AP** is configured via `wifi:` with an `ap:` block (SSID: `Testroutaryencoder`, hardcoded password). This ensures the device remains accessible if WiFi credentials fail, which is a safety net during development.

**Captive portal** is enabled, providing a web UI for WiFi reconfiguration if the device cannot connect to the primary network.

## Validation

**ESPHome config validation:**
- Run `esphome config mini-rotary-screen.yaml` to validate YAML structure and component compatibility before flashing
- ESPHome performs type checking, pin conflict detection, and component dependency resolution at compile time
- Compilation errors (C++ from lambda blocks) surface during `esphome compile`

**Secrets validation:**
- `!secret` references in `mini-rotary-screen.yaml` require a `secrets.yaml` file in the same directory or ESPHome config directory
- Required secrets: `ota_rotaryencoder_1_password`, `wifi_ssid`, `wifi_password`, `mqtt_broker`, `mqtt_user`, `mqtt_password`
- Missing secrets cause a compile-time error with the secret key name

**Lambda C++ validation:**
- All `lambda:` blocks are compiled as C++ by the ESPHome build system (esp-idf framework)
- Type mismatches, undefined identifiers, and syntax errors surface during `esphome compile`
- The `id()` macro references must match defined component IDs; mismatches are compile errors

**No runtime assertion framework** is present. No `assert()` calls or custom error handlers in the lambda code.

## Debug & Logging

**ESPHome logger component** is enabled with default settings (no explicit `level:` set, defaults to `DEBUG`):
```yaml
logger:
```

**Serial output** is available via USB UART during development for log output.

**Explicit log statements in lambdas:**
- `ESP_LOGI("main", "editing_mode = %s", id(editing_mode) ? "true" : "false")` — logs on every encoder button press when toggling edit mode
- Tag used: `"main"` (single tag, no per-subsystem tagging)
- Only one explicit log call exists in the entire file; all other state changes are silent

**ESPHome Home Assistant API** is configured with encryption. When connected to HA, entity states (`backlight`, `Rotary Encoder`, `Encoder Push`, `Key0`) are visible in the HA device dashboard, providing remote state inspection without serial access.

**MQTT broker** receives published values:
- `home/heater/target_temp/set` is published on encoder button press (save action), providing an observable event for verifying the save flow

**Display as debug output:**
- The physical display itself serves as the primary visual feedback mechanism during testing
- The status dot (top-right corner) reflects `heating_active` state (orange=heating, green=idle)
- `editing_mode` changes the arc color (orange=editing, blue=normal) and shows hint text

**No log levels configured per-component.** No `logger: logs:` section overrides exist.

## Known Test Scenarios

Based on the implemented feature set in `mini-rotary-screen.yaml`, the following scenarios represent the functional scope that should be verified on hardware:

**Boot behavior:**
- Device boots, backlight turns on at 100% brightness
- Display shows Screen 0 (Heater control) by default

**Screen navigation:**
- Rotate encoder clockwise: advances `selected_control` (0 → 1 → 2 → wraps to 0)
- Rotate encoder anticlockwise: decrements `selected_control` (wraps around)
- Screen indicator dots at y=32 update to reflect current screen

**Edit mode (Screen 0 only):**
- Press encoder button: toggles `editing_mode`; arc color changes orange; hint text updates
- Rotate CW in edit mode: increases `target_temp` by 0.5°C, clamped at 30.0°C
- Rotate CCW in edit mode: decreases `target_temp` by 0.5°C, clamped at 5.0°C
- Press encoder button again (save): publishes to `home/heater/target_temp/set` via MQTT

**Sleep/dim behavior:**
- After 30s inactivity (150 ticks × 200ms): backlight dims to 15%
- After 60s inactivity (300 ticks × 200ms): display shows sleep screen (near-black with faint text)
- Any encoder rotation or button press: resets sleep timer, restores brightness
- Waking from sleep does not trigger edit/navigation actions (guarded by `was_sleeping` flag)

**MQTT data display:**
- Publishing to `home/heater/current_temp` updates current temp on Screen 0
- Publishing to `home/heater/heating_active` with `"true"` or `"1"` updates heating indicator color and status text
- Publishing to `home/hotwater/temp` updates Screen 2 hot water temperature with color thresholds (≥55°C orange, ≥40°C yellow, else blue)
- Publishing to `home/hotwater/minutes` updates available minutes with color thresholds (≥30 green, ≥10 yellow, else red)
- Publishing to `home/temperatures/outside` and `home/temperatures/inside` updates Screen 1

**Encoder bounce filtering:**
- Rapid direction reversals within 80ms are suppressed by the `enc_last_dir`/`enc_last_ms` guard

**Fallback AP:**
- If WiFi credentials fail, device creates AP `Testroutaryencoder` for reconfiguration via captive portal

**Key0 button (GPIO26):**
- Press wakes screen (resets sleep timer) but has no other action
