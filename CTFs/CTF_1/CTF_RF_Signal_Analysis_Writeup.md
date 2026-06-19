# RF Signal Analysis CTF - Writeup

---

## Answers

**Q1. What is the sample rate of the `pass_ctf.iq` recording?**

`48000` (48 kS/s)

> Open `pass_ctf.json`. Read the `samplerate` field directly.
> This value must be set on the `samp_rate` variable block in GNU Radio - every
> timing calculation downstream depends on it being correct.

---

**Q2. What sync word marks the start of each frame in the bitstream?**

`0x1ACFFC1D`

> Run the demodulation flowgraph on `pass_ctf.iq` to produce framed bytes, then:
>
> ```bash
> xxd pass_ctf_BPF.txt | head -5
> ```
>
> The first four bytes of each frame are `1a cf fc 1d`.
>
> Distractor: `0xAAAAAA` is a common preamble pattern used before the sync word
> to help clock recovery lock in. It is not the frame delimiter - it appears in
> the bitstream before the sync word and gets stripped by the `Correlate Access
> Code - Tag` block.

---

**Q3. What modulation scheme was used to transmit this signal?**

`2-FSK` (Binary FSK / BFSK)

> Build a minimal GNU Radio flowgraph: `File Source` -> `Throttle` ->
> `QT GUI Frequency Sink` with `samp_rate = 48000`. Run it.
>
> You will see two distinct energy concentrations symmetric around the center
> frequency. Two discrete tones that switch between them = binary
> frequency-shift keying.
>
> Distractor: BPSK produces a single carrier with phase transitions. QPSK shows
> four phase states. What you see here is unambiguously two separate tones.

---

**Q4. What is the symbol rate of this downlink in bits per second?**

`2400 bps`

> This value is the `sym_rate` variable in the GNU Radio flowgraph. It controls
> the `Clock Recovery MM` block's omega parameter:
>
> ```
> omega = samp_rate / sym_rate = 48000 / 2400 = 20
> ```
>
> You can measure it experimentally from the `QT GUI Time Sink` by counting
> how many samples fit in one symbol period. At 48 kS/s with 20 samples per
> symbol, each symbol lasts ~416 us.
>
> Distractor: 1200 bps is the symbol rate used in Lab 1 (TheIntercepter) for
> ODYSSEY-1. This is a different satellite and a different recording - do not
> reuse Lab 1 parameters.

---

**Q5. What is the FSK frequency deviation in Hz?**

`4500 Hz`

> Set `fdev` in the `Quadrature Demod` block and observe the output on a
> `QT GUI Time Sink`. With the correct deviation value the demodulated float
> signal swings cleanly between +1 and -1.
>
> You can also measure it directly from the `QT GUI Frequency Sink`: the two
> tone peaks sit at approximately -4500 Hz and +4500 Hz relative to center.
>
> Distractor: 2000 Hz is the deviation used in Lab 1. Using that value here
> causes the `Quadrature Demod` gain to be wrong by a factor of 2.25, producing
> a compressed float output. The `Binary Slicer` can still threshold it, but
> intersymbol noise margin is reduced and you may see bit errors on longer
> frames.

---

**Q6. How many IQ samples represent exactly one transmitted symbol?**

`20 samples per symbol`

Working:

```
sps = samp_rate / sym_rate = 48000 / 2400 = 20.0
```

> This is the `omega` value passed to `Clock Recovery MM`. It tells the block
> how far apart (in samples) to expect consecutive bit boundaries.
>
> Distractor: some students compute `48000 / 1200 = 40` by reusing the Lab 1
> symbol rate. The Clock Recovery block will appear to lock but decodes at
> half speed, producing garbage where every other bit is merged.

---

**Q7. What is the satellite name embedded in the telemetry payload?**

`ODYSSEY-2`

> Run the full demodulation chain and parse the framed output:
>
> ```bash
> xxd pass_ctf_BPF.txt
> ```
>
> The first frame (TYPE=0x01, SEQ=7) contains a JSON payload. Read the `sat`
> field exactly, including capitalisation and the hyphen.
>
> Distractor: `ODYSSEY-1` is the satellite from Labs 1-3. This is a different
> recording from a different satellite. Read the value from the actual bytes.

---

**Q8. What Unix epoch timestamp is embedded in the telemetry payload?**

