# Feature: #003 Partner Clock

## Goal

The e-ink always shows your partner's local time next to your own, so you instantly know
whether it's a good moment to reach out.

## Motivation

Time-zone awareness is the most basic form of presence in a long-distance relationship;
checking world clocks on a phone adds friction that this project is meant to remove.

## Behavior

1. The gadget syncs time via NTP (ESP-IDF's SNTP) on boot and periodically thereafter.
2. Your own and your partner's timezone come from the retained `ldr/{you}/config` topic
   — set in the app, cached in NVS on the gadget (see
   [#009 Button Mapping](009-button-mapping.md), which owns the config payload).
3. The e-ink renders both clocks (yours + partner's), updated at most once per minute via
   partial refresh, with a periodic full refresh per the discipline in
   [hardware.md](../hardware.md). During quiet hours the cadence drops to hourly
   ([#010 Quiet Hours / Night Mode](010-quiet-hours.md)).
4. There is no dedicated MQTT topic for the clock itself — both clocks are computed
   locally from synced NTP time plus the configured timezones.

## MQTT Topics

No dedicated topic — see [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Subscribes: `ldr/{you}/config` (own timezone + partner timezone fields).
- Publishes: nothing.

## Hardware

- e-ink (Waveshare 2.9" V2, SPI) — see [hardware.md](../hardware.md).

## States & Edge Cases

- **NTP unreachable on boot:** show `--:--` until synced, retry with backoff.
- **DST transitions:** store IANA timezone names (e.g. `Europe/Berlin`), not fixed UTC
  offsets, so DST is handled correctly without a firmware update.
- **Config not yet received on first boot:** fall back to the NVS-cached config; if none
  exists yet, fall back to a UTC placeholder.
- **Refresh budget:** the e-ink refresh budget is shared with status/connection-quality
  rendering — the clock's own-minute cadence must not starve those.
