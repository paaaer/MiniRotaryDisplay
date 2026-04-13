# INTEGRATIONS.md

**Analysis Date:** 2026-04-13

## Overview

The device integrates with a home automation ecosystem via two parallel channels: the ESPHome Native API (for Home Assistant control plane) and MQTT (for real-time sensor data). Hardware integrations include a rotary encoder, two push buttons, an SPI TFT display, and a PWM-controlled backlight.

## External Services

**WiFi Network:**
- Mode: Infrastructure (station) with fallback AP
- Credentials: `!secret wifi_ssid`, `!secret wifi_password`
- Fallback AP SSID: `Testroutaryencoder` (hardcoded password in config)
- Captive portal enabled on fallback AP for re-configuration

**Home Assistant (ESPHome Native API):**
- Protocol: ESPHome binary protocol over TCP
- Encryption: AES-128, key defined inline in `mini-rotary-screen.yaml` (line 24)
- Exposes: backlight light entity (`Display Backlight`), encoder sensor (`Rotary Encoder`), button sensors (`Key0`, `Encoder Push`)
- Direction: Home Assistant can observe entity states and control backlight brightness

**MQTT Broker:**
- Credentials: `!secret mqtt_broker`, `!secret mqtt_user`, `!secret mqtt_password`
- Direction: Inbound only (device subscribes, reads data, never publishes sensor readings except one outbound command)
- Subscribed topics (all inbound):

| Topic | Variable Updated | Type |
|---|---|---|
| `home/heater/current_temp` | `current_temp` (float) | Temperature in degrees C |
| `home/heater/heating_active` | `heating_active` (bool) | `"true"` / `"1"` = active |
| `home/hotwater/temp` | `hw_temp` (float) | Temperature in degrees C |
| `home/hotwater/minutes` | `hw_minutes` (float) | Minutes of hot water available |
| `home/temperatures/outside` | `outside_temp` (float) | Temperature in degrees C |
| `home/temperatures/inside` | `inside_temp` (float) | Temperature in degrees C |

- Published topic (outbound, one topic only):

| Topic | Trigger | Payload |
|---|---|---|
| `home/heater/target_temp/set` | Encoder push button (save action on screen 0) | Float formatted as `"%.1f"` string |

**OTA Updates:**
- Platform: ESPHome OTA
- Transport: WiFi (same network as operational use)
- Authentication: password via `!secret ota_rotaryencoder_1_password`

## Hardware Integrations

**ST7789V TFT Display (240x320px):**
- Bus: SPI (VSPI) at 20MHz
- CS: GPIO5, DC: GPIO4, RESET: GPIO2
- Color order: BGR, colors inverted
- Rotation: 0 degrees (portrait)
- Refresh rate: 50ms interval (20 fps)
- Content: 3 screens navigated by rotary encoder rotation

**Backlight (PWM LED):**
- Output: LEDC peripheral on GPIO16, 1kHz
- ESPHome light entity: `backlight` (monochromatic)
- Transition length: 400ms default, 1000ms on dim, 200ms on wake, 0ms on brightness restore
- Brightness states:
  - Full (100%) — active use
  - Dim (15%) — after 30s inactivity
  - Screen off (display renders near-black sleep screen) — after 60s inactivity

**Rotary Encoder:**
- Pins: A=GPIO32, B=GPIO33 (both INPUT_PULLUP)
- ESPHome platform: `rotary_encoder`, resolution 1
- Debounce: software bounce filter — direction reversal within 80ms is ignored
- Clockwise action: wake screen OR increment `target_temp` (edit mode) OR advance `selected_control`
- Anticlockwise action: wake screen OR decrement `target_temp` (edit mode) OR retreat `selected_control`
- Target temp range: 5.0–30.0 degrees C, step 0.5

**Push Buttons:**
- Key0 (GPIO26, INPUT_PULLUP, inverted, 20ms debounce): wakes screen only
- Encoder push (GPIO25, INPUT_PULLUP, inverted, 25ms debounce): wakes screen AND toggles `editing_mode` on screen 0 AND publishes target temp to MQTT on save

## Data Flows

**Inbound sensor data flow:**
```
MQTT Broker
  └─► ESPHome mqtt component (topic subscription)
        └─► lambda: parses string payload → updates global float/bool variable
              └─► display lambda reads globals every 50ms → renders to TFT
```

**User input flow (screen navigation):**
```
Rotary encoder rotation (GPIO32/33)
  └─► rotary_encoder sensor (on_clockwise / on_anticlockwise)
        └─► script: wake_screen (resets sleep_ticks, restores brightness)
              └─► lambda: updates selected_control (0, 1, 2) mod num_controls
                    └─► display lambda reads selected_control → renders correct screen
```

**User input flow (temperature editing):**
```
Encoder push button (GPIO25)
  └─► binary_sensor on_press
        └─► script: wake_screen
              └─► lambda: toggles editing_mode (if selected_control == 0)
                    └─► Rotary CW/CCW adjusts target_temp ±0.5 while editing_mode
                          └─► Second push: editing_mode=false → mqtt.publish target_temp
```

**Sleep/dim state machine flow:**
```
interval (every 200ms)
  └─► lambda: increments sleep_ticks
        ├─► sleep_ticks >= 150 (30s): dim backlight to 15%
        └─► sleep_ticks >= 300 (60s): display renders dark sleep screen

Any input (encoder rotate or push, Key0 push)
  └─► script: wake_screen
        └─► resets sleep_ticks = 0, restores backlight to 100%
```

## APIs & Protocols

**ESPHome Native API:**
- Binary framing over TCP, port 6053 (default)
- AES-128 encryption
- Used by: Home Assistant ESPHome integration
- Entities exposed: `Display Backlight` (light), `Rotary Encoder` (sensor), `Key0` (binary sensor), `Encoder Push` (binary sensor)

**MQTT:**
- Standard MQTT protocol (port determined by broker config in secrets)
- QoS: default (ESPHome default is QoS 0)
- Payload format: plain UTF-8 strings (numeric floats as ASCII, booleans as `"true"`/`"1"`)
- No retained messages configured explicitly

**SPI (display bus):**
- VSPI peripheral
- CLK: GPIO18, MOSI: GPIO23
- No MISO (display is write-only)
- 20MHz data rate to ST7789V

**LEDC (PWM):**
- ESP32 hardware PWM peripheral
- 1kHz carrier, 8-bit duty cycle resolution (ESPHome default)
- Controls backlight brightness via monochromatic light abstraction

---

*Integration audit: 2026-04-13*
