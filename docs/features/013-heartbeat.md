# Feature: #013 Heartbeat

## Goal

See your partner's live heartbeat as an animation on your gadget's OLED while they're
sharing it.

## Motivation

The most intimate ambient signal the project offers; deliberately sourced from Apple
Watch only — there is no on-device heart-rate sensor, a hardware decision (see
[hardware.md](../hardware.md) and
[architecture.md § Heartbeat data flow](../architecture.md#heartbeat-data-flow)).

## Behavior

1. **Sender side:** you open the iOS app in the foreground and start a workout session
   on your Apple Watch. The app uses HealthKit (`HKWorkoutSession` +
   `HKLiveWorkoutBuilder`) to receive HR samples roughly every 1–5 s and publishes
   `ldr/{you}/heartrate` `{"bpm":72,"source":"watch","ts":…}`, at QoS 0, not retained.
2. **Receiver side:** the partner's gadget subscribes to your heartrate. While values are
   fresh (`ts` within a staleness threshold, e.g. 15 s) the OLED renders a heartbeat
   animation paced to the received BPM; when stale, it transitions to a calm idle
   animation.
3. Heartbeat is **intermittent by design** — it only exists while the app is
   foregrounded and a workout is active. This is not a bug or a connectivity gap; it's
   the direct consequence of sourcing HR exclusively from HealthKit.
4. **OLED burn-in protection:** the idle animation itself is not left running forever —
   it runs for at most 10 min after the last fresh BPM sample, then the OLED sleeps until
   fresh data arrives. This rule is owned by
   [#010 Quiet Hours / Night Mode](010-quiet-hours.md) (it applies at all times, not only
   during quiet hours) and is amended here for visibility.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes (app): `ldr/{you}/heartrate`.
- Subscribes (gadget): `ldr/{other}/heartrate`.

## Hardware

- OLED (SSD1306, I²C `0x3C`) — full-frame updates in ~20 ms, well suited to animation.
  See [hardware.md](../hardware.md).

## States & Edge Cases

- **App backgrounded / workout ended mid-stream:** values go stale → idle fallback, no
  error state is shown.
- **Clock skew between phone and gadget:** the gadget paces the animation to the
  received cadence and tolerates a few seconds of skew rather than trying to align to
  absolute time.
- **BPM spikes/dropouts:** smooth animation transitions rather than jump-cutting on
  every sample.
- **Both partners streaming simultaneously:** no conflict — each direction is
  independent.
