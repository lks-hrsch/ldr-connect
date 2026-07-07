# Feature: #001 Setup & Provisioning

## Goal

From unboxing to online in one guided app flow: a blank gadget gets WiFi, broker address,
TLS trust, identity, and keys — no cables, no hardcoded secrets, doable by a non-technical
partner in another city.

## Motivation

The second device lives with your partner, far away. The repo is public, so **no
secrets** — WiFi credentials, broker credentials, keys — may ever be baked into firmware;
everything personal has to arrive via provisioning.

## Behavior

1. **Setup mode:** on first boot, when NVS holds no config, or after a factory reset, the
   gadget enters setup mode. It opens a WPA2 SoftAP named `ldr-connect-XXXX` (`XXXX` from
   the MAC address) with a random per-device password. The e-ink shows setup instructions
   plus a QR code encoding the AP name and password; the lamp breathes blue.
2. **App wizard:** the iOS app's setup wizard scans the gadget's QR code, joins the AP
   programmatically via `NEHotspotConfiguration`, and talks to the gadget's local HTTP
   server (`esp-idf-svc`'s `EspHttpServer`, e.g. `192.168.71.1`) over the closed AP link.
3. **Provisioning payload (single JSON POST):** home WiFi SSID and password; broker
   host/port plus CA certificate (or a pinned fingerprint); per-device MQTT credentials;
   person identity; a timezone pair; initial button mapping and lamp defaults; and the key
   material from [#002 Security & Pairing](002-security.md) — own private key and partner
   public / derived key. **This is the one moment keys transit a link** — a closed,
   WPA2-protected, local AP with no path to the internet.
4. **Apply & verify:** the gadget persists everything atomically to NVS as a versioned
   blob, replies OK, shuts the AP down, connects to home WiFi, then to the broker over
   TLS, and publishes retained presence. The app leaves the AP, reconnects to the
   internet, and verifies end-to-end by observing the retained presence via the broker —
   the wizard then shows success. The lamp flashes green; the e-ink switches to the
   normal dashboard.
5. **Local management channel (post-setup):** after provisioning, the gadget keeps its
   local HTTP server running — now in STA mode on the home WiFi — and announces itself
   via mDNS/Bonjour: service type `_ldr-connect._tcp`, hostname `ldr-connect-XXXX.local`
   (`XXXX` matching the SoftAP name), with TXT records for person and firmware version.
   The iOS app discovers it via `NWBrowser`.

   **Discovery decision:** mDNS was chosen over ARP/subnet scanning — iOS provides no
   raw-socket/ARP access, Apple's local-network privacy model is built around declared
   Bonjour services, and mDNS survives DHCP address changes by resolving the `.local`
   name. ARP scanning was considered and rejected.

   **Purpose:** this channel exists for everything MQTT cannot fix without a working
   broker connection — changing WiFi credentials (while the old WiFi still works),
   changing the broker host/CA, or re-provisioning keys ([#002](002-security.md)
   rotation). Routine settings still ride the retained `ldr/{you}/config` topic
   ([#009 Button Mapping](009-button-mapping.md)); the SoftAP remains the bootstrap path
   only, and this local channel is the maintenance path.

   **Authentication is mandatory:** on the home LAN the endpoint is reachable by any
   device, so every mutating request MUST be wrapped in the same ChaCha20-Poly1305 AEAD
   envelope used for MQTT payloads (key material from [#002](002-security.md), with
   **AAD = the request path**). Unauthenticated access is limited to a read-only
   `GET /identify`, returning `{"name": …, "person": …, "fw_version": …}`. A wrong or
   missing envelope returns `401` with no further detail — see
   [#002 Security & Pairing](002-security.md).

   **iOS requirements:** `NSBonjourServices` (`_ldr-connect._tcp`) and
   `NSLocalNetworkUsageDescription` in Info.plist; first discovery triggers the iOS
   local-network permission prompt.

   **Firmware note:** mDNS is a managed component in ESP-IDF 5.x — add it via
   `esp-idf-sys` extra components (`EspMdns` from `esp-idf-svc`).
6. **Later changes:** routine settings (button mapping, lamp defaults, timezones) ride the
   normal retained `ldr/{you}/config` topic (#009) over MQTT. The SoftAP channel is
   bootstrap-only — needed again only for what MQTT can't fix without a working
   connection: WiFi change, broker change, key re-provisioning.
7. **Re-enter setup / factory reset:** a button-1 hold (or the app's guided flow) wipes
   NVS and returns the gadget to setup mode — see
   [#016 Factory Reset](016-factory-reset.md) for the full spec.
8. **Decision note:** SoftAP + local HTTP was chosen over BLE provisioning because
   `esp-idf-svc` has mature AP and HTTP-server support in Rust std, while BLE
   (`esp32-nimble`) would add a second wireless stack for a one-time flow. BLE is noted
   as a possible future alternative.

## MQTT Topics

None during setup — that is the point. Post-setup, configuration uses
`ldr/{you}/config` ([#009 Button Mapping](009-button-mapping.md)); the first successful
connection publishes presence ([#006 Presence](006-presence.md)). See
[mqtt-topics.md](../mqtt-topics.md) for the full contract.

## Hardware

e-ink (setup instructions and QR code), RGB lamp (state signaling: breathing blue during
setup, green flash on success, red on error), button 1 (factory-reset hold), NVS. See
[hardware.md](../hardware.md).

## States & Edge Cases

- **Wrong home-WiFi password:** the connection attempt fails; after a timeout the gadget
  returns to setup mode, and the e-ink shows a specific "WiFi failed" hint.
- **WiFi ok but broker unreachable:** a distinct error state on the e-ink — credentials
  vs. network problems are diagnosable separately.
- **App loses the AP link mid-provisioning:** the POST is idempotent, so retrying is
  safe.
- **User declines the `NEHotspotConfiguration` prompt:** the wizard falls back to a
  manual-join screen showing the AP name and password from the QR code.
- **Provisioning timeout in setup mode:** the AP stays up; there is no auto-exit.
- **Partial NVS write:** the versioned blob is written atomically — the previous config
  stays valid until a full new blob commits.
- **Re-running setup on an already-configured device:** only reachable via the
  deliberate button hold.
- **Gadget QR damaged or unreadable on e-ink:** the AP name is also printed as plain
  text, as a fallback.
- **Gadget not discoverable via mDNS:** AP isolation or a guest-WiFi network can block
  it; the app falls back to manual IP entry.
- **Local-network permission denied:** the wizard explains how to re-enable it in iOS
  Settings.
- **Stale mDNS cache after an IP change:** resolve on connect, retry once.
- **Concurrent config via MQTT and the local channel:** last-write-wins by `ts` — the
  same rule used for NVS vs. retained config.
