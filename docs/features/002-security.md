# Feature: #002 Security & Pairing

## Goal

All intimate data between the two of you is end-to-end encrypted — the broker only ever
relays ciphertext. Identity is established once per person via a key ceremony: generate a
key by shaking your phone, push it to your own gadget, then pair with your partner by
scanning each other's QR code.

## Motivation

The broker is exposed to the internet. TLS protects the transport, but a compromised or
curious broker could still read payloads. End-to-end encryption makes the broker
zero-knowledge for intimate content (status, gestures, heartbeat, climate). The ceremony —
shaking, then scanning each other's code — doubles as an intentional, emotional pairing
ritual that fits the product, not just a security step.

## Behavior

1. **Key generation (iOS app):** generate a Curve25519 keypair via CryptoKit. An entropy
   ritual precedes it — the user shakes the phone for ~10 s; accelerometer samples are
   hashed (SHA-256) and mixed with `SecRandomCopyBytes` output via HKDF into the seed.
   **The iOS CSPRNG alone is already cryptographically sufficient — shaking is entropy
   augmentation and UX ritual, never the sole source.**
2. **Key storage:** the private key exists ONLY on the person's own devices — iOS
   Keychain on the phone, and pushed once to the own gadget over the local provisioning
   channel (see [#001 Setup & Provisioning](001-setup.md)), stored in NVS. ESP32 flash /
   NVS encryption is optional hardening on top of that. The private key never touches the
   broker, the partner, or any QR code.
3. **Pairing ceremony:** person A's app displays a QR code containing their person id,
   Curve25519 **public** key, and a short fingerprint. Person B scans it with their app's
   camera; then the roles swap (B shows, A scans). **Private keys never appear in any
   QR** — this is a public-key exchange only.
4. **Shared-key derivation:** both sides compute the X25519 ECDH shared secret from (own
   private key, partner public key), then HKDF-SHA256 → a 256-bit symmetric key. Before
   activation, both apps display the same emoji/word fingerprint; partners confirm it
   matches — by call or in person — as a verification step. The app then pushes the
   derived key material to the own gadget, either via the [#001](001-setup.md)
   provisioning channel or a later re-provisioning pass.
5. **Payload encryption:** intimate topics carry a ChaCha20-Poly1305 AEAD envelope instead
   of plaintext JSON:

   ```json
   {"v": 1, "kid": 1, "n": "<base64 96-bit nonce>", "ct": "<base64 ciphertext>"}
   ```

   The plaintext (the original JSON payload, including `ts`) lives inside `ct`. **AAD is
   the full topic name**, binding the ciphertext to the topic it was published on and
   preventing cross-topic replay. Encrypted topics: `status`, `interaction`, `heartrate`,
   `env`, `lamp`, `config`, `countdown`, `image`. Plaintext topics: `presence` (the LWT
   payload must be static so the broker can deliver it without a live client), `ping`,
   `image_ack` (timestamps only, no intimate content — same reasoning as `ping`), and
   `ota/*` (firmware integrity is already covered by the SHA-256 in the offer; signing
   the offer itself is noted as possible future work). The optional Home Assistant
   subtree `ldr/{you}/ha/#` is a deliberate plaintext export surface, not an exception to
   this rule — see [#018 Home Assistant Integration](018-home-assistant.md). The same
   envelope also authenticates the local
   management channel described in [#001 Setup & Provisioning](001-setup.md), with
   **AAD = the HTTP request path** instead of a topic name.
6. **Crypto on the gadget (Rust):** `x25519-dalek` + `chacha20poly1305` + `hkdf`
   (RustCrypto crates) — pure Rust, and they work fine in the `esp-idf` std environment.
7. **Rotation / re-pairing:** re-run the ceremony. The `kid` (key id / epoch) field in the
   envelope lets receivers handle the changeover gracefully — messages under the old key
   are rejected once both sides have confirmed the new epoch.

## MQTT Topics

No dedicated topics. This feature changes the **payload format** of the topics listed
above — see [mqtt-topics.md](../mqtt-topics.md) for the underlying contract each one
still follows (retained flags, publishers, purpose). Retained encrypted messages work
unchanged, since AEAD is stateless and decryptable after a reboot.

## Hardware

Nothing on the gadget beyond NVS key storage (see [hardware.md](../hardware.md)); the
ceremony uses the phone's accelerometer and camera. ESP32 flash encryption is an optional
hardening step, not a requirement.

## States & Edge Cases

- **Lost or replaced phone:** generate a new keypair and re-run the ceremony — this is
  revocation by rotation, there is no separate revocation flow.
- **Replaced gadget:** re-provision the key via [#001](001-setup.md).
- **Nonce handling:** a random 96-bit nonce per message; at this message volume, collision
  risk is negligible.
- **Replay protection:** AAD topic (or path) binding, plus the `ts` freshness window
  inside the plaintext.
- **Scanning the wrong person's QR:** the fingerprint check fails to match → abort before
  activation.
- **Ceremony interrupted halfway:** no partial activation — a key only activates once
  both directions have scanned and the fingerprint has been confirmed.
- **Gadget offline during key rotation:** the new key applies on the gadget's next
  config/provisioning contact. Until then, partner messages fail to decrypt — the gadget
  shows a discreet "re-pair needed" state instead of rendering garbage.
- **LAN attacker probing the local management endpoint** (see [#001](001-setup.md)):
  only `/identify` is readable without authentication; any mutating request requires the
  shared key.
