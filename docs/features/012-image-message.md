# Feature: #012 Image Messages

## Goal

Send a picture from your phone to your partner's e-ink; it stays on their display until
they acknowledge it with a button press.

## Motivation

A photo that physically occupies the partner's desk until they've seen it — closer to a
postcard than a chat message.

## Behavior

1. **Sender (app):** pick a photo → crop to **296×128** → convert to 1-bit via
   **Floyd–Steinberg dithering**, with a live preview (the e-ink is monochrome — the app
   preview *is* the received look). The raw bitmap is exactly 296×128/8 = **4,736
   bytes**.
2. **Publish:** wrap the bitmap in the AEAD envelope
   ([#002 Security & Pairing](002-security.md)) and publish it, **retained**, as **one**
   message on `ldr/{you}/image`; the plaintext inside the envelope is
   `{"img":"<base64 4736B bitmap>","ts":…}`. **Size budget:** envelope + base64 ≈ 6.4 KB,
   under `CONFIG_MQTT_BUFFER_SIZE` (8 KB) — no chunking is needed.
3. **Receiver render (takeover mode):** the gadget full-refreshes the image
   full-screen, hiding the normal dashboard, and the lamp plays a notification pulse
   ([#005 RGB Lamp](005-rgb-lamp.md)). The image persists across reboots — takeover state
   is derived from the retained message plus NVS ack tracking, not held in RAM alone.
   **During quiet hours** ([#010 Quiet Hours](010-quiet-hours.md)), the takeover and lamp
   pulse are deferred until wake, rather than firing at night.
4. **Acknowledge:** pressing any mapped ack button (default: button 1, short press,
   during takeover — takeover mode overrides the normal button mapping, see
   [#009 Button Mapping](009-button-mapping.md)) makes the gadget publish
   `ldr/{you}/image_ack` `{"acked_ts":<image ts>,"ts":…}` (QoS 1, **not retained**,
   **plaintext**), store `acked_ts` in NVS, and full-refresh back to the dashboard. The
   sender's app (and optionally a subtle lamp blink) shows "seen".
5. **Replace / reboot semantics:** a newer image replaces an unacknowledged one via
   retained overwrite; an ack always references the `ts` of the image it acknowledges.
   The gadget ignores any retained image whose `ts` is ≤ the NVS `acked_ts`, which is
   what prevents an already-seen image from reappearing after a reboot.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes: `ldr/{you}/image` (retained, encrypted); `ldr/{you}/image_ack` (QoS 1, not
  retained, plaintext).
- Subscribes: `ldr/{other}/image`; the app also observes `ldr/{other}/image_ack`.
- **Why `image_ack` is plaintext:** it carries only `{"acked_ts","ts"}` — timestamps, no
  intimate content — the same reasoning that keeps `ping` plaintext (see
  [mqtt-topics.md § Payload Encryption](../mqtt-topics.md#payload-encryption)). Only the
  image itself is encrypted.
- Related: [#002 Security & Pairing](002-security.md), [#005 RGB Lamp](005-rgb-lamp.md),
  [#008 Interactions](008-interactions.md), [#009 Button Mapping](009-button-mapping.md),
  [#010 Quiet Hours](010-quiet-hours.md), [#017 OTA Firmware Updates](017-ota-updates.md).

## Hardware

- e-ink — full-screen takeover; **single full refresh** on entry and on exit, no
  partial-refresh churn (respects the display's refresh discipline). See
  [hardware.md](../hardware.md).
- Buttons — acknowledge press.
- RGB lamp — notification pulse on arrival, subtle blink on the sender's side once seen.

## States & Edge Cases

- **Receiver offline at send time:** delivered on reconnect, since the topic is
  retained — the opposite trade-off from the transient gestures in
  [#008 Interactions](008-interactions.md), and deliberate here.
- **Duplicate ack under QoS 1:** idempotent — same `acked_ts` re-stored, no re-render.
- **Image arrives during an OTA transfer:** defer the render until OTA completes (see
  [#017 OTA Firmware Updates](017-ota-updates.md)).
- **Image arrives during quiet hours:** defer the takeover and notification pulse until
  wake (see [#010 Quiet Hours](010-quiet-hours.md)).
- **Undecryptable image after key rotation:** skip the render; show the "re-pair needed"
  state ([#002](002-security.md)).
- **Ack button also mapped to an interaction:** takeover mode captures the press for
  acknowledgement; the normal mapping resumes once takeover exits
  ([#009](009-button-mapping.md)).
- **Sender clears the image:** publishing `{"img":null,"ts":…}` clears the retained
  message and exits takeover if the image was still unacknowledged.
