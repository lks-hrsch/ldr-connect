# Feature: #015 Power Metering

## Goal

See how much power the gadget draws in real time — nerd telemetry, and an early-warning
system for the LED power budget.

## Motivation

A single USB-C inlet makes one measurement point cover the whole device; this also
validates that the lamp's brightness cap (see [#005 RGB Lamp](005-rgb-lamp.md)) is actually
holding.

## Behavior

1. An INA226 (with a 10–20 mΩ external shunt; INA219 as a pragmatic alternative) sits
   high-side on the 5 V rail directly behind the USB-C jack, measuring total device
   consumption.
2. Firmware samples at ~1 Hz and publishes retained `ldr/{you}/power`
   `{"v":5.02,"i":0.34,"w":1.7,"ts":…}`, every 60 s or immediately on a significant
   change (ΔW ≥ 0.5 W).
3. The app shows the live value plus a local chart.
4. Firmware watches VBUS: if voltage sags below ~4.7 V, it reduces LED brightness
   (brownout protection) and flags this in the payload.
5. **Cumulative energy:** firmware integrates the sampled power over time into a
   running kWh counter, persisted to NVS hourly (and on graceful shutdown) — the flash
   wear vs. loss trade-off, at most 1 h of accumulation lost on a power cut. This counter
   is consumed by [#018 Home Assistant Integration](018-home-assistant.md) to feed the HA
   Energy Dashboard.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes: `ldr/{you}/power` (retained).

## Hardware

- INA226 or INA219 (I²C `0x40`), on the USB-C 5 V rail. Requires the dual 5.1 kΩ CC
  pulldowns described in [hardware.md](../hardware.md).

## States & Edge Cases

- **Shunt range limits:** the INA226's 16-bit ADC has a ±81.9 mV shunt-voltage limit —
  shunt sizing must respect this (see [hardware.md](../hardware.md)).
- **VBUS sag handling:** brightness reduction is automatic and reversible as voltage
  recovers.
- **Sensor failure:** omit `power` from telemetry; the app shows "n/a".
