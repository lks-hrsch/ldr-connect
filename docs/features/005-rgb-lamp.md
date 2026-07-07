# Feature: #005 RGB Lamp

## Goal

A friendship lamp: set a color for your partner's lamp from your app, and the lamp
doubles as the notification channel for every incoming event.

## Motivation

Ambient light is the least intrusive, most emotional signal available — and, by design,
the project's only notification channel (there are no push notifications; see
[architecture.md](../architecture.md)).

## Behavior

The lamp has two layers:

1. **Base state:** the app publishes retained `ldr/{you}/lamp`
   `{"r":255,"g":80,"b":40,"bri":30,"mode":"solid|breathe|off","ts":…}`. Each gadget
   subscribes to its **own** `ldr/{you}/lamp` topic — but either partner's app may write
   it. This is a deliberate **ACL exception**: `lamp` is the one topic where writing
   across the partner boundary is allowed, so you can set *their* lamp color directly
   (see [mqtt-topics.md § Conventions](../mqtt-topics.md#conventions)). The gadget
   applies the base state and caches it in NVS. If enabled, Home Assistant writes this
   same base-state layer over its own `ha/lamp/set` topic — last-write-wins by `ts`, same
   as the app — and the gadget mirrors every base-state change back to `ha/lamp/state`
   (see [#018 Home Assistant Integration](018-home-assistant.md)).
2. **Notification overlays:** incoming events from the partner — interaction, status,
   presence — trigger a temporary, type-specific animation that overrides the base
   state for a few seconds, then reverts to it.

Firmware caps global brightness at ~30–40 % as a hard power/thermal budget (see
[hardware.md](../hardware.md)).

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Subscribes: `ldr/{you}/lamp` (retained; writable by either partner's app).
- Overlay triggers come from: `ldr/{other}/interaction`
  ([#008](008-interactions.md)), `ldr/{other}/status` ([#007](007-status-mood.md)),
  `ldr/{other}/presence` ([#006](006-presence.md)).
- Also writable via `ha/lamp/set`, mirrored to `ha/lamp/state`, when
  [#018 Home Assistant Integration](018-home-assistant.md) is enabled.

## Hardware

- WS2812/SK6812 ring or stick, 8–16 LEDs, driven via RMT. Budget ~60 mA/LED at full
  white — the brightness cap is mandatory, not optional. See
  [hardware.md](../hardware.md).

## States & Edge Cases

- **Overlay vs. base priority and revert timing:** an overlay must cleanly revert to
  whatever the base state was, even if the base state changed while the overlay was
  playing.
- **Multiple overlapping events:** queue them — never stack brightness/color across
  simultaneous overlays.
- **Lamp "off" mode:** must still allow notification overlays to play (configurable —
  "off" means the base state is dark, not that the lamp is disabled).
- **First boot without retained lamp state:** fall back to the NVS cache; if none
  exists, default to warm white at low brightness.
- **Brownout protection:** the brightness cap doubles as the primary mitigation — see
  [#015 Power Metering](015-power-metering.md) for the runtime VBUS-sag response.
- **Quiet hours:** event-driven light (notification overlays, celebrations) is
  suppressed; explicit base-state commands (app or HA) and local user-initiated feedback
  still play — the full rule is owned by
  [#010 Quiet Hours / Night Mode](010-quiet-hours.md).
