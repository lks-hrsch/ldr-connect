# Feature: #004 Reunion Countdown

## Goal

The e-ink shows how many days remain until you next see each other; either partner can
set or change the date from the app.

## Motivation

The countdown is the emotional anchor of every LDR — a number that only shrinks.

## Behavior

1. **Set / change the date:** either app picks a reunion date via a date picker →
   publishes **retained** `ldr/{you}/countdown`, AEAD-encrypted per
   [#002 Security & Pairing](002-security.md); the plaintext inside the envelope is
   `{"date":"2026-08-14","label":"Dresden ✈","ts":…}`.
2. **Shared-value resolution:** both gadgets subscribe to **both** countdown topics —
   their own `ldr/{you}/countdown` **and** the partner's `ldr/{other}/countdown` — and
   render whichever carries the **freshest `ts`**. The reunion date is a single shared
   couple-value with **last-write-wins** semantics, not a per-person value; subscribing
   to one's own topic here is a deliberate exception to the usual
   publish-own / subscribe-partner split.
3. **Rendering:** the e-ink shows "N days" (plus the optional label) in one line or
   corner of the dashboard. The number decrements once per day at **local midnight**,
   each gadget using its own configured timezone
   ([#003 Partner Clock](003-partner-clock.md)) — the two gadgets may disagree by a few
   hours around midnight; this is accepted, not corrected.
4. **Day-of celebration:** on the reunion day, the e-ink shows a celebration state
   ("Today! 🎉" rendered in 1-bit graphics) and the lamp plays a special animation
   **once** ([#005 RGB Lamp](005-rgb-lamp.md)), guarded so a reboot on the day doesn't
   re-trigger it. **During quiet hours** ([#010 Quiet Hours](010-quiet-hours.md)), the
   lamp animation is deferred until wake; the e-ink celebration state is unaffected.
5. **Clearing:** publishing `{"date":null,"ts":…}` removes the countdown from **both**
   e-inks via the same retained-overwrite mechanism.

## MQTT Topics

See [mqtt-topics.md](../mqtt-topics.md) for the full contract.

- Publishes/subscribes: `ldr/{you}/countdown` **and** `ldr/{other}/countdown` (both
  retained, AEAD-encrypted per [#002](002-security.md)).
- Related: [#003 Partner Clock](003-partner-clock.md), [#005 RGB Lamp](005-rgb-lamp.md),
  [#010 Quiet Hours](010-quiet-hours.md).

## Hardware

- e-ink — one dashboard line/corner. See [hardware.md](../hardware.md).
- RGB lamp — day-of celebration animation only.

## States & Edge Cases

- **Date in the past:** treated as "no countdown"; the app validates on entry so this
  shouldn't reach the gadget.
- **Simultaneous conflicting edits:** the freshest `ts` wins; the app shows the current
  effective value live, so a stale edit is visibly overridden.
- **No date set:** the dashboard countdown section is hidden.
- **Timezone edge around midnight:** documented above as an accepted disagreement, not a
  bug.
- **Stale retained value after key rotation:** undecryptable → hidden, with a discreet
  "re-pair needed" state per [#002 Security & Pairing](002-security.md).
