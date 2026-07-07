# Features

> **Living document.** Update this index whenever a feature doc is added or renamed —
> don't fork a new doc.

One doc per user-facing feature, each following [`_template.md`](_template.md). Start
there when adding a new feature; give it the next number in sequence.

| # | Feature | Summary |
| --- | --- | --- |
| 001 | [Setup & Provisioning](001-setup.md) | Unboxing-to-online SoftAP wizard, plus a local mDNS channel for later maintenance. |
| 002 | [Security & Pairing](002-security.md) | End-to-end encrypted payloads, established via a shake-to-generate, scan-to-pair key ceremony. |
| 003 | [Partner Clock](003-partner-clock.md) | e-ink shows your partner's local time next to your own. |
| 004 | [Reunion Countdown](004-countdown.md) | Days-until-we-meet countdown on the e-ink; either partner sets the shared date. |
| 005 | [RGB Lamp](005-rgb-lamp.md) | Friendship-lamp color control + the project's only notification channel. |
| 006 | [Presence](006-presence.md) | Lightweight "I'm here" signal via retained state + LWT. |
| 007 | [Status & Mood](007-status-mood.md) | Share a mood/status ("home & bored", "call me") on the partner's e-ink. |
| 008 | [Interactions (Hug / Kiss)](008-interactions.md) | Button press → partner's lamp lights up with a physical gesture. |
| 009 | [Button Mapping](009-button-mapping.md) | Configure what each physical button does, from the app. |
| 010 | [Quiet Hours / Night Mode](010-quiet-hours.md) | Configurable silent hours: lamp/OLED off, gestures queued and replayed on wake. |
| 011 | [Room Climate](011-environment.md) | Temperature/humidity telemetry from your partner's place. |
| 012 | [Image Messages](012-image-message.md) | Send a 1-bit photo to your partner's e-ink; it stays until they press to acknowledge. |
| 013 | [Heartbeat](013-heartbeat.md) | Apple Watch heart rate, via HealthKit, animated on the partner's OLED. |
| 014 | [Connection Quality](014-connection-quality.md) | Self-diagnostic link-health icon on the e-ink. |
| 015 | [Power Metering](015-power-metering.md) | Live device power draw and brownout protection. |
| 016 | [Factory Reset](016-factory-reset.md) | Defined wipe path — button hold or guided app flow — back to out-of-box state. |
| 017 | [OTA Firmware Updates](017-ota-updates.md) | Firmware updates over MQTT, with no server beyond the broker. |
| 018 | [Home Assistant Integration](018-home-assistant.md) | Opt-in HA bridge: lamp as light entity, energy dashboard export, home/away context in — intimate data never exposed. |

## See also

- [Architecture](../architecture.md) — system overview.
- [Hardware](../hardware.md) — per-gadget bill of materials.
- [MQTT Topics](../mqtt-topics.md) — the full topic/payload contract.
