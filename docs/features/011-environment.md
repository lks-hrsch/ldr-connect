# Feature: #011 Room Climate

## Goal

See temperature and humidity at your partner's place — a small window into their world
("it's cold there today").

## Motivation

Ambient context deepens presence; the same sensor also provides useful telemetry about
the gadget's own environment.

## Behavior

1. The gadget reads its climate sensor (SHT45 precision pick; BME280 alternative) every
   60 s.
2. It publishes retained `ldr/{you}/env` `{"t":21.5,"rh":45,"ts":…}`, at most every
   5 minutes, or immediately on a significant change (Δt ≥ 0.3 °C or Δrh ≥ 2 %).
3. The partner's app displays the current values plus a local time-series chart (the app
   buffers this while foregrounded; there is no server-side history — see
   [architecture.md](../architecture.md)).
4. Optionally, the partner's env reading is shown in a corner of the e-ink.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes: `ldr/{you}/env` (retained).
- Subscribes (app, both directions): `ldr/{you}/env`, `ldr/{other}/env`.

## Hardware

- SHT45 (I²C `0x44`, precision) or BME280 (I²C `0x76`, adds pressure). **Must be
  thermally decoupled** from the ESP32/LEDs/displays — enclosure self-heating otherwise
  skews readings by 3–8 °C. Placement rules in [hardware.md](../hardware.md).

## States & Edge Cases

- **Sensor init failure:** the gadget omits `env` from its telemetry; the app shows
  "n/a" rather than a stale or fabricated reading.
- **Self-heating drift despite decoupling:** document an optional calibration offset as
  a future mitigation.
- **Stale env (>15 min):** the app greys out the values rather than presenting them as
  current.
