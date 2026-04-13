# CONCERNS.md

**Analysis Date:** 2026-04-13

## Overview

`mini-rotary-screen.yaml` is a single-file ESPHome configuration for an ESP32-based
rotary encoder display. The project is functional but has several security, reliability,
and maintainability issues that should be addressed before treating this as a
production-grade Home Automation peripheral.

---

## Security

**Hardcoded API encryption key:**
- The ESPHome native API encryption key is committed directly in `mini-rotary-screen.yaml`
  line 24: `key: "pbjMGwv067T2eCrzDuOYoy41DwrDPAMoVmDKhhFfRIc="`.
- Risk: Anyone with read access to the repo can authenticate against the device's
  native API, enabling remote control and sensor data exfiltration.
- Fix: Move to `!secret api_encryption_key` and store in `secrets.yaml` (gitignored).

**Hardcoded fallback AP password:**
- The Wi-Fi access point fallback SSID/password is hardcoded in `mini-rotary-screen.yaml`
  line 35-36: `password: "aCo5cygdz5PB"`.
- Risk: The AP password is visible in version control. Anyone who finds the device's
  fallback AP can join it and attempt further access.
- Fix: Move to `!secret ap_password`.

**Partially correct secrets usage:**
- OTA password, Wi-Fi credentials, and MQTT credentials all correctly use `!secret`
  (lines 28, 31-32, 40-42). The gap is the two items above.

**No MQTT TLS configuration:**
- MQTT connection (`mqtt:` block, lines 39-61) uses no TLS settings. If the broker
  is on a LAN-only network this is acceptable, but there is no enforcement.
- Risk: Credentials and temperature/control data travel in plaintext.
- Fix: Add `ssl_fingerprints:` or `certificate_authority:` if broker supports TLS.

**MQTT target temperature write with no authentication guard:**
- On encoder push, the device publishes to `home/heater/target_temp/set` (line 394).
  Any MQTT client with broker access can also publish to that topic, and the device
  will update `target_temp` indirectly via the incoming `home/heater/current_temp`
  subscription. There is no server-side validation of the value range before it reaches
  downstream actuators.

---

## Reliability

**`enc_last_ms` stored as `int`, millis() is `uint32_t`:**
- `enc_last_ms` global is declared `type: int` (line 141), but `millis()` returns
  `uint32_t`. The cast `(int)now` on lines 419 and 439 will silently overflow after
  ~24.8 days of uptime, causing the bounce-filter comparison to produce an incorrect
  negative delta and potentially suppressing valid encoder events for one direction
  for ~80 ms.
- Fix: Change global type to `uint32_t` and remove the cast.

**Bounce filter direction assumption is brittle:**
- The software bounce filter (lines 416-420, 436-440) suppresses a new event only if
  the direction reversed within 80 ms. Mechanical encoders can produce same-direction
  glitches; the filter will not catch those. False counts accumulate silently.
- Fix: Consider ESPHome's built-in `publish_initial_value` and hardware-level RC
  filtering, or increase `resolution:` to 2 or 4 to require more pulses per detent.

**Display lambda runs unconditionally at 50 ms:**
- `update_interval: 50ms` (line 192) means the full display lambda (120+ lines,
  including 60-step arc rendering with `cosf`/`sinf` calls) runs 20 times per second
  regardless of whether any state has changed.
- Risk: Wastes CPU and SPI bus bandwidth; may cause visible tearing if a lambda
  overruns its slot.
- Fix: Track a `display_dirty` flag; only re-render when state actually changes, or
  reduce `update_interval` to 100-200ms when sleeping.

**Sleep detection duplicated across lambda and script:**
- `sleeping` is recalculated inline in both the display lambda (line 193) and the
  sleep/dim interval lambda (line 149). The `was_sleeping` flag in `wake_screen`
  (line 168) adds a third representation of the same threshold.
- If `SLEEP_TIMEOUT_TICKS` or `DIM_TIMEOUT_TICKS` change, all three locations must
  stay consistent.

