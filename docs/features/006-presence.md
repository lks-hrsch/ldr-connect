# Feature: #006 Presence

## Goal

Lets each person feel when their partner's gadget is online — a lightweight "I'm here"
signal, shared in real time between the two devices.

## Motivation

The core promise of `ldr-connect` is to close distance with small, ambient signals rather
than active messaging. Presence is the simplest of those signals: no typing, no explicit
action — the other person's gadget just knows.

## Behavior

1. **Sender side:** on boot/connect, a gadget publishes `{"online":true,"ts":…}` to its
   own `ldr/{you}/presence` topic, retained, and registers a Last Will & Testament
   (LWT) of `{"online":false,"ts":…}` on the same topic with the broker.
2. As long as the connection is alive, the retained message stays `online:true`.
3. **Receiver side:** the partner's gadget (and the iOS app) subscribes to
   `ldr/{other}/presence`. On receiving `online:true`, it drives its e-ink status
   indicator to show the partner is connected.
4. If the gadget disconnects uncleanly (power loss, WiFi drop), the broker fires the LWT
   automatically — no gadget-side "goodbye" message is required.
5. While quiet hours are active, the regular retained presence payload carries an
   optional `"quiet":true` flag — rendered as a moon icon next to the partner's presence
   indicator. The static LWT payload never includes it; the privacy trade-off of the
   plaintext flag is documented in
   [#010 Quiet Hours / Night Mode](010-quiet-hours.md).

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract. Relevant topic:

- `ldr/{you}/presence` — retained + LWT, published by your gadget; mirrored as
  `ldr/{other}/presence` on your partner's side, which you subscribe to. (link to
  [mqtt-topics.md](../mqtt-topics.md))

## Hardware

- Output: e-ink status area shows partner connection state. (link to
  [hardware.md](../hardware.md))
- No dedicated input for this feature — presence is connection-derived, not
  button-triggered.

## States & Edge Cases

- **Partner offline:** the broker's LWT flips the retained message to `online:false`
  automatically; no staleness timer is needed for this topic specifically.
- **Both devices online at once:** no conflict, each direction is independent.
- **Broker unreachable from a gadget's perspective:** the gadget shows partner state as
  unknown/stale rather than a frozen "online" — driven by the same connection-quality
  signal as `ldr/{you}/ping` (see [mqtt-topics.md](../mqtt-topics.md)).
