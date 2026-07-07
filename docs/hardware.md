# Hardware

> **Living document.** This reflects current, final decisions. Update it in place as
> parts get swapped or revised — don't fork a new doc.

Both gadgets (one per person) are built identically. There is **no heart-rate sensor on
the gadget** — heart rate comes from the paired iPhone via Apple Watch (see
[architecture.md](architecture.md#heartbeat-data-flow)).

## Bill of materials

| Part | Choice | Bus | Notes |
| --- | --- | --- | --- |
| MCU | ESP32-S3-DevKitC-1-N16R8 | — | 16 MB flash (required for dual OTA slots), 8 MB PSRAM. ~15 €, BerryBase/Reichelt. |
| e-ink | Waveshare 2.9" e-Paper V2, 296×128 | SPI | Supports partial refresh. |
| OLED | SSD1306 0.96" 128×64 | I²C `0x3C` | Upgrade option: SSD1309 2.42", same driver crate. |
| Climate (precision) | SHT45 | I²C `0x44` | ±0.1 °C, ±1 % RH. |
| Climate (pragmatic) | BME280 | I²C `0x76` | Adds pressure. |
| Power metering (precision) | INA226 + 10–20 mΩ external shunt | I²C `0x40` | 16-bit; ±81.9 mV shunt limit. |
| Power metering (pragmatic) | INA219 | I²C `0x40` | Stock 100 mΩ shunt, up to ~3.2 A. |
| RGB lamp | WS2812/SK6812 ring or stick, 8–16 LEDs | RMT | ~60 mA/LED at full white. |
| Input | 2–4 buttons | GPIO | Mapping configurable via retained MQTT config, cached in NVS. |

## MCU gotchas (ESP32-S3-DevKitC-1-N16R8)

- PSRAM must be configured as **OPI PSRAM**.
- Onboard RGB LED is **GPIO38 on board v1.1** (GPIO48 on v1.0) — check board revision.
- **GPIO35/36/37 are unavailable** (used by octal flash/PSRAM).

## Display discipline

- **e-ink**: clock updates at most once per minute; connection quality is shown coarsely.
  Mix partial refreshes with periodic full refreshes to avoid ghosting, and put the panel
  to sleep between updates. During quiet hours the cadence drops to hourly (see
  [features/010-quiet-hours.md](features/010-quiet-hours.md)).
- **OLED**: drives the heartbeat animation — see
  [architecture.md § Display split](architecture.md#display-split) for why this content
  is split across the two displays. Sleeps after at most 10 min of idle animation, and
  throughout quiet hours — burn-in protection, owned by
  [features/010-quiet-hours.md](features/010-quiet-hours.md).

## Climate sensor placement

The climate sensor (SHT45 or BME280) **must be thermally decoupled** from the ESP32,
LEDs, and displays — vent slots, edge placement. Enclosure self-heating otherwise skews
readings by 3–8 °C.

## Power metering placement

High-side, on the **5 V rail directly behind the USB-C jack**, so it measures the entire
device's draw (not just one subsystem).

## RGB lamp brightness cap

At full white, the lamp budget is ~60 mA/LED. Firmware **must cap global brightness to
~30–40 %** to stay within safe power/thermal margins.

## Power input (USB-C)

Single USB-C jack, **5 V only, no USB-PD**.

**CRITICAL:** two **separate** 5.1 kΩ pulldowns, CC1→GND and CC2→GND. Never bridge them —
bridging breaks e-marked cables. Without both pulldowns, the source caps supply current at
500–900 mA, causing brownouts.

## I²C bus

One shared bus, 400 kHz, 4.7 kΩ pullups.

| Address | Device |
| --- | --- |
| `0x3C` | OLED (SSD1306/SSD1309) |
| `0x40` | INA226 / INA219 (power metering) |
| `0x44` | SHT45 (or `0x76` for BME280) |

No address conflicts. In Rust, share the bus via `embedded-hal-bus`'s `RefCellDevice`
(embedded-hal 1.0).

## Rust crate stack

| Crate | Purpose |
| --- | --- |
| `esp-idf-hal` | HAL for ESP32-S3 peripherals |
| `esp-idf-svc` | `EspMqttClient`, `EspOta`, `EspTls` |
| `epd-waveshare` | e-ink driver |
| `ssd1306` + `embedded-graphics` | OLED driver + rendering |
| `ws2812-esp32-rmt-driver` + `smart-leds` | RGB lamp over RMT |
| `sht4x` or `bme280` | Climate sensor driver |
| `ina226` or `ina219` | Power metering driver |
| `embedded-hal-bus` | Shared I²C bus access |
| `serde` / `serde_json` | MQTT payload (de)serialization |

## See also

- [Architecture](architecture.md) — how this hardware fits into the overall system, OTA
  process, and display split rationale.
- [MQTT Topics](mqtt-topics.md) — which topics each sensor/actuator publishes to or is
  driven by (`env`, `power`, `lamp`, `config`).
