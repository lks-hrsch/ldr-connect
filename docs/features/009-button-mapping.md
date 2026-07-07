# Feature: #009 Button Mapping

## Goal

Assign what each physical button does — hug, kiss, status cycle, … — from the app, with
no reflashing needed.

## Motivation

Buttons are the gadget's main input; their meaning should be personal and changeable,
not hardcoded in firmware.

## Behavior

1. You edit the mapping in the app's UI → the app publishes retained
   `ldr/{you}/config`
   `{"buttons":{"1":"hug","2":"kiss","3":"status_cycle","4":"quiet_toggle"},"lamp_defaults":{…},"tz":{…},"schema":1,"ts":…}`.
2. The gadget subscribes to its own config, applies it, and caches the full config in
   NVS so it keeps working offline and after a reboot.
3. On a button press, firmware resolves the current mapping and executes the
   corresponding action — publishing an interaction ([#008](008-interactions.md)), cycling
   status ([#007](007-status-mood.md)), toggling `quiet_toggle`
   ([#010 Quiet Hours](010-quiet-hours.md)), etc. Unmapped buttons do nothing.
4. The same config payload also carries lamp defaults ([#005](005-rgb-lamp.md)),
   timezone settings ([#003 Partner Clock](003-partner-clock.md)), the quiet-hours
   schedule ([#010](010-quiet-hours.md)), and the Home Assistant opt-in block
   ([#018 Home Assistant Integration](018-home-assistant.md)).

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes (app): `ldr/{you}/config` (retained).
- Subscribes (gadget): `ldr/{you}/config`.
- Related: [#008 Interactions](008-interactions.md), [#007 Status & Mood](007-status-mood.md),
  [#003 Partner Clock](003-partner-clock.md), [#010 Quiet Hours](010-quiet-hours.md).

## Hardware

- 2–4 GPIO buttons, debounced in firmware. See [hardware.md](../hardware.md).

## States & Edge Cases

- **First boot without config:** fall back to built-in defaults (button 1 = hug, button
  2 = kiss).
- **Malformed config JSON:** reject it, keep the last valid config from NVS, and report
  the failure via log.
- **Config schema versioning:** the `schema` field allows forward-compatible changes to
  the payload shape.
- **Race between NVS cache and retained delivery on boot:** the newer `ts` wins.
- **Image takeover active:** while an unacknowledged image is displayed
  ([#012 Image Messages](012-image-message.md)), takeover mode captures button presses
  for acknowledgement and the normal mapping is suspended until it exits.
