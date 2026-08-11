# Protocol — Gaonitz CLED-5501C 433 MHz remote

Reverse-engineered from RTL-SDR captures of the original remote, cross-checked
against the device's FCC test report
([FCC ID 2BD2C-ZZ02DY01007](https://fcc.report/FCC-ID/2BD2C-ZZ02DY01007)).

## RF parameters

| Parameter | Value |
| :--- | :--- |
| Frequency | 433.92 MHz |
| Modulation | ASK/OOK |
| Encoding | Pulse-width (PWM), **inverted polarity** |
| Bit period | **constant 1000 µs** (mark + space always sum to ~1 ms) |
| `1` bit | 240 µs HIGH, 760 µs LOW |
| `0` bit | 760 µs HIGH, 240 µs LOW |
| Preamble | ten `1` bits; the tenth bit's LOW period is stretched into a 6500 µs sync gap |
| Payload | 33 bits, transmitted **5×** |
| Inter-repeat gap | 6500 µs |

### Polarity (read this)

The signal is **inverted**. `rtl_433`'s PWM slicer will happily decode either
polarity to a clean bitstream, so the SDR is *not* a reliable check of on-air
polarity. The lamp only responds to the inverted keying shown above. In the raw
ESPHome timing arrays this looks like `240, -760` for a `1` and `760, -240`
for a `0` (positive = carrier ON, negative = carrier OFF).

### Constant bit period (also read this)

Every bit occupies the same ~1000 µs whether it's a `1` or a `0`; only the
HIGH/LOW split changes. A naive "fixed short gap after every mark" scheme
produces bit periods that vary ~2× and, while it still decodes on an SDR, will
not drive the lamp.

## Frame structure

```
[preamble: 10 × "1" bits, last LOW stretched to 6500 µs sync]
[payload 33 bits] [6500 µs sync]
[payload 33 bits] [6500 µs sync]
[payload 33 bits] [6500 µs sync]
[payload 33 bits] [6500 µs sync]
[payload 33 bits] [6500 µs sync]
```

In raw timing terms the preamble is nine `240, -760` pairs followed by
`240, -6500` — i.e. **ten** 240 µs marks, with the tenth bit's low period
serving as the sync gap. (The count observed on the real remote wobbled
between 9 and 10 identical marks across captures — consistent with a carrier
warm-up burst rather than a precise data field. Ten is what the verified
config sends; if adapting, small variations in the run length appear
harmless.)

## Per-button payloads

33-bit payloads, MSB first. Hex is the 33 bits packed into 9 hex digits: the
final digit holds the 33rd bit plus three padding zeros (so a trailing `1` bit
packs as `8` = `1000`).

| Button | Hex | 33-bit binary (nibble grouped) |
| :--- | :--- | :--- |
| Power | `a979aedb8` | `1010 1001 0111 1001 1010 1110 1101 1011 1` |
| Night Light | `a979becb8` | `1010 1001 0111 1001 1011 1110 1100 1011 1` |
| Max Brightness | `a979b6c38` | `1010 1001 0111 1001 1011 0110 1100 0011 1` |
| Brightness | `a979a4d18` | `1010 1001 0111 1001 1010 0100 1101 0001 1` |
| Color Temperature | `a979a5d08` | `1010 1001 0111 1001 1010 0101 1101 0000 1` |
| Timing † | `a979b2c78` | `1010 1001 0111 1001 1011 0010 1100 0111 1` |

† The **Timing** payload is captured and transmitted correctly, but its effect on
the lamp is unconfirmed — no observable response was seen from either the original
remote or this bridge. The button label comes from the remote's own marking. The
code is included for completeness.

All six share the same first 16 bits (`a979...`), differing only in the command
portion — typical of a fixed-code (non-rolling) remote, which is why simple
replay works.

## Reference `rtl_433` decoder

To decode captures (of either the real remote or your ESP32 output) with an
explicit PWM decoder rather than the auto-guesser:

```
rtl_433 -f 433.92M -X 'n=lamp,m=OOK_PWM,s=260,l=760,r=6752,g=0,t=0,y=0'
```

A **correctly transmitted** (inverted-polarity) signal — including the real
remote — decodes to the table values above directly (Power = `a979aedb8`).

**If your capture instead reads the bitwise complement** (Power = `56865a2f0`),
your transmit polarity is backwards. That complement was exactly the symptom of
the non-inverted transmission during development: the SDR decoded it cleanly, the
bits looked plausible, and the lamp ignored it. A persistent complement between
your output and the expected payload is the single most reliable tell that
HIGH/LOW needs swapping.