`1713434700`

> Same source as Q7 - read the `epoch` field from the telemetry JSON payload
> in the first decoded frame.
>
> Cross-check:
>
> ```bash
> python3 -c "import datetime; print(datetime.datetime.utcfromtimestamp(1713434700))"
> # 2024-04-18 10:05:00 UTC
> ```
>
> Distractor: `1713371337` is the epoch from Lab 1 `pass_01.iq`. `1713372600`
> is the epoch from Lab 3 `takeover_pass.iq`. Both are wrong here - the epoch
> must come from this specific recording.

---

**Q9. Using the satellite name and epoch you extracted, compute the uplink
authentication token. What is it?**

`f1fb9cc3`

> The auth scheme is revealed in the `hint` field of the telemetry payload:
>
> ```
> sha1(str(epoch)+sat+"-BLUE")[:8]
> ```
>
> Working in Python:
>
> ```python
> import hashlib
> epoch = 1713434700
> sat   = "ODYSSEY-2"
> auth  = hashlib.sha1(f"{epoch}{sat}-BLUE".encode()).hexdigest()[:8]
> print(auth)  # f1fb9cc3
> ```
>
> The input string concatenates: decimal epoch, satellite name, literal `-BLUE`.
> No spaces, no separators between components.
>
> Distractor: using `sha256` gives a different 8-character prefix. Using the
> Lab 1 epoch `1713371337` with `ODYSSEY-2` gives a different result. Using
> `ODYSSEY-1` with the correct epoch also gives a different result. All four
> inputs must be correct: algorithm, epoch, satellite name, and salt.

---

**Q10. What integrity algorithm protects each frame, and which bytes does it cover?**

`CRC16-CCITT` (`binascii.crc_hqx`, init `0xFFFF`) - covers
`[VER][SEQ][TYPE][LEN][PAYLOAD]`. The 4-byte SYNC word is excluded.

> The frame layout is:
>
> ```
> [SYNC 4B][VER 1B][SEQ 2B][TYPE 1B][LEN 2B][PAYLOAD nB][CRC16 2B]
>  ^^^^^^                                                  ^^^^^^^^^
>  excluded          CRC covers this range                 result
> ```
>
> You can verify by computing it yourself against the decoded frame bytes:
>
> ```python
> import binascii
> def crc16_ccitt(b, init=0xFFFF):
>     return binascii.crc_hqx(b, init)
>
> # frame_bytes starts at VER (byte after the 4-byte SYNC)
> crc_input = frame_bytes[0 : 6 + payload_len]   # [VER..PAYLOAD]
> crc_calc  = crc16_ccitt(crc_input)
> ```
>
> Distractor: CRC32 is used by Ethernet and many embedded protocols but is not
> used here. CRC16 with init `0x0000` is a different variant and produces
> different output on the same data. The specific variant here is CRC16-CCITT
> with init `0xFFFF`, implemented by Python's `binascii.crc_hqx`.

---

## What Did I Just Do?

### The signal chain - from RF to bytes

A satellite does not transmit JSON over the air directly. The data passes through
several encoding layers before it becomes radio waves, and decoding it means
reversing those layers in order:

```
Transmitted:
[JSON payload]
    -> framed with SYNC word + header + CRC16
    -> FSK modulated (frequency shifts encode 0s and 1s)
    -> upconverted to UHF carrier, transmitted

Your analysis (reversed):
[IQ file]
    -> Quadrature Demod     (FSK -> float stream)
    -> Low-Pass Filter      (remove high-freq noise)
    -> Clock Recovery MM    (find symbol boundaries)
    -> Binary Slicer        (float -> 0/1 bits)
    -> Correlate Access Code (find SYNC -> frame start)
    -> Repack Bits          (bits -> bytes)
    -> xxd / JSON parse     (bytes -> readable data)
```

### What is an IQ file?

An IQ file stores **complex baseband samples** - pairs of values (I, Q)
representing the real and imaginary components of the signal after it has been
downconverted from its carrier frequency to near-DC. `complex_float32` means
each sample is two 32-bit IEEE 754 floats (I then Q), so 8 bytes per sample.

At 48000 sps and 8 bytes per sample, `pass_ctf.iq` represents about 1.5 seconds
of captured signal. A real satellite pass over a ground station lasts roughly
5-15 minutes at low Earth orbit - this is a short synthetic extract.

