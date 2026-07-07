# Feature: #018 Home Assistant Integration

## Goal

The gadget appears in Home Assistant as a native device — the lamp as a controllable
light entity (a night-light use case), power/energy in the Energy Dashboard — and HA can
feed context back (home/away, bedtime routines) without ever seeing intimate data.

## Motivation

The gadget sits on a desk or nightstand inside a smart home. HA integration turns the
lamp into a first-class night light, exposes the INA metering to the Energy Dashboard,
and automates the project's original idea — "status: home" set automatically when the
phone joins the home WiFi (via HA person tracking). The end-to-end encryption
([#002 Security & Pairing](002-security.md)) makes this a deliberate privacy boundary,
not an accident: HA gets a curated plaintext surface, never the intimate payloads.

## Behavior

1. **Opt-in:** disabled by default, enabled via an
   `"ha":{"enabled":true,"climate_export":false}` block in the retained encrypted config
   topic ([#009 Button Mapping](009-button-mapping.md)). HA connects to the **same**
   Mosquitto broker as an additional client, with its own credentials and ACL (see the
   ACL extension below).
2. **Privacy boundary:**

   | Exported (plaintext, `ha` namespace) | Never exported |
   | --- | --- |
   | Lamp state + control, power (W/V/A), cumulative energy (kWh), availability (online/offline), diagnostics (WiFi RSSI, firmware version, uptime, link-quality bucket from [#014](014-connection-quality.md)), room climate — **only** if `climate_export=true` (a separate opt-in) | Status/mood text, interactions/gestures, heartbeat, images, countdown, any encrypted payload content |

   HA cannot decrypt the AEAD envelopes ([#002](002-security.md)) by design; the export
   surface is a conscious dual-publish of operational data only.
3. **Topic namespace:** HA-facing state/command topics live under `ldr/{you}/ha/…`
   (plaintext), e.g. `ha/lamp/state`, `ha/lamp/set`, `ha/power/state`, `ha/energy/state`,
   `ha/climate/state`, `ha/rssi/state`, `ha/presence/set`, `ha/quiet/state`,
   `ha/quiet/set`, `ha/identify/press`. `state` topics are retained (HA repopulates after
   a restart); `set`/`press` command topics are not. The encrypted contract topics remain
   untouched.
4. **MQTT Discovery:** on (re)connect with `ha.enabled`, the gadget publishes retained
   discovery configs under the standard prefix:
   `homeassistant/<component>/ldrc-<XXXX>/<object_id>/config` (per-entity discovery;
   `XXXX` from the MAC, matching the mDNS/SoftAP name). Every config carries a shared
   `device` block (`identifiers: ["ldrc-<XXXX>"]`, `name: "ldr-connect <person>"`,
   `manufacturer: "ldr-connect"`, `model: "ESP32-S3 gadget"`, `sw_version: "<fw>"`) so all
   entities group under one HA device. Availability for **all** entities is
   `availability_topic = ldr/{you}/presence` with an `availability_template` extracting
   the `online` boolean (the presence topic is plaintext JSON — this works natively).
   **Birth handling (critical):** the gadget subscribes to `homeassistant/status` and
   re-publishes all discovery configs (after a short random delay) whenever HA announces
   `online` — otherwise entities vanish after an HA restart. **Decision:** HA also
   supports device-based discovery (a single config per device) since 2024.9; per-entity
   discovery is chosen here for maximal documentation coverage and debuggability.
5. **Entities:**

   | Entity | Component | Details |
   | --- | --- | --- |
   | Lamp | `light` | schema `json`, brightness + RGB color, `effect_list` `["solid","breathe"]`; command `ha/lamp/set`, state `ha/lamp/state` |
   | Power | `sensor` | `device_class: power`, W, `state_class: measurement` |
   | Voltage / Current | `sensor` | `device_class: voltage`/`current`, `entity_category: diagnostic` |
   | Energy | `sensor` | `device_class: energy`, kWh, `state_class: total_increasing` |
   | Temperature / Humidity | `sensor` | only when `climate_export=true`; `device_class: temperature`/`humidity` |
   | WiFi RSSI | `sensor` | `device_class: signal_strength`, dBm, `entity_category: diagnostic` |
   | Firmware version, uptime, link quality | `sensor` | `entity_category: diagnostic`; link quality from [#014](014-connection-quality.md) |
   | Quiet mode | `switch` | state `ha/quiet/state`, command `ha/quiet/set` — lets bedtime routines silence the gadget ([#010](010-quiet-hours.md)); same hold-until-next-boundary semantics as the button |
   | Identify | `button` | `ha/identify/press` — the lamp plays a short identify animation, for matching gadget ↔ HA device |
   | Partner-device online | `binary_sensor` (`connectivity`) | sourced from the partner's presence topic — plaintext anyway, useful for automations like "lamp red when partner gadget offline" |

   The Energy sensor requires firmware to integrate INA power over time
   ([#015 Power Metering](015-power-metering.md)), persisting the counter to NVS hourly
   and on graceful shutdown (a flash-wear-vs-loss trade-off: at most 1 h of accumulation
   lost on a power cut).
6. **Lamp control semantics:** HA light commands write the same base-state layer as the
   app's lamp topic ([#005 RGB Lamp](005-rgb-lamp.md)) — last-write-wins by `ts`; the
   gadget mirrors every base-state change back to `ha/lamp/state` so the app and HA stay
   consistent. Notification overlays still temporarily override the base state.
   **Quiet-hours amendment** (cross-cutting, owned by
   [#010 Quiet Hours](010-quiet-hours.md)): during quiet hours the lamp is dark for all
   animations; the single exception is an explicit base-state light command from the app
   or HA — a deliberately switched-on night light is user intent, not a notification.
   This enables the night-light use case: an HA automation can turn the lamp on warm/dim
   at bedtime, off in the morning, or motion-triggered.
7. **HA → gadget context:** HA automations publish `{"home":true,"ts":…}` to
   `ldr/{you}/ha/presence/set` (sourced from an HA person entity / phone WiFi presence).
   The gadget maps this to its own auto-status — setting mood `"home"`/`"away"` on the
   **encrypted** status topic ([#007 Status & Mood](007-status-mood.md)) — unless the
   user has set a manual status more recently (manual wins for 2 h, or until cleared). HA
   thereby automates the project's founding idea ("status: home" when the phone is on
   home WiFi) while only ever handling information it already owns.
8. **ACL extension** (updates the broker ACL spec in
   [architecture.md](../architecture.md)): HA credentials may **read** `ldr/+/ha/#`,
   `ldr/+/presence`, and `homeassistant/#`, and **write** only `ldr/+/ha/+/set`,
   `ldr/+/ha/+/press`, and `homeassistant/status` (the birth message is published by HA
   itself). Gadgets additionally get **write** on `homeassistant/#` for their own
   discovery configs, and **read** on `homeassistant/status`. HA gets **no** access to
   the encrypted topics — defense in depth on top of the encryption.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- New plaintext subtree: `ldr/{you}/ha/#` (state/command per the entity table above).
- New retained discovery configs under `homeassistant/…` (outside the `ldr` namespace).
- The encrypted contract topics are unchanged.
- Related: [#002 Security & Pairing](002-security.md), [#005 RGB Lamp](005-rgb-lamp.md),
  [#006 Presence](006-presence.md), [#007 Status & Mood](007-status-mood.md),
  [#009 Button Mapping](009-button-mapping.md), [#010 Quiet Hours](010-quiet-hours.md),
  [#014 Connection Quality](014-connection-quality.md),
  [#015 Power Metering](015-power-metering.md).

## Hardware

No new hardware. WS2812 lamp (light entity, identify animation); INA226/INA219
(power/energy); SHT45/BME280 (optional climate export); NVS (energy counter). See
[hardware.md](../hardware.md).

## States & Edge Cases

- **HA restart:** the birth message on `homeassistant/status` triggers a discovery
  republish from every gadget with `ha.enabled`.
- **Gadget reboot:** retained discovery configs survive at the broker; state topics
  repopulate on reconnect.
- **`ha.enabled` switched off:** the gadget publishes empty retained payloads to all of
  its own discovery config topics, cleanly removing the device from HA, then stops
  dual-publishing.
- **Energy counter across a factory reset** ([#016 Factory Reset](016-factory-reset.md)):
  the counter is wiped; HA sees this as the start of a new `total_increasing` cycle,
  which it handles natively.
- **Factory reset residue:** a factory reset
  ([#016 Factory Reset](016-factory-reset.md)) does **not** remove the device from HA on
  either path — the retained discovery configs and `ha/*/state` topics stay at the
  broker, and HA shows the device as unavailable until it is removed manually there (or
  a re-provisioned gadget with `ha.enabled` republishes them).
- **Identify pressed during quiet hours:** suppressed — quiet means dark; only explicit
  base-state light commands are exempt ([#010 Quiet Hours](010-quiet-hours.md)). Press
  it again outside quiet hours.
- **Conflicting lamp writes, app vs. HA:** last-write-wins by `ts`; both are mirrored to
  `ha/lamp/state`.
- **Quiet mode toggled simultaneously via button and HA switch:** same state machine,
  last event wins; `ha/quiet/state` reflects the result.
- **`climate_export` toggled off:** remove the climate discovery configs via an empty
  retained payload.
- **HA publishes malformed JSON to a `set` topic:** ignore it, log the failure.
- **Partner gadget has HA disabled:** the partner-online `binary_sensor` still works —
  presence is plaintext regardless of HA integration.
- **Broker ACL misconfiguration** is the only thing standing between HA and ciphertext —
  intimate content stays unreadable either way
  ([#002 Security & Pairing](002-security.md)).
