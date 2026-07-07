# Feature: #007 Status & Mood

## Goal

Share what you're up to — "home & bored", "call me on FaceTime", "busy", "sleeping" — so
your partner sees it at a glance on their gadget.

## Motivation

The core original idea of the project: low-friction ambient state sharing instead of
texting.

## Behavior

1. **Sender side:** you set a status in the iOS app (a predefined set, or free text) →
   the app publishes retained `ldr/{you}/status`
   `{"mood":"home_bored","text":"call me","ts":…}`.
2. Optionally, a mapped button can cycle or set a status directly (see
   [#009 Button Mapping](009-button-mapping.md)).
3. **Auto-status from Home Assistant (optional):** if
   [#018 Home Assistant Integration](018-home-assistant.md) is enabled, HA may publish
   `"home"`/`"away"` context sourced from phone-WiFi presence, which the gadget applies
   as an automatic status — unless a manual status was set more recently, in which case
   the manual status wins for 2 h or until cleared.
4. **Receiver side:** the partner's gadget subscribes to your status and renders a
   mood icon/text on its e-ink, within the display's refresh discipline.
5. A new status may trigger a subtle lamp pulse on the partner's gadget (see
   [#005 RGB Lamp](005-rgb-lamp.md)).

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes: `ldr/{you}/status` (retained).
- Subscribes: `ldr/{other}/status`.

## Hardware

- e-ink (status rendering) — see [hardware.md](../hardware.md).
- Buttons, optionally, for status cycling (see [#009 Button Mapping](009-button-mapping.md)).

## States & Edge Cases

- **Stale status:** show a relative age (e.g. "3h ago") derived from `ts` rather than
  implying it's current.
- **Partner offline:** presence overrides status display — show the offline state and
  dim the status rather than presenting it as live.
- **Free text length:** capped to fit the e-ink layout.
- **Retention:** the retained message survives reboots by design — status persists
  without the sender needing to resend it.
