# Feature: #008 Interactions (Hug / Kiss)

## Goal

Press a button and your partner's gadget lights up with a hug or kiss — a physical
gesture across distance.

## Motivation

The emotional heart of the project; inspired by friendship lamps and Bond Touch.

## Behavior

1. **Sender side:** you press a mapped button (see
   [#009 Button Mapping](009-button-mapping.md)) or tap in the app → publish
   `ldr/{you}/interaction` `{"type":"hug","ts":…}`, at QoS 1, **not retained** — a
   gesture is transient by design.
2. You get local feedback (a short lamp blink) on publish confirmation.
3. **Receiver side:** the partner's gadget plays a type-specific lamp animation (e.g. a
   warm slow pulse for hug, a quick double flash for kiss — exact effects defined in
   [#005 RGB Lamp](005-rgb-lamp.md)) and updates a "last gesture: hug, 12:34" line on its
   e-ink. **During the receiver's own quiet hours**
   ([#010 Quiet Hours](010-quiet-hours.md)), the lamp animation does not fire — the
   gesture is instead counted and queued for a wake-time summary; the "last gesture" line
   still updates.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes: `ldr/{you}/interaction` (QoS 1, not retained).
- Subscribes: `ldr/{other}/interaction`.
- Related: [#009 Button Mapping](009-button-mapping.md), [#005 RGB Lamp](005-rgb-lamp.md),
  [#010 Quiet Hours](010-quiet-hours.md).

## Hardware

- Buttons (GPIO) — see [hardware.md](../hardware.md).
- WS2812/SK6812 lamp.
- e-ink (last-gesture line).

## States & Edge Cases

- **Receiver gadget offline at send time:** the gesture is lost — deliberate, since the
  topic is not retained. The e-ink "last gesture" line only ever reflects gestures
  received while online. This is an accepted trade-off; a retained-status mirror is a
  possible future extension, not implemented here.
- **Button debounce:** required in firmware to avoid duplicate sends on a single press.
- **Rapid repeated presses:** rate-limit the lamp animation queue so gestures don't pile
  up into a strobe effect.
- **Duplicate delivery under QoS 1:** dedupe on the receiver using `ts`.
