# Feature: #017 OTA Firmware Updates

## Goal

Update the gadget's firmware from the iOS app over MQTT — no cables, no extra server.

## Motivation

The broker must stay the only server component, and your partner's device (in another
city) needs to be updatable remotely. See
[architecture.md § OTA over MQTT](../architecture.md#ota-over-mqtt) for the system-level
rationale.

## Behavior

1. The app selects the `.bin` (built via `espflash save-image` — **not** the ELF) and
   computes its SHA-256, then publishes retained `ldr/{you}/ota/offer`
   `{"version":"1.4.0","size":…,"sha256":"…","chunks":…,"chunk_size":2048,"ts":…}`.
2. The gadget compares the offered version against its own, then calls
   `EspOta::initiate_update()` targeting the inactive partition slot.
3. The app streams sequence-numbered raw binary chunks (2–4 KB, QoS 1) on
   `ldr/{you}/ota/chunk`; the gadget writes each chunk straight to flash via
   `update.write()` — the full image is never buffered in RAM.
4. The gadget reports progress on `ldr/{you}/ota/status`
   `{"state":"writing","recv":…,"total":…,"ts":…}`.
5. After the last chunk: SHA-256 verify → `complete()` → restart.
6. The new firmware self-tests (WiFi + MQTT reconnect), then calls
   `mark_running_slot_valid()`. `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` reverts
   automatically if that validation call never happens.

**Prerequisites:** a dual-OTA partition table (two app slots on 16 MB flash), and
`CONFIG_MQTT_BUFFER_SIZE=8192` to accommodate the chunk size.

**Fallback:** HTTPS-OTA is the explicit fallback if MQTT-OTA proves unstable — in that
case the offer would carry a URL instead of chunks. Decision threshold: repeated chunk
instability/failures in practice.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- `ldr/{you}/ota/offer` — retained, app → gadget.
- `ldr/{you}/ota/chunk` — QoS 1, app → gadget.
- `ldr/{you}/ota/status` — gadget → app, progress reporting.

## Hardware

- No dedicated sensor; uses the MCU's flash and dual OTA partitions (see
  [hardware.md](../hardware.md)).
- The lamp shows a distinct "updating" animation during the transfer (see
  [#005 RGB Lamp](005-rgb-lamp.md)); the e-ink shows the update state. During quiet
  hours the lamp animation is suppressed but the e-ink update state is not
  ([#010 Quiet Hours](010-quiet-hours.md)).

## States & Edge Cases

- **Chunk gap/timeout:** request a resume via the status message, or abort after a
  timeout via `abandon_update`.
- **Duplicate chunks under QoS 1:** dedupe by sequence number.
- **SHA mismatch:** abort and keep the currently running slot.
- **Power loss mid-write:** the old slot still boots — inherent to the A/B partition
  scheme.
- **App backgrounded mid-transfer:** the transfer pauses; it can resume or restart.
- **Version downgrade:** allowed, but flagged in the app UI.
