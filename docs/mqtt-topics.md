# MQTT Topics

> **Living document.** This reflects current, final decisions. Update it in place as
> topics are added or payloads change — don't fork a new doc.

This is the canonical contract for every message exchanged between the two gadgets, the
two iOS apps, the broker — and, when enabled, the optional Home Assistant client
([#018](features/018-home-assistant.md)). See [architecture.md](architecture.md) for the
components
that publish/subscribe, and [hardware.md](hardware.md) for the sensors and actuators
behind `env`, `power`, and `lamp`.

## Namespace

Every topic lives under one of two prefixes: `ldr/{you}/…` or `ldr/{other}/…`. `{you}` is
your own device's identity; `{other}` is your partner's. Both are real, distinct topic
prefixes — one per person — written here as `{you}`/`{other}` instead of real names so
the docs read the same regardless of who's running which device. All payloads are
**JSON with a unix `ts` field**.

Each client publishes only under its own `ldr/{you}/#` and subscribes to its partner's
`ldr/{other}/#` — plus its own prefix where a feature needs it (see the ACL convention
below) — enforced by broker ACLs (see
[architecture.md § Mosquitto broker](architecture.md#3-mosquitto-broker)). The one topic
tree outside these prefixes is the optional `homeassistant/#` discovery tree used by
[#018 Home Assistant Integration](features/018-home-assistant.md).

## Topics

Every topic kind exists twice — once under each person's prefix. **You publish** under
`ldr/{you}/…` and **you subscribe** to the mirrored `ldr/{other}/…` (the same topic kind,
published by your partner's device). Both concrete topics are listed for every row below.

| You publish | You subscribe (partner's mirror) | Retained? | Publisher | Payload example | Purpose |
| --- | --- | --- | --- | --- | --- |
| `ldr/{you}/presence` | `ldr/{other}/presence` | yes + LWT | gadget | `{"online":true,"quiet":true,"ts":…}` | Device online/offline, via retained state + Last Will; optional `"quiet"` flag (plaintext) reflects Quiet Hours — see [#010 Quiet Hours](features/010-quiet-hours.md) |
| `ldr/{you}/heartrate` | `ldr/{other}/heartrate` | no | iOS app (HealthKit/Watch, foreground + workout only) | `{"bpm":72,"source":"watch","ts":…}` | Partner OLED animates BPM while fresh; falls back to idle animation when stale |
| `ldr/{you}/status` | `ldr/{other}/status` | yes | app or gadget | `{"mood":"home_bored","text":"call me","ts":…}` | Mood/status text shown on partner e-ink |
| `ldr/{you}/interaction` | `ldr/{other}/interaction` | no | gadget (button) or app | `{"type":"hug","ts":…}` | Hug/kiss events → lamp + e-ink on partner side |
| `ldr/{you}/env` | `ldr/{other}/env` | yes | gadget | `{"t":21.5,"rh":45,"ts":…}` (add `"p"` if BME280) | Climate telemetry |
| `ldr/{you}/power` | `ldr/{other}/power` | yes | gadget | `{"v":5.02,"i":0.34,"w":1.7,"ts":…}` | Device power draw |
| `ldr/{you}/config` | `ldr/{other}/config` | yes | app → gadget | `{"buttons":{"1":"hug","2":"kiss"},"lamp_defaults":{…},"tz":{…},"quiet_hours":{"enabled":true,"start":"22:30","end":"07:00"},"ha":{"enabled":false,"climate_export":false},"schema":1,"ts":…}` | Gadget config (button mapping, lamp defaults, timezone, quiet hours schedule — see [#010](features/010-quiet-hours.md) — and the HA opt-in block, [#018](features/018-home-assistant.md)); gadget caches it in NVS |
| `ldr/{you}/ping` | `ldr/{other}/ping` | no | gadget | `{"rtt_ms":42,"quality":"good\|poor\|offline","ts":…}` | Connection-quality heartbeat |
| `ldr/{you}/lamp` | `ldr/{other}/lamp` | yes | app → gadget | `{"r":255,"g":80,"b":40,"bri":30,"mode":"solid","ts":…}` | Lamp control |
| `ldr/{you}/ota/offer` | `ldr/{other}/ota/offer` | yes | app → gadget | `{"version":"1.2.3","size":1048576,"sha256":"…","chunks":512,"chunk_size":2048,"ts":…}` | Retained OTA offer — see [architecture.md § OTA over MQTT](architecture.md#ota-over-mqtt) |
| `ldr/{you}/ota/chunk` | `ldr/{other}/ota/chunk` | no | app → gadget | raw binary, 2–4 KB, QoS 1 | OTA firmware chunk stream |
| `ldr/{you}/ota/status` | `ldr/{other}/ota/status` | no | gadget → app | `{"state":"writing","recv":128,"total":512,"ts":…}` | OTA write progress — see [ota-updates.md](features/017-ota-updates.md) |
| `ldr/{you}/countdown` | `ldr/{other}/countdown` | yes | app | `{"date":"2026-08-14","label":"Dresden ✈","ts":…}` | Shared reunion date; freshest `ts` wins — see [countdown.md](features/004-countdown.md) |
| `ldr/{you}/image` | `ldr/{other}/image` | yes | app | `{"img":"<base64 4736B bitmap>","ts":…}` | 1-bit e-ink image message (~6.4 KB, fits the 8 KB buffer, no chunking) — see [image-message.md](features/012-image-message.md) |
| `ldr/{you}/image_ack` | `ldr/{other}/image_ack` | no | gadget | `{"acked_ts":1735689600,"ts":…}` | Acknowledges an image message, by its `ts` — see [image-message.md](features/012-image-message.md) |

## Home Assistant Subtree (opt-in)

When [#018 Home Assistant Integration](features/018-home-assistant.md) is enabled, the
gadget dual-publishes a curated, **plaintext** operational surface under
`ldr/{you}/ha/…`, alongside retained MQTT Discovery configs under `homeassistant/…` —
outside the `ldr` namespace, per the Home Assistant discovery convention. Summary by
entity group:

| Topic(s) | Direction | Retained? | Entity |
| --- | --- | --- | --- |
| `ha/lamp/state`, `ha/lamp/set` | gadget ↔ HA | state: yes | light — mirrors the base-state lamp layer |
| `ha/power/state`, `ha/energy/state` | gadget → HA | yes | sensor — power (W), cumulative energy (kWh) |
| `ha/climate/state` | gadget → HA | yes | sensor — only if `climate_export=true` |
| `ha/rssi/state` | gadget → HA | yes | sensor — WiFi RSSI, diagnostic |
| `ha/quiet/state`, `ha/quiet/set` | gadget ↔ HA | state: yes | switch — quiet-hours toggle |
| `ha/identify/press` | HA → gadget | no | button — identify animation |
| `ha/presence/set` | HA → gadget | no | HA-sourced home/away context |
| `homeassistant/…` | gadget → HA | yes | MQTT Discovery configs (per-entity, outside the `ldr` namespace) |

See [#018 Home Assistant Integration](features/018-home-assistant.md) for the full
entity table, discovery flow, and ACL scope.

## Conventions

- **Encoding:** JSON everywhere, except `ota/chunk` which is raw binary (application
  payload, not JSON-wrapped).
- **`ts`:** every payload carries a unix timestamp; consumers use it for staleness checks
  (most notably `heartrate` — see
  [architecture.md § Heartbeat data flow](architecture.md#heartbeat-data-flow)).
- **Retained vs. LWT:** retained = last-known state for a topic (config, lamp, env,
  power, status, countdown, image, ota/offer); LWT on `presence` is how the broker
  reports a gadget going offline unexpectedly.
- **ACLs:** each device/app may **write** only under its own `ldr/{you}/#`, and **read**
  both its own `ldr/{you}/#` and the partner's `ldr/{other}/#` — enforced by the broker,
  not the clients. Own-topic read is required, not incidental: the gadget subscribes to
  its own `config` and `lamp`, both gadgets subscribe to their own `countdown`
  ([#004](features/004-countdown.md)), and `ping` is a self-loopback
  ([#014](features/014-connection-quality.md)). **Exception:** `lamp` is writable
  cross-partner — the app is permitted to write `ldr/{other}/lamp` as well, so you can
  set your partner's lamp color/mode directly (see
  [rgb-lamp.md](features/005-rgb-lamp.md)).

## Payload Encryption

Eight topics carry intimate content and are end-to-end encrypted rather than plain JSON:
`status`, `interaction`, `heartrate`, `env`, `lamp`, `config`, `countdown`, `image`. Each
payload is wrapped in a ChaCha20-Poly1305 AEAD envelope:

```json
{"v": 1, "kid": 1, "n": "<base64 96-bit nonce>", "ct": "<base64 ciphertext>"}
```

The original JSON (including `ts`) lives inside `ct`; **AAD is the full topic name**,
binding each ciphertext to the topic it was published on. `presence`, `ping`, and
`ota/*` stay plaintext: `presence`'s LWT payload must be static so the broker can deliver
it without a live client, and `ota/*` integrity is already covered by the SHA-256 carried
in the offer. `image_ack` is likewise plaintext — it carries only `{"acked_ts","ts"}`,
timestamps with no intimate content, the same reasoning as `ping`. See
[security.md](features/002-security.md) for key generation, the pairing ceremony, and
rotation.

The `ha` subtree (see [Home Assistant Subtree](#home-assistant-subtree-opt-in) above) is
a separate, deliberate plaintext export surface for Home Assistant — never a leak of the
encrypted topics above — see
[#018 Home Assistant Integration](features/018-home-assistant.md).

## See also

- [Architecture](architecture.md) — system overview, OTA process, broker ACL model.
- [Hardware](hardware.md) — sensors and actuators behind `env`, `power`, `lamp`.
- [Security & Pairing](features/002-security.md) — payload encryption, key ceremony.
- [Reunion Countdown](features/004-countdown.md) — shared countdown resolution rule.
- [Image Messages](features/012-image-message.md) — image size budget, ack flow.
- [Quiet Hours / Night Mode](features/010-quiet-hours.md) — `quiet` presence flag,
  `quiet_hours` config schedule.
- [Home Assistant Integration](features/018-home-assistant.md) — the `ha` subtree,
  MQTT Discovery, ACL scope.
