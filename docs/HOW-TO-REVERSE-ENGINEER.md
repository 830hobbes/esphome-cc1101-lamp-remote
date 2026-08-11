# How to reverse-engineer a 433 MHz OOK remote (the method)

This is the workflow used to derive the protocol in this repo. It generalizes to
most cheap fixed-code 433 MHz OOK remotes.

## You need

- An **RTL-SDR** (any RTL2832U dongle; an R820T tuner is common).
- **`rtl_433`** (native package, or the `hertzg/rtl_433` Docker image).
- The **physical remote** you're cloning.
- Optionally: the device's **FCC filing** (search the FCC ID printed on it at
  `fcc.report` or `fccid.io`). The test report often states pulse widths, bit
  period, and repeat counts — a great independent cross-check.

## Step 1 — Capture the real remote

Run `rtl_433` in analyzer mode and press each button several times:

```
rtl_433 -f 433.92M -A -Y minsnr=15
```

Note the reported pulse-width buckets, gap buckets, and especially the
**"Pulse+gap period distribution"** — see Step 4.

## Step 2 — Pin down the encoding

From the analyzer output, identify:

- **Short vs long mark widths** (here ~260 µs and ~760 µs).
- **Gap widths** — are they constant, or do they vary with the mark? (Here they
  vary, because the *total period* is constant — PWM.)
- **Preamble / sync** — a long uniform run of marks and/or a long gap.
- **Repeat count** — how many times the payload repeats per press.

Then force an explicit decoder to read stable bits:

```
rtl_433 -f 433.92M -X 'n=lamp,m=OOK_PWM,s=260,l=760,r=6752,g=0,t=0,y=0'
```

Capture each button; record the `codes:` payload for each.

## Step 3 — Watch out for these traps

These each cost real debugging time:

1. **Polarity is invisible to the decoder.** `rtl_433` normalizes it away, so a
   clean decode does NOT confirm on-air polarity. If your transmitter reproduces
   the bits perfectly but the device ignores it, **invert HIGH/LOW and retry.**
   A persistent bitwise *complement* between your output and the "expected"
   payload is a strong tell that polarity is flipped.

2. **The auto-classifier's label is not the truth.** `-A` may call your signal
   "Manchester" and the real remote "PWM" (or vice-versa) even when the measured
   pulse widths are identical — it's a heuristic sitting near a decision
   boundary. Trust the *measurements*, not the guessed label.

3. **Capture windows merge/split repeats.** The same press can show up as 4, 5,
   or more payload copies, and multiple presses can merge into one long capture,
   depending on timing. Don't infer protocol structure from the repeat count of a
   single capture.

4. **Byte-alignment padding.** `rtl_433`'s packed `data:` hex pads the final byte.
   When decoding by hand, slice the *first* N real bits, not the last N.

## Step 4 — The bit-period check that actually matters

Compare each bit's **total period (mark + following gap)**, not just mark width:

- If every bit period is **constant** (e.g. ~1000 µs) and only the mark/gap
  *split* changes → it's true PWM. Reproduce that: give a `1` a short-mark +
  long-gap and a `0` a long-mark + short-gap (or vice-versa) so periods stay
  constant.
- A naive "fixed short gap after every mark" scheme makes bit periods vary ~2×.
  It still decodes on an SDR but often won't drive a real receiver, whose
  demodulator times off the total period.

Look specifically at the analyzer's **"Pulse+gap period distribution"** line:
one tight cluster = constant period (good); two clusters = your periods vary.

## Step 5 — Build the transmitter and verify the loop

Reproduce the frame with your transmitter (see the ESPHome config in this repo),
then capture *your* output with the **same** `rtl_433 -X` decoder and compare,
bit for bit, against the real remote. Iterate until they match on: payload,
polarity, preamble, repeat structure, and bit period.

### Match the bit count / frame structure explicitly

One of the most useful cross-checks: **your transmitter and the real remote
should decode to the same repeating structure** — same preamble length, same
payload length, and the same number of payload repeats per press. A bit-count
mismatch is a fast, objective signal that something is structurally wrong.

The classic failure here: an already-repeating raw array *also* wrapped in a
`repeat:` block, so every press transmits N copies of (preamble + N payloads) —
e.g. a 174-bit frame going out as an ~870-bit blob. The SDR bit count makes this
obvious the moment you look for it.

**Caveat — don't chase the raw `len:` count blindly.** `rtl_433`'s capture window
merges and splits repeats depending on timing, so the same button can legitimately
report 4 vs 5 payload copies (or two presses can merge into one package). Compare
the *repeating unit* — preamble + one payload — and the repeat count of a single
clean press, not the raw total length of whatever landed in one capture. The point
is "does the decoded structure match," and bit count is how you catch when it
doesn't.

Then — the only test that ultimately counts — **press it at the actual device.**
The SDR can confirm the bitstream; only the device confirms the analog envelope
is good enough.

## Meta-lesson

Hold onto small unexplained observations instead of reasoning them away. In this
project, a consistent, "cosmetic-looking" difference in how the SDR classified
the two transmitters was the polarity problem leaking through the whole time.
The clean-looking bit match kept making it *feel* solved when it wasn't.
Calibrated doubt beats confident tunnel vision.