**`was_sleeping` race window:**
- `was_sleeping` is set in `wake_screen` script (line 168), then read in the encoder
  on_clockwise / on_anticlockwise lambdas (lines 420, 440) and the push handler
  (line 384). ESPHome runs on a cooperative scheduler; if a second interrupt fires
  before the first script fully executes, `was_sleeping` may be stale.

**No MQTT reconnect/LWT configuration:**
- There is no `birth_message:`, `will_message:`, or `reboot_timeout:` under the
  `mqtt:` block. If the broker restarts, ESPHome's default reconnect behavior applies
  but subscribers have no way to detect the device went offline.

---

## Maintainability

**Massive inline display lambda (120+ lines):**
- The entire rendering pipeline for all three screens lives in a single
  `display: lambda:` block starting at line 192. This is the maximum complexity
  ESPHome lambdas support without external C++ components. Adding a fourth screen
  requires editing a deeply nested, hard-to-test block.
- Fix: Extract per-screen drawing into named `script:` blocks or a custom ESPHome
  C++ component.

**Hundreds of magic colour hex literals:**
- Colours like `Color(0x0C, 0x15, 0x20)` appear 20+ times with no named constants.
  Changing the theme requires hunting every occurrence. ESPHome YAML does not have
  a colour alias mechanism, but substitutions could encode the most-used values.

**Magic pixel coordinates throughout:**
- Y-positions such as `148`, `262`, `291`, `308` are layout coordinates with no
  comments explaining the grid. Rotating the display or changing font sizes requires
  manually recalculating every value.

**`num_controls` global is a manual count:**
- `num_controls` is hardcoded to `3` (line 105). Adding a screen requires remembering
  to increment this global separately. There is no compile-time check that it matches
  the actual number of `if/else if` branches in the display lambda.

**`selected_control` uses raw integers, not symbolic enum:**
- Screen indices `0`, `1`, `2` are scattered through encoder handlers and display
  lambda with no named constants. The meaning of each index is only understood by
  reading all branches.

**`editing_mode` only meaningful on screen 0:**
- `editing_mode` is a global boolean but the encoder push handler already gates it
  to `selected_control == 0` (line 385). If a second editable screen is added this
  pattern breaks. There is no per-screen edit state.

**Substitution constants only for timeouts:**
- `DIM_TIMEOUT_TICKS` and `SLEEP_TIMEOUT_TICKS` are correctly extracted as
  substitutions, but the temperature range limits (`5.0`, `35.0`, `30.0`), the
  encoder step size (`0.5`), and the hot-water thresholds (`55.0`, `40.0`, `30.0`,
  `10.0`) are all magic numbers in lambdas with no substitutions.

---

## Missing Features

**No persistence for `target_temp`:**
- `target_temp` is a plain global with `initial_value: 18.0`. On device reboot it
  resets to 18°C. ESPHome's `restore_value: true` on a global backed by
  `preferences` (flash) would survive reboots. File: `mini-rotary-screen.yaml` line 110.

**Encoder editing only wired for screen 0:**
- The `switch` blocks in `on_clockwise` / `on_anticlockwise` (lines 422-426,
  444-447) contain only `case 0:`. Screens 1 and 2 have no interactive controls
  even though `editing_mode` can technically be activated from them (it cannot —
  the push guard prevents it, but the switch still falls through silently).

**No Wi-Fi signal or uptime status display:**
- The device exposes three informational screens but none show connectivity health
  (RSSI, MQTT connection state, uptime). A disconnected device shows stale data
  with no visual indication.

**No visual indicator for stale MQTT data:**
- If the MQTT broker goes offline, all temperature globals retain their last value
  indefinitely. There is no timestamp tracking or "last updated" display.