### Why FSK?

**Frequency-Shift Keying** is common in small satellite downlinks because:

- It is robust to amplitude variations caused by the satellite tumbling, antenna
  misalignment, or path loss changes through the pass. Only frequency matters,
  not amplitude.
- It is simple to demodulate - a `Quadrature Demod` block (frequency
  discriminator) converts it to a 1-D float in one step.
- At these parameters the occupied bandwidth is roughly `2 * (fdev + sym_rate/2)
  = 2 * (4500 + 1200) = 11400 Hz`, which fits comfortably in a standard UHF
  allocation.

The two tones at +-4500 Hz are the **mark** (binary 1) and **space** (binary 0)
frequencies. `Quadrature Demod` outputs a positive float for the mark tone and
a negative float for the space tone. `Binary Slicer` thresholds that at zero to
produce clean 0x00 and 0x01 bytes.

### Why does sps matter?

At 48 kS/s and 2400 symbols/s, each symbol lasts exactly `48000 / 2400 = 20`
samples. The `Clock Recovery MM` block uses this as its starting estimate
(`omega`) and runs a Mueller-Muller timing error detector to track the exact
sample instant within each symbol period where the signal is most stable. This
is what allows decoding a real signal even when the transmitter's oscillator is
not perfectly matched to the receiver's sample clock - the loop corrects for
small frequency differences continuously.

If you set `omega = 40` (the Lab 1 value) instead of `20`, clock recovery locks
at half the actual symbol rate. The `Binary Slicer` sees two samples per symbol
and treats them as two separate bits. Every byte in the output is wrong because
the bit boundaries are misaligned.

### The SYNC word 0x1ACFFC1D

This is the actual **CCSDS Attached Sync Marker (ASM)** defined in
CCSDS 131.0-B-3. It is not invented for this lab - it is used in real satellite
telemetry systems. Its bit pattern `00011010110011111111110000011101` was chosen
for good autocorrelation properties: it is unlikely to appear by chance in data
and unlikely to be confused with a shifted version of itself.

The `Correlate Access Code - Tag` block in GNU Radio searches the bitstream for
this exact 32-bit pattern and places a stream tag at each hit. Downstream blocks
use that tag as the frame boundary.

### The authentication scheme - and why it fails

The auth tag formula:

```
auth = sha1(str(epoch) + sat + "-BLUE")[:8]
```

Has three problems that make it trivially breakable once you have one downlink
frame:

**1. No secret key.** The only inputs are the epoch (in the telemetry
plaintext), the satellite name (also in the telemetry plaintext), and the salt
`-BLUE` (hardcoded in the software, recoverable from any binary or source
copy). An attacker who intercepts one frame has all three inputs and can compute
the tag with one `hashlib.sha1()` call.

**2. Truncated to 32 bits.** `sha1` produces 160 bits. Keeping only 8 hex
characters (32 bits) means there are 2^32 (~4 billion) possible tags - trivially
brute-forceable even without the formula, and completely deterministic once you
have the formula.

**3. No freshness binding.** The epoch is a fixed property of the pass, not a
challenge issued by the ground station. The same tag is valid forever for any
ground station that accepts the same epoch. There is no session nonce, no
expiry, and no replay protection.

The correct mitigations are demonstrated in Lab 4 (TheRelay): server-issued
nonces that bind each auth session, timestamp freshness windows that reject
frames older than +-300 seconds, and replay caches that drop duplicate frame
hashes even if the timestamp looks fresh.

### Real-world context

The attack path you practiced - intercept downlink, read credentials in
plaintext, derive uplink auth - has been demonstrated against real satellite
systems:

- Several early CubeSat missions used AX.25 with no authentication. Anyone
  with an SDR and the published callsign could inject command frames.
- The **CCSDS Security Layer (SDLS, CCSDS 355.0-B)** exists specifically to
  add authenticated encryption to the CCSDS frame layer, but adoption in small
  satellite programs is slow due to hardware and software complexity.
- The `0x1ACFFC1D` sync word appearing in this capture is the real CCSDS ASM -
  any production satellite using CCSDS framing will have the same marker in its
  downlink, making it a useful signature when analyzing unknown captures.
