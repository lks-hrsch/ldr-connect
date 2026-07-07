# Feature: #010 Quiet Hours / Night Mode

## Goal

The gadget knows when to be silent. During configured quiet hours the lamp stays dark and
the OLED sleeps; incoming gestures are queued rather than lost, and greet you on wake:
"While you slept: 2 hugs."

## Motivation

The product is built on timezone difference — which guarantees notifications landing at
3 a.m. Hardware is the **only** notification channel (no push), so without damping, a
well-meant hug pulses the lamp next to the bed. An ambient device must know when to be
quiet. This feature also owns OLED burn-in protection (display sleep), which applies
beyond night mode.

## Behavior

1. **Schedule config:** quiet hours live in the retained encrypted config topic
   ([#009 Button Mapping](009-button-mapping.md)):
   `{"quiet_hours":{"enabled":true,"start":"22:30","end":"07:00"}}` — local wall-clock
   times evaluated against the gadget's own configured timezone
   ([#003 Partner Clock](003-partner-clock.md)); windows spanning midnight are supported.
   Per-device: each partner configures their own gadget independently.
2. **Manual toggle:** a new mappable button action, `quiet_toggle`
   ([#009](009-button-mapping.md)), flips quiet mode immediately (guests, sick day, nap).
   The manual state holds until the **next** scheduled boundary, then the schedule
   resumes. The app can toggle equivalently via config.
3. **While quiet:** the lamp suppresses all *event-driven* light — notification overlays
   ([#005 RGB Lamp](005-rgb-lamp.md)), celebrations. Two things still play: explicit
   base-state commands from the app or Home Assistant (a deliberately switched-on night
   light is user intent, not a notification — see
   [#018 Home Assistant Integration](018-home-assistant.md)), and local, user-initiated
   feedback — the send-confirmation blink ([#008](008-interactions.md)), the
   factory-reset hold ramp ([#016](016-factory-reset.md)), the HA identify animation
   ([#018](018-home-assistant.md)) — each a direct response to a deliberate action by
   someone present. The OLED is put to sleep (no idle animation — burn-in protection).
   The e-ink keeps updating (non-emissive) but reduces to hourly clock refreshes. Buttons
   remain fully functional — **sending** gestures at night is allowed and unaffected.
4. **Queueing instead of dropping:** incoming interactions
   ([#008 Interactions](008-interactions.md)) are counted per type and persisted to NVS
   (`{"hug":2,"kiss":1}`, plus the `ts` of the last one). Image messages
   ([#012 Image Messages](012-image-message.md)) are retained anyway — the full-screen
   takeover and lamp pulse are **deferred** to wake. The countdown day-of celebration
   ([#004 Reunion Countdown](004-countdown.md)) is likewise deferred to wake.
   Status/presence/env updates continue silently on the e-ink within the reduced refresh
   budget.
5. **Quiet visibility for the partner:** the gadget includes `"quiet":true` in its regular
   retained presence publishes ([#006 Presence](006-presence.md), plaintext topic; the
   static LWT payload is unchanged). The partner's e-ink shows a moon icon next to
   presence — you know your hug will be queued, not shown. **Privacy note:** this exposes
   a coarse sleep flag to the broker in plaintext; accepted because the broker can already
   approximate sleep windows from traffic patterns, and the LWT-must-be-static constraint
   plus topic topology make presence the correct carrier (see
   [#002 Security & Pairing](002-security.md)).
6. **Wake** (schedule end or manual toggle-off): the OLED wakes; the e-ink full-refreshes
   to the dashboard with a summary line — "While you slept: 2 hugs, 1 kiss" — read from
   NVS. If the queue is non-empty, the lamp plays one gentle sunrise animation. The
   summary clears after 30 min or on any button press; a deferred image takeover renders
   after the summary clears.
7. **OLED burn-in protection (always, not only at night):** amends
   [#013 Heartbeat](013-heartbeat.md)'s idle behavior — the idle animation runs for at
   most 10 min after the last fresh BPM sample, then the OLED sleeps until fresh data
   arrives. This rule is owned here and referenced from #013.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract. No new topics — this rides:

- `ldr/{you}/config` — `quiet_hours` block, encrypted ([#009](009-button-mapping.md)).
- `ldr/{you}/presence` — `"quiet"` field, plaintext ([#006](006-presence.md)).

## Hardware

- WS2812 lamp — off during quiet hours; sunrise animation on wake.
- OLED — sleep command (burn-in protection).
- e-ink — moon icon, wake summary, reduced refresh cadence.
- Buttons — `quiet_toggle`, still fully active.
- NVS — gesture queue.

See [hardware.md](../hardware.md).

## States & Edge Cases

- **Time not yet NTP-synced on boot:** the schedule cannot be evaluated, so quiet mode
  stays off; the manual toggle still works. The schedule activates once the clock syncs.
- **Reboot during a quiet window:** the NVS queue survives; the schedule is re-evaluated
  on boot and quiet mode is re-entered.
- **Manual toggle vs. schedule boundary:** the manual state holds until the next
  boundary, then the schedule wins.
- **Queue overflow:** cap the per-type count at 99 and display "99+".
- **Gesture arrives while the wake summary is showing:** the normal animation resumes;
  the summary is already consumed.
- **Summary + deferred image collision:** the summary shows first; the image takeover
  renders after it clears.
- **OTA during quiet hours:** proceeds normally; the e-ink shows the update state, but the
  lamp's "updating" animation is suppressed.
- **DST shift inside the window:** IANA timezone handling per
  [#003 Partner Clock](003-partner-clock.md) covers it.
- **Differing quiet windows between partners:** independent by design — the moon icon
  communicates it.
- **Factory reset** ([#016 Factory Reset](016-factory-reset.md)): wipes the schedule and
  the queue.
