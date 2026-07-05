![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# The Whisper Channel CTF - Writeup

---

## Answers

**Q1. How many distinct signals are visible in the spectrum?**

`2`

> Build a minimal GNU Radio flowgraph:
> `File Source` -> `Throttle` -> `QT GUI Frequency Sink` (samp_rate=48000).
>
> The obvious signal dominates the right side of the spectrum with two bright
> FSK tones. Look carefully at the left side - there are two much fainter
> peaks sitting about 12 dB below the noise floor of the main signal. They
> are easy to miss on a first pass because the y-axis is dominated by the
> strong signal. Try right-clicking the Frequency Sink and lowering the
> reference level, or zooming the y-axis range, to bring the weaker signal
> into view.
>
> Distractor: a full-band noise floor is always present. What distinguishes
> the hidden signal from noise is that it has two discrete tones at fixed
> frequencies rather than a flat or random distribution - the same signature
> that identifies the main signal as FSK.

---

**Q2. What is the approximate carrier frequency offset of the stronger (main)
signal, in Hz?**

`+5000 Hz`

> The main signal's two FSK tones sit at approximately +3000 Hz and +7000 Hz.
> The carrier is the midpoint:
>
> ```
> (+3000 + +7000) / 2 = +5000 Hz
> ```
>
> Distractor: +3000 Hz or +7000 Hz are the individual tone positions, not the
> carrier center. The carrier is always at the midpoint of the two tones.

---

**Q3. What is the approximate carrier frequency offset of the hidden signal,
in Hz?**

`-8500 Hz`

> The hidden signal's two FSK tones sit at approximately -9500 Hz and -7500 Hz.
> The carrier is the midpoint:
>
> ```
> (-9500 + -7500) / 2 = -8500 Hz
> ```
>
> This is on the opposite side of baseband from the main signal - a deliberate
> choice to maximize spectral separation. Both signals fit cleanly within the
> 48 kHz recording bandwidth with no overlap.

---

**Q4. By measuring the two FSK tone positions of the hidden signal, what is
its frequency deviation (fdev) in Hz?**

`1000 Hz`

> FSK deviation is the distance from the carrier center to one tone:
>
> ```
> fdev = (tone_high - tone_low) / 2
>      = (-7500 - -9500) / 2
>      = 2000 / 2
>      = 1000 Hz
> ```
>
> This is half the spacing between the two tone peaks. Alternatively, fdev
> is also the distance from the carrier center (-8500 Hz) to either tone:
>
> ```
> -7500 - (-8500) = +1000 Hz  (mark tone, +fdev)
> -9500 - (-8500) = -1000 Hz  (space tone, -fdev)
> ```
>
> Distractor: the full tone spacing is 2000 Hz. Confusing full spacing with
> fdev gives you 2000 instead of 1000, which would set the wrong `Quadrature
> Demod` gain and produce a compressed float output.

---

**Q5. What is the symbol rate of the hidden signal in bits per second?
Show your working.**

`600 bps`

> Set up a demodulation flowgraph for the hidden signal:
> mix the carrier down to baseband, low-pass filter, FM demodulate, and
> feed into a `Clock Recovery MM` block. The `omega` value the loop settles
> on tells you the samples-per-symbol (sps). Then:
>
> ```
> sym_rate = samp_rate / sps = 48000 / 80 = 600 bps
> ```
>
> You can also estimate it visually from the `QT GUI Time Sink` after
> demodulation - each symbol at 600 bps lasts `1/600 = 1.667 ms`, which
> is `1.667e-3 * 48000 = 80 samples`. Compare this to the main signal's
> `1/1200 = 0.833 ms` per symbol (40 samples) - the hidden signal's
> symbols are visibly twice as wide on the time axis.
>
> Distractor: 1200 bps is the main signal's symbol rate. The hidden signal
> is deliberately slower, making it look smoother on the time sink and
> causing a student who reuses the main signal's `sps=40` to get bit errors
> throughout.

---

**Q6. Approximately how many dB weaker is the hidden signal compared to the
main signal?**

`~13 dB`

> Read the peak power of the main signal's dominant tone and the hidden
> signal's dominant tone from the Frequency Sink's y-axis and subtract.
> The exact value depends on FFT window size and averaging, but should
> consistently land around 12-14 dB.
>
> At -13 dB the hidden signal's amplitude is about 0.22x the main signal's
> amplitude in linear terms - clearly not noise (noise would be flat and
> uncorrelated) but easily overlooked against a dominant signal if you are
> not specifically looking for secondary peaks.

---

**Q7. Decode the hidden signal. What is the flag contained in its payload?**

`FLAG{wh1sp3r_fr3qu3ncy_f0und}`

> Full demodulation steps for the hidden signal:
>
> **Step 1 - Mix carrier to baseband:**
> ```python
> t = np.arange(len(data)) / 48000
> mixed = data * np.exp(-1j * 2 * np.pi * (-8500) * t)
> ```
>
> **Step 2 - Low-pass filter** (cutoff = fdev + sym_rate = 1000 + 600 = 1600 Hz):
> ```python
> from numpy.fft import fft, ifft, fftfreq
> N = len(mixed)
> freqs = fftfreq(N, 1/48000)
> spec = fft(mixed)
> spec[np.abs(freqs) > 1600] = 0
> filtered = ifft(spec).astype(np.complex64)
> ```
>
> **Step 3 - FM demodulate:**
> ```python
> diff  = np.diff(np.unwrap(np.angle(filtered)))
> demod = diff / (2 * np.pi * 1000 / 48000)
> ```
>
> **Step 4 - Clock recovery** (sps=80):
> ```python
> # sample at the center of each symbol period
> bits = (demod[40::80] > 0).astype(np.uint8)
> ```
>
> **Step 5 - Find SYNC word and parse frame** (same as CTF-RF):
> ```python
> SYNC_BIN = '00011010110011111111110000011101'
> bitstr   = ''.join(map(str, bits))
> idx      = bitstr.find(SYNC_BIN)
> # repack bits -> bytes -> parse JSON payload
> ```

---

**Q8. The hidden signal's JSON payload contains a field identifying the channel
type. What is its value?**

`COVERT`

> Read the `chan` field from the decoded JSON payload. The full payload is:
>
> ```json
> {"sat":"ODYSSEY-2","chan":"COVERT","flag":"FLAG{wh1sp3r_fr3qu3ncy_f0und}",
>  "sym_rate":600,"fdev":1000}
> ```
>
> Note that the payload also helpfully includes `sym_rate` and `fdev` fields -
> a self-describing covert channel, consistent with an attacker who wants
> their receiver to auto-configure rather than hardcode parameters.

---

**Q9. What does the main signal's payload contain?**

Normal telemetry for `ODYSSEY-2` in `NOMINAL` mode: `batt_v=7.91`, `temp_c=22.3`.

> Decode the main signal using the same pipeline but with its own parameters:
> carrier offset +5000 Hz, fdev=2000 Hz, sym_rate=1200 bps (sps=40). The
> payload is:
>
> ```json
> {"sat":"ODYSSEY-2","epoch":1713434700,"mode":"NOMINAL","batt_v":7.91,"temp_c":22.3}
> ```
>
> This is the decoy - it looks exactly like the ODYSSEY-2 telemetry from
> CTF-RF, which a student who stops at the obvious signal will decode, find
> nothing unusual, and mark as done. The challenge is recognizing there is
> a second signal at all.

---

**Q10. What is the sample rate of the recording?**

`48000` (48 kS/s)

> Read `samplerate` from `whisper_channel.json`. Consistent with the rest
> of the CTF series.

---

## What Did I Just Do?

### The concept: covert RF channel

A **covert channel** uses a communication medium in a way that was not
intended or is not easily observed. In RF, the simplest covert channel is
a second transmitter operating at a frequency near a legitimate signal but
at a much lower power level - low enough that casual spectrum monitoring
doesn't flag it, but high enough that a receiver configured to look for it
can decode it.

This is not a theoretical attack. Real examples include:

- **Subcarrier injection** - hiding a second modulated signal as a
  subcarrier within the sidebands of a licensed broadcast. Used in analog
  TV to carry closed captions (a legitimate use) but also in covert
  communication research.
- **Spread spectrum overlay** - spreading a signal's energy across a wide
  bandwidth so it sits below the noise floor of narrowband receivers. GPS
  uses this legitimately; it also makes the signal invisible to spectrum
  analyzers not configured for the right despreading code.
- **SDR-based covert channels in satellite links** - demonstrated in
  academic research where a low-power secondary signal is injected into
  a satellite's transponder bandwidth without the primary operator
  noticing, since typical satellite ground equipment monitors aggregate
  power rather than fine-grained spectral content.

### What made the hidden signal hard to spot

Three deliberate design choices made it non-obvious:

**1. Amplitude.** At ~13 dB below the main signal, the hidden signal's
peaks are visible in a high-resolution FFT but are not the first thing
your eye goes to. Most spectrum analyzers and SDR software default to a
display range that makes the dominant peaks fill the screen - the weaker
peaks end up near the bottom of the visible range, easy to dismiss as
noise.

**2. Carrier placement.** The hidden carrier is at -8500 Hz, on the
opposite side of baseband from the main signal at +5000 Hz. A student
who sees the main signal, notes its two tones at +3000/+7000 Hz, and
immediately goes to demodulate it never looks at the negative frequency
half of the spectrum.

**3. Symbol rate.** At 600 bps (half the main signal's rate), the hidden
signal's symbols are twice as wide in time. This means it transmits its
preamble and frame more slowly, so the signal's active window in the time
domain overlaps more with the silence regions. On a time sink it can
look like a very slow, noisy signal rather than a coherent digital
waveform.

### The demodulation pipeline - what changes for the hidden signal

The pipeline is identical to CTF-RF, with three parameter substitutions:

| Step | CTF-RF (main) | Whisper (hidden) |
|---|---|---|
| Carrier mix-down | -5000 Hz | +8500 Hz (negate to mix -8500 to 0) |
| LPF cutoff | 3200 Hz | 1600 Hz |
| Quadrature Demod gain | samp_rate/(2pi*2000) | samp_rate/(2pi*1000) |
| Clock Recovery omega | 40 | 80 |

Everything else - the SYNC word, frame format, CRC algorithm, JSON payload
structure - is identical. The covert channel reuses the same protocol as
the legitimate link, which means a receiver that already knows how to
decode the main signal only needs to re-tune and adjust three numbers to
decode the hidden one.

### Why this matters operationally

Spectrum monitoring for satellite links typically operates at the level of
"is the transponder's total occupied power within its licensed band?" rather
than "does every spectral component within the band belong to an authorized
transmitter?" A low-power secondary signal injected into the same baseband
recording as a legitimate downlink can pass aggregate power checks without
triggering any alert, as long as the total power budget stays within limits.

Detection requires either:
- **High-resolution spectral analysis** with enough FFT bins to resolve the
  individual tones of the hidden signal against the main signal's sidebands.
- **Anomaly detection** on expected vs. actual spectrum shape - the hidden
  signal creates asymmetry between the positive and negative frequency halves
  of the baseband recording that a symmetric main signal alone would not
  produce.
- **Known-signal subtraction** - if you know exactly what the main signal
  looks like, you can model it, subtract it from the recording, and analyze
  the residual. What remains after subtraction is everything that isn't the
  expected signal, including covert channels.

  
***                                                                 

<b><i>Looking for a different CTF/Lab? </br>[Lab Directory](/navigation.md)</i></b>
