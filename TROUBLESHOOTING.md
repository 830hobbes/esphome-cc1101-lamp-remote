# Troubleshooting

Symptom → likely cause → fix. Ordered roughly by how much time each one tends to
waste.

## Config compiles and runs, but the SDR sees **no RF at all**

- **`gdo0_pin` is set inside the `cc1101:` block.** On TX this severs the RMT→pad
  routing (ESPHome #16876). **Fix:** remove `gdo0_pin` from `cc1101:`; let
  `remote_transmitter` own the data pin.
- **YAML `data_pin` doesn't match the physical wire.** The boot log and TX log
  look perfect because none of that depends on where the wire actually is.
  **Fix:** confirm the CC1101 `GDO0` wire lands on the GPIO named by `data_pin`.
- **Chip not detected / SPI dead.** Boot log shows `FFFF`/`0000`/`FF0F` instead of
  a real chip ID. **Fix:** check CS/MOSI/MISO/SCK wiring and 3.3 V power.

## SDR sees strong RF, decodes the correct bits, but the **device ignores it**

This is the big one. The bitstream is right but the *waveform* isn't.

- **Polarity is inverted.** Most likely cause. The decoder hides polarity, so a
  correct decode proves nothing here. **Fix:** swap HIGH/LOW (`240,-760` ↔
  `760,-240`). A persistent bitwise complement vs the expected payload is the
  tell.
- **Bit period isn't constant.** If you used a fixed short gap after every mark,
  `1` and `0` bits have different total periods. **Fix:** make every bit sum to
  the same period (here ~1000 µs); short-mark/long-gap for one bit value,
  long-mark/short-gap for the other.
- **Missing / wrong preamble.** Some receivers need the carrier warm-up burst to
  arm. **Fix:** prepend the preamble (here: ten short marks — nine `240, -760`
  pairs plus a `240, -6500` whose gap doubles as the sync).
- **Repeat structure wrong.** Sending the preamble before *every* repeat, or
  wrapping an already-repeating array in another `repeat:`, changes what the
  receiver sees. **Fix:** preamble once, then payload×N with sync gaps, as one
  flat array; no extra `repeat:` wrapper.

## `rtl_433` reports a **different modulation name** for your TX vs the remote

- **Cosmetic.** The `-A` auto-guesser is a heuristic near a decision boundary
  (e.g. "Manchester" vs "PWM"). If the measured pulse/gap widths match, the label
  difference is not a real signal difference. Use an explicit `-X` decoder and
  compare the actual bits.

## Your TX bit count **doesn't match the real remote's**

- **Structural mismatch — worth fixing.** Decode both with the same explicit
  `-X` decoder and compare the repeating unit: same preamble length, same payload
  length, same repeats per press. If your frame is much longer than the remote's,
  the usual cause is an already-repeating raw array *also* wrapped in a `repeat:`
  block (e.g. a 174-bit frame going out as ~870 bits). **Fix:** put the repeats
  and sync gaps in the array *or* in a `repeat:` block — never both.
- Confirming your transmitted bit count matches the remote's is a quick, objective
  checkpoint. Do it every time you change the frame.

## Payload length / repeat count **changes between captures**

- **Capture-window artifact.** How many repeats land in one detected package
  depends on timing; multiple presses can also merge. Don't treat this as a
  protocol property. Compare the *repeating unit*, not the total length. (This is
  the flip side of the check above: a bit-count *difference* matters, but a
  bit-count *wobble* across captures of the same button usually doesn't.)

## Range is marginal

- **Antenna, not power.** A bare CC1101 module radiates weakly. A ~17.3 cm
  quarter-wave wire helps more than raising `output_power`. `output_power: 11` is
  the max but usually a small effect compared to a real antenna.

## Device works with the real remote from across the room, ESP32 only up close

- Likely antenna/matching quality on a generic CC1101 breakout vs the remote's
  tuned PCB antenna. Improve the antenna, or consider a learn-and-replay bridge
  (e.g. Broadlink RM4 Pro) that reproduces the original analog waveform instead
  of regenerating it from a spec.
