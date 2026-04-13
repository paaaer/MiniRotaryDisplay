# STACK.md

**Analysis Date:** 2026-04-13

## Overview

This project is an ESPHome YAML firmware configuration for an ESP32-based rotary encoder display unit. The entire firmware is defined in a single declarative YAML file compiled and flashed via the ESPHome toolchain. There is no application-layer code in a traditional language — all logic is expressed as ESPHome lambdas (inline C++) within the YAML config.

## Primary Language & Framework

**Framework:**
- ESPHome — declarative IoT firmware platform for ESP32/ESP8266
- Config file: `mini-rotary-screen.yaml`

**Language:**
- YAML — primary config syntax for all ESPHome component declarations
- C++ (inline lambdas) — used inside `lambda:` blocks for display rendering, encoder debounce logic, MQTT message parsing, and state management

**MCU Platform:**
- ESP32 (`board: esp32dev`)
- Framework: `esp-idf` (Espressif IoT Development Framework, not Arduino)

## Key Dependencies

**ESPHome Built-in Components (no external packages, all resolved by ESPHome compiler):**

| Component | Purpose |
|---|---|
| `esphome` core | Device identity, boot hooks |
| `esp32` / `esp-idf` | MCU target and low-level framework |
| `wifi` | Network connectivity |
| `api` (ESPHome Native API) | Home Assistant integration with AES encryption |
| `ota` (ESPHome platform) | Over-the-air firmware updates |
| `mqtt` | MQTT broker subscription and publishing |
| `captive_portal` | Fallback WiFi config portal via AP mode |
| `logger` | Serial/UART debug logging |
| `spi` | SPI bus driver for display (VSPI: CLK=GPIO18, MOSI=GPIO23) |
| `display` / `mipi_spi` / `ST7789V` | TFT display driver |
| `output` / `ledc` | PWM output for backlight brightness control |
| `light` / `monochromatic` | Backlight dimming abstraction |
| `font` / `gfonts://Roboto` | Google Fonts Roboto at 4 sizes (10, 14, 26, 38px) |
| `sensor` / `rotary_encoder` | Quadrature encoder input |
| `binary_sensor` / `gpio` | Push button inputs |
| `globals` | Persistent in-memory state variables |
| `interval` | 200ms polling tick for sleep/dim state machine |
| `script` | Named reusable action sequences (`wake_screen`) |

**Font:**
- Roboto (Google Fonts, fetched at build time via `gfonts://Roboto` URI)
- Sizes: 10px (`font_tiny`), 14px (`font_small`), 26px (`font_large`), 38px (`font_xl`)

## Build & Deploy

**Build Tool:**
- ESPHome CLI or ESPHome Dashboard
- Command (typical): `esphome run mini-rotary-screen.yaml`
- Command (compile only): `esphome compile mini-rotary-screen.yaml`

**Secrets:**
- Credentials stored in a `secrets.yaml` file (not committed), referenced via `!secret` tags
- Required secrets: `wifi_ssid`, `wifi_password`, `ota_rotaryencoder_1_password`, `mqtt_broker`, `mqtt_user`, `mqtt_password`

**OTA Updates:**
- Platform: `esphome` OTA
- Password protected via `!secret ota_rotaryencoder_1_password`
- Fallback: captive portal AP (`Testroutaryencoder` SSID) for re-flashing over WiFi

**Flash method:**
- Initial flash: USB serial
- Subsequent updates: OTA over WiFi

## Runtime Environment

**Hardware target:**
- ESP32 DevKit (38-pin or compatible `esp32dev` board)
- Framework: esp-idf (not Arduino)

**Display:**
- ST7789V 240x320 TFT via SPI at 20MHz data rate
- Backlight PWM on GPIO16 via LEDC peripheral, 1kHz frequency

**Network:**
- WiFi (infrastructure mode with AP fallback)
- MQTT broker for sensor data ingestion
- ESPHome Native API for Home Assistant control plane

**Timing:**
- Display refresh: every 50ms (20 fps)
- Sleep/dim tick: every 200ms
- Dim threshold: 150 ticks (30 seconds of inactivity)
- Sleep threshold: 300 ticks (60 seconds of inactivity)
- Encoder bounce filter: 80ms direction-reversal window

**Pin assignments:**

| GPIO | Function |
|---|---|
| GPIO2 | Display RESET |
| GPIO4 | Display DC (Data/Command) |
| GPIO5 | Display CS (Chip Select) |
| GPIO16 | Backlight PWM (LEDC) |
| GPIO18 | SPI CLK |
| GPIO23 | SPI MOSI |
| GPIO25 | Encoder push button |
| GPIO26 | Key0 push button |
| GPIO32 | Encoder pin A |
| GPIO33 | Encoder pin B |

---

*Stack analysis: 2026-04-13*
