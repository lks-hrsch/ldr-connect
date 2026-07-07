# Feature: #016 Factory Reset

## Goal

Return a gadget to out-of-the-box state — via a guided flow in the app, or, without any
app at all, via a button hold on the device.

## Motivation

Devices get re-paired, re-gifted, debugged, or sold; a defined wipe path is a security
feature (keys!) as much as a convenience.

## Behavior

Two paths reach the same end state.

1. **(A) Hardware path — no app, no network:** hold button 1 for **10 s**; the lamp
   shows a color-ramp hold-progress (deliberate-action feedback, so it can't happen by
   accident). At 10 s the gadget wipes NVS and reboots into setup mode
   ([#001 Setup & Provisioning](001-setup.md), SoftAP). **This is the canonical spec for
   the mechanism summarized in [#001](001-setup.md).** The hold is detected below the
   button-mapping layer, so it works even while an image takeover
   ([#012 Image Messages](012-image-message.md)) has captured normal presses. During
   quiet hours the lamp stays dark ([#010 Quiet Hours](010-quiet-hours.md)) — the hold
   progress is shown on the e-ink instead.
2. **(B) App path — guided:** reachable **only** via the local mDNS management channel
   ([#001](001-setup.md)), **never** via MQTT. Rationale: no remote-wipe path shall
   exist even if broker credentials or the account are compromised — physical LAN
   proximity is a deliberate requirement. Flow: the app discovers the gadget → sends an
   AEAD-authenticated reset request (**AAD = the request path**, key material per
   [#002 Security & Pairing](002-security.md)) → the app shows explicit confirmation
   listing the consequences → the gadget replies OK, then wipes and reboots into setup
   mode.
3. **(B) Guided cleanup:** after the wipe, the app publishes **zero-length retained**
   messages to all of the OWN person's retained topics — `presence`, `status`, `env`,
   `power`, `lamp`, `config`, `countdown`, `image`, `ota/offer` — clearing stale retained
   state from the broker. The wiped gadget can no longer do this itself, and the broker
   retains state independently of the device, so only the app (still holding the old
   credentials at that moment) can clear it. Non-retained topics need no cleanup.
4. **What is wiped:** all of NVS — WiFi credentials, broker host/CA, MQTT credentials,
   key material ([#002](002-security.md)), config/button mapping, lamp defaults, image
   ack state, cached countdown, the quiet-hours schedule and gesture queue
   ([#010](010-quiet-hours.md)), and the cumulative energy counter
   ([#015](015-power-metering.md), consumed by [#018](018-home-assistant.md)). **Not
   wiped:** the firmware itself — it stays at its current version; factory reset is not a
   firmware rollback (see [#017 OTA Firmware Updates](017-ota-updates.md)).

## MQTT Topics

None for the reset itself — that's the point of path B's design (see
[mqtt-topics.md](../mqtt-topics.md)). The guided cleanup publishes zero-length retained
messages to the topics listed above.

## Hardware

- Button 1 — 10 s hold triggers path A.
- RGB lamp — hold-progress color ramp; brief red flash on wipe (e-ink progress instead
  during quiet hours, [#010](010-quiet-hours.md)).
- e-ink — shows setup instructions after reboot ([#001](001-setup.md)).

See [hardware.md](../hardware.md).

## States & Edge Cases

- **Hold released early:** the progress ramp resets; nothing happens.
- **Reset requested during an OTA write:** the app path refuses with a clear error; the
  hardware path always wins regardless — the A/B slot table keeps the device bootable
  either way (see [#017 OTA Firmware Updates](017-ota-updates.md)).
- **App path attempted from outside the LAN:** the gadget is simply unreachable —
  expected, not an error state.
- **Partner's stale retained data still shows the wiped person until cleanup runs:** the
  guided flow (path B) handles this; the hardware path (path A) leaves stale retained
  state on the broker — a documented known limitation.
- **Home Assistant residue:** neither path removes the device from HA — the retained
  discovery configs and `ha/*/state` topics stay at the broker, and HA shows the device
  as unavailable until it is removed manually there. Deliberate: factory reset carries
  no HA-specific logic ([#018 Home Assistant Integration](018-home-assistant.md)).
- **Re-provisioning after reset:** requires either a new key ceremony or a re-push of
  existing keys ([#002 Security & Pairing](002-security.md)).
