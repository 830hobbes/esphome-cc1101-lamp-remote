# ESP32 + CC1101 → Gaonitz 433 MHz LED Lamp Remote (ESPHome)

Control a cheap 433.92 MHz RF LED lamp from Home Assistant using an ESP32 and a
CC1101 radio, via ESPHome. This repo contains a working ESPHome config plus the
reverse-engineered protocol details, so you can use it directly (if you have the
same remote) or adapt it to your own 433 MHz OOK device.

**Target device:** Gaonitz "Round 6-button touch remote control," model
**CLED-5501C**, **FCC ID [2BD2C-ZZ02DY01007](https://fcc.report/FCC-ID/2BD2C-ZZ02DY01007)**
(Dongguan Gaonitz Technology Co., Ltd.). Buttons: Power, Night Light,
Max Brightness, Brightness, Color Temperature, Timing.

> The FCC test report at the link above independently confirmed the pulse widths
> (~0.74 ms / ~0.25 ms) and the 33-pulse-per-repeat structure. If you're
> reverse-engineering a similar remote, its FCC filing is a genuinely useful
> cross-check.

---

## What this does

- The **CC1101** provides the 433.92 MHz RF carrier and does the ASK/OOK keying.
- The **ESP32 RMT peripheral** (through ESPHome's `remote_transmitter`) generates
  the OOK envelope on the CC1101's `GDO0` data-in pin.
- Six **template buttons** are exposed to Home Assistant, one per remote key.

This is a **transmit-only** bridge (no receive path).

---

## Hardware

### Bill of materials

| Qty | Item | Notes |
| :-- | :--- | :--- |
| 1 | ESP32 dev board | Verified on a HiLetgo **ESP32-WROOM-32U DevKit v4**; most ESP32 boards work |
| 1 | CC1101 433 MHz module | The common 8-pin breakout; must be the **433 MHz** variant |
| 1 | 433 MHz antenna | Or ~17.3 cm of solid-core wire (¼-wave whip) |
| 7 | jumper wires | female–female for header-pin modules |
| 1 | USB cable | to flash/power the ESP32 |

### Wiring diagram

![Wiring diagram](docs/wiring-diagram.svg)

### Connections

| CC1101 pin | → | ESP32 pin | Function | YAML key |
| :--- | :-: | :--- | :--- | :--- |
| VCC | → | **3.3 V** | Power — **3.3 V only** | — |
| GND | → | GND | Ground | — |
| CSN / CS | → | GPIO5 | SPI chip select | `cs_pin` |
| SCK / CLK | → | GPIO18 | SPI clock (VSPI) | `spi.clk_pin` |
| MOSI | → | GPIO23 | SPI MOSI (VSPI) | `spi.mosi_pin` |
| MISO | → | GPIO19 | SPI MISO (VSPI) | `spi.miso_pin` |
| GDO0 | → | GPIO4 | OOK data in (RMT-driven) | `data_pin` |
| GDO2 | → | — | unconnected (TX-only) | — |

> ⚠️ **Power:** the CC1101 is a **3.3 V** part. Connecting VCC to 5 V will
> permanently damage it. Use the ESP32's 3V3 pin, not VIN/5V.

> ⚠️ **Antenna:** solder a 433 MHz antenna (or ~17.3 cm wire) to the CC1101's
> ANT pad. A bare module transmits weakly — fine for a nearby SDR, often too weak
> to reach the target device.

### Hardware setup steps

1. **Power off** — don't wire anything with USB plugged in.
2. Connect **GND first**, then **VCC to the ESP32 3V3 pin** (double-check it's 3V3, not 5V/VIN).
3. Wire the four SPI lines: CSN→GPIO5, SCK→GPIO18, MOSI→GPIO23, MISO→GPIO19.
4. Wire **GDO0→GPIO4** (this is the data line the ESP32 keys). Leave GDO2 unconnected.
5. Attach the antenna to the CC1101 ANT pad.
6. Plug in USB and proceed to flashing.

Pins are configurable — `GPIO4` for data and the SPI pins are just the ESP32
VSPI defaults; any usable output GPIOs work as long as the YAML `substitutions`
and `spi:` block match your wiring. Avoid ESP32 input-only pins (34–39) and be
mindful of strapping pins. Full detail in [`docs/WIRING.md`](docs/WIRING.md).

---

## Quick start (software)

1. Install ESPHome (**≥ 2025.11**, required for the native `cc1101` component).
2. Copy [`esphome/rf-lamp-bridge.yaml`](esphome/rf-lamp-bridge.yaml) into your
   ESPHome config directory.
3. Create `secrets.yaml` from [`secrets.yaml.example`](secrets.yaml.example) and
   fill in your Wi-Fi credentials.
4. If your wiring differs from the table above, edit the `substitutions:` block
   (`cs_pin`, `data_pin`) and the `spi:` block at the top of the YAML to match.
5. Flash: `esphome run esphome/rf-lamp-bridge.yaml` (USB for the first flash,
   OTA thereafter).
6. Adopt the device in Home Assistant — six buttons appear. Press one near the
   lamp.

If the buttons transmit (verify with an SDR) but the lamp doesn't respond, see
[`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) — start with polarity.

### Verification status

Being precise about what's been proven on hardware:

- **Power, Night Light, Max Brightness, Brightness, Color Temperature** —
  confirmed working on the physical lamp.
- **Timing** — transmits correctly (verified against the real remote's capture via
  RTL-SDR), but its effect is unconfirmed: it isn't clear what this button is
  supposed to do on the lamp, and no observable response was seen from **either**
  the original remote or this bridge. The RF is right; the behavior is unknown.
  Possibly a sleep/countdown feature that needs a particular state or a press
  sequence, or a function this lamp model doesn't implement.
- **Frame encoding** (inverted polarity, 1000 µs bit period, preamble, 5 repeats)
  — confirmed working on hardware.

If you know what the Timing button does on this lamp family, an issue or PR
clarifying it would be welcome.

---

## Adapting to your own remote

If your lamp/remote is different, the payloads here won't match — but the method
will. See [`docs/PROTOCOL.md`](docs/PROTOCOL.md) for the full protocol and
[`docs/HOW-TO-REVERSE-ENGINEER.md`](docs/HOW-TO-REVERSE-ENGINEER.md) for the
RTL-SDR / `rtl_433` workflow used to capture and decode it.

**The single most important lesson** (it cost the most debugging time): an
RTL-SDR / `rtl_433` demodulator **normalizes polarity away**, so a clean bit
decode does **not** prove your transmitted polarity is correct. This remote's
OOK is **inverted** relative to a naive reading. If your SDR decodes your ESP32's
output perfectly but the device still ignores it, **suspect polarity first** —
swap HIGH/LOW and try again.

---

## Files

```
esphome/rf-lamp-bridge.yaml     Complete, working ESPHome config (6 buttons)
docs/PROTOCOL.md                Full RF protocol spec + per-button codes
docs/WIRING.md                  Wiring detail and antenna notes
docs/wiring-diagram.svg         Wiring diagram (rendered in the README)
docs/HOW-TO-REVERSE-ENGINEER.md RTL-SDR + rtl_433 capture/decode walkthrough
docs/TROUBLESHOOTING.md         Symptom → cause table from a long debug session
```

---

## Credits & license

Reverse-engineered from RTL-SDR captures and the device's public FCC filing.
Released under the [MIT License](LICENSE). No affiliation with Gaonitz or the FCC.
