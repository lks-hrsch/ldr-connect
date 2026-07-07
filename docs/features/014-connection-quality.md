# Feature: #014 Connection Quality

## Goal

A glance at the e-ink tells you whether your gadget's link to the broker is good, poor,
or down — so you know that silence means "offline," not "ignored."

## Motivation

In an ambient-presence system, link health is itself presence information.

## Behavior

1. The gadget measures round-trip time to the broker every 30–60 s (via a self-loopback
   topic it publishes and subscribes to itself, or by timing MQTT `PINGREQ`/`PINGRESP`).
2. It classifies the result into coarse buckets: **good** (<150 ms), **poor**
   (<800 ms), **offline** (no `PINGRESP` / reconnecting).
3. It publishes `ldr/{you}/ping` `{"rtt_ms":42,"quality":"good","ts":…}`, not retained.
4. Your own e-ink shows **your own** link-quality icon. Your partner's overall
   reachability is derived from presence ([#006](006-presence.md)), not from their ping —
   ping is a self-diagnostic, not a cross-device signal.
5. The e-ink updates only when the quality bucket **changes** (not on every
   measurement), with hysteresis (two consecutive samples required before switching)
   to respect the display's refresh discipline.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes: `ldr/{you}/ping` (not retained).
- Subscribes (app, optional): both `ldr/{you}/ping` and `ldr/{other}/ping`, for
  diagnostics.

## Hardware

- e-ink status icon. See [hardware.md](../hardware.md).

## States & Edge Cases

- **Flapping links:** hysteresis (two consecutive samples) prevents the icon from
  flickering between buckets.
- **Reconnect backoff interacting with LWT:** brief offline flickers during reconnect
  are suppressed rather than surfaced.
- **WiFi up but broker unreachable:** counts as offline, same as no WiFi.
- **During OTA:** the connection-quality measurement is skipped so it doesn't compete
  with the firmware transfer (see [#017 OTA Firmware Updates](017-ota-updates.md)).