**Target temperature not clamped consistently:**
- CW rotation clamps at `30.0f` (line 424) but the arc display uses `maxT = 35.0f`
  (line 228) and the label shows "35" (line 272). The upper bound is visually
  promised as 35°C but the encoder can only reach 30°C.
  File: `mini-rotary-screen.yaml` lines 228, 272, 424.

**No OTA progress or error feedback on display:**
- During OTA updates the display continues rendering normally. A user turning the
  encoder during an OTA flash could produce unexpected MQTT publishes.

---

## Performance

**60-step arc rendered with `cosf`/`sinf` every 50 ms:**
- The arc on screen 0 calls `cosf` and `sinf` 120 times per render cycle (60 segments
  × 2 trig functions), executed at 20 Hz. On ESP32 this is manageable but unnecessary
  when the arc only needs updating when `target_temp` changes.
- Fix: Pre-compute arc segment coordinates into a static array at boot, then index
  into it during render. Or dirty-flag the display.

**Full `it.fill()` background repaint every frame:**
- Every 50 ms the entire 240×320 framebuffer is filled (line 204), then redrawn.
  The ST7789V at 20 MHz SPI can handle this, but it means no partial update
  optimisation is possible with the current approach.

**`char buf[16]` stack allocation inside display lambda:**
- Minor: `buf` is allocated on each display call. Not a concern in practice on ESP32
  but worth noting if the stack frame grows.

**Font loading from `gfonts://`:**
- Four sizes of Roboto are compiled in (lines 85-96). Unused glyphs are included by
  default unless `glyphs:` is specified. Specifying only the characters actually
  rendered (digits, `°C`, the handful of uppercase labels) would reduce flash usage
  and compile time.

---

## Technical Debt

**Device name is a typo: "TestRoutaryEncoder":**
- `friendly_name: TestRoutaryEncoder` (line 3) and the AP SSID `Testroutaryencoder`
  (line 35) spell "rotary" as "routary". This propagates to Home Assistant entity
  names and mDNS hostname.

**ESPHome name is "testroutaryencoder":**
- The `name:` field (line 2) is the mDNS/hostname. Renaming requires a full
  reflash; it cannot be done via OTA alone without Home Assistant entity migration.

**No `secrets.yaml` committed or documented:**
- The repository contains no `secrets.yaml.example` or documentation listing which
  secret keys are required (`wifi_ssid`, `wifi_password`, `mqtt_broker`, `mqtt_user`,
  `mqtt_password`, `ota_rotaryencoder_1_password`). A new contributor cannot build
  the firmware without discovering required keys by reading the YAML.

**MQTT topics are hardcoded strings:**
- Topics like `"home/heater/current_temp"` and `"home/heater/target_temp/set"`
  appear as bare strings (lines 44-61, 394). No substitution variables. Changing
  the topic namespace requires editing multiple places.

**`resolution: 1` on rotary encoder:**
- Using resolution 1 means every electrical pulse fires an event. Most mechanical
  encoders have 2 or 4 pulses per physical detent, causing multi-step jumps per
  click. The software bounce filter partially mitigates this but the root cause
  is the resolution setting. File: `mini-rotary-screen.yaml` line 410.

---

## Known Issues

**Target temperature upper bound mismatch (35 vs 30):**
- As noted under Missing Features: the arc scale and min/max labels promise 5–35°C,
  but the encoder clamps at 30°C. This is a visible bug to any user who tries to
  set a temperature above 30°C.
- Files: `mini-rotary-screen.yaml` lines 228, 272, 424.

**`millis()` cast overflow after ~24 days:**
- As noted under Reliability: `enc_last_ms` stored as `int` will overflow, causing
  the direction-bounce filter to malfunction. File: `mini-rotary-screen.yaml` line 141.

**Wake-from-sleep suppresses first interaction:**
- After waking, `was_sleeping` causes the encoder and push handler to swallow the
  first event (lines 384, 420, 440). This is intentional per the comment logic but
  means the user must interact twice: once to wake, once to act. There is no
  documentation of this design decision.

---

*Concerns audit: 2026-04-13*
