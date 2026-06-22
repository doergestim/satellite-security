![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Orbital Forensics CTF - Writeup

---

## Answers

**Q1. Comparing `ODYSSEY-2.tle` and `ODYSSEY-2-forged.tle` line by line, which two
orbital elements differ between them?**

**Mean motion** and **Right Ascension of the Ascending Node (RAAN)**.

> Open both files side by side and diff line 2 character by character:
>
> ```bash
> diff <(cat ODYSSEY-2.tle) <(cat ODYSSEY-2-forged.tle)
> ```
>
> Line 1 (epoch, drag terms) is identical in both files. Line 2 differs in two fields:
> the RAAN column (`245.1234` vs `247.1288`) and the mean motion column at the end of
> the line (`15.21345678` vs `15.21390678`). Inclination, eccentricity, argument of
> perigee, and mean anomaly are untouched - a forger only needs to nudge two numbers to
> shift where the groundstation thinks the satellite will be.

---

**Q2. What is the difference in mean motion (in revolutions/day) between the real and
forged TLE? Give your answer to 5 decimal places.**

`0.00045` rev/day

> Mean motion is the last numeric field before the revolution number on line 2 (columns
> 53-63). Subtract directly:
>
> ```
> 15.21390678 - 15.21345678 = 0.00045
> ```
>
> This is a tiny absolute change - well within what a careless glance at the file would
> miss - but mean motion compounds over time, so even a change this small visibly shifts
> where the satellite is predicted to be after enough orbits.

---

**Q3. According to `pointing_error.csv`, what is the angular pointing error at the very
start of the analysis window (the first row)?**

`3.3119 degrees`

> Open `pointing_error.csv` and read the `angular_error_deg` column of the first data
> row (right after the header). The forged TLE is already measurably wrong before the
> pass even properly begins - this is the baseline drift purely from the RAAN and mean
> motion offset accumulated since the TLE epoch.

---

**Q4. At what point in the analysis window does the angular pointing error first exceed
5 degrees? Give your answer as minutes:seconds from the start of the window.**

`3:45` (3 minutes 45 seconds into the window)

> Scan down the `angular_error_deg` column until you find the first value greater than
> `5.0000`. The corresponding `time` column entry is `2024-04-19T00:43:45`. The window
> started at `2024-04-19T00:40:00` (from `gs_config.json`), so the elapsed time is
> `00:43:45 - 00:40:00 = 3:45`.
>
> A 5-degree pointing error is significant for a real dish - most ground station
> antennas at UHF/S-band have a beamwidth narrow enough that a few degrees of mis-point
> meaningfully drops the received signal strength, and at higher frequencies (X-band,
> Ka-band) it can mean losing the satellite from the beam entirely.

---

**Q5. What is the maximum angular pointing error recorded anywhere in
`pointing_error.csv`?**

`14.4583 degrees`

> Either scan the column manually or use a quick command:
>
> ```bash
> python3 -c "
> import csv
> rows = list(csv.DictReader(open('pointing_error.csv')))
> m = max(rows, key=lambda r: float(r['angular_error_deg']))
> print(m)
> "
> ```
>
> The peak occurs partway through the pass (`2024-04-19T00:48:45`), not at the start or
> end. This is typical of TLE forgery effects - the error does not grow smoothly or
> monotonically because azimuth and elevation respond differently to the RAAN and mean
> motion changes as the geometry between the satellite and the groundstation evolves
> through the pass.

---

**Q6. In the TLE format, which line contains the Right Ascension of the Ascending Node
(RAAN), and roughly which columns is it found in?**

**Line 2**, approximately **columns 18-25**.

> The TLE format is strictly column-based (fixed-width, not delimited), defined by the
> original NORAD/NASA two-line element specification. Line 2's layout is:
>
> ```
> 2 NNNNN III.IIII RRR.RRRR EEEEEEE AAA.AAAA MMM.MMMM NN.NNNNNNNNRRRRRC
>   ^satnum ^incl    ^RAAN    ^ecc    ^argp    ^mean    ^mean     ^rev  ^checksum
>                                              anomaly  motion    num
> ```
>
> Counting from column 1: satellite number (3-7), inclination (9-16), RAAN (18-25),
> eccentricity with implied leading decimal (27-33), argument of perigee (35-42), mean
> anomaly (44-51), mean motion (53-63), revolution number (64-68), checksum (69).

---

**Q7. `ODYSSEY-2.tle` (the catalog copy) has a corrupted checksum on line 2. What is the
correct checksum digit, and what was the file shipped with instead?**

Correct digit: **`4`**. The file was shipped with **`7`**.

> The TLE checksum algorithm: sum every digit in the line (treating a literal minus sign
> `-` as 1, ignoring everything else - letters, spaces, periods, plus signs), then take
> the result modulo 10. The final character of the line is the only thing excluded from
> its own checksum calculation.
>
> ```python
> def tle_checksum(line):
>     total = 0
>     for c in line[:-1]:   # exclude the existing checksum character
>         if c.isdigit():
>             total += int(c)
>         elif c == '-':
>             total += 1
>     return total % 10
>
> line2 = "2 56789  97.4500 245.1234 0011234  88.7654 271.3456 15.21345678 4521" + "7"
> print(tle_checksum(line2))   # -> 4, but the file's last character is 7
> ```
>
> A checksum mismatch on a TLE you pulled from a supposedly authoritative catalog is a
> red flag independent of anything else in this challenge - it tells you the file was
> either corrupted in transit or edited by something that didn't recompute the checksum
> afterward. In this scenario it's a deliberately planted error for you to catch; in a
> real pipeline a checksum failure should hard-fail the parser rather than silently
> propagate a malformed orbit into the propagator.

---

**Q8. Using the real TLE and an SGP4 propagator, what is the satellite's altitude (in
km) at the start of the analysis window defined in `gs_config.json`?**

`502.109 km`

> `gs_config.json` specifies `start_iso: 2024-04-19T00:40:00`. Propagate
> `ODYSSEY-2.tle` (using the corrected checksum from Q7) to that exact timestamp with
> SGP4, take the resulting ECI position vector, and compute its magnitude minus the
> Earth's mean equatorial radius:
>
> ```python
> from sgp4.api import Satrec, jday
> import math
>
> l1 = "1 56789U 24108A   24109.50000000  .00001234   00000-0  12345-4 0  9990"
> l2 = "2 56789  97.4500 245.1234 0011234  88.7654 271.3456 15.21345678 45214"
> sat = Satrec.twoline2rv(l1, l2)
>
> jd, fr = jday(2024, 4, 19, 0, 40, 0)
> e, r, v = sat.sgp4(jd, fr)
> alt_km = math.sqrt(sum(c**2 for c in r)) - 6378.137
> print(alt_km)   # 502.109
> ```
>
> This altitude is consistent with the mean motion in the TLE: roughly 15.21
> revolutions/day corresponds to an orbital period of about 94.7 minutes, which by
> Kepler's third law implies a semi-major axis of roughly 6880 km - an altitude of
> about 500 km above the reference ellipsoid, matching what SGP4 returns directly.

---

**Q9. Using the satellite's orbital speed at that same moment, estimate how many metres
of position error a 1-second timing error would introduce. Show your working.**

**Approximately 7615 metres** (about 7.6 km).

> SGP4 returns a velocity vector alongside the position vector. Take its magnitude:
>
> ```python
> speed_km_s = math.sqrt(sum(c**2 for c in v))
> print(speed_km_s)   # 7.6146 km/s
> ```
>
> Position error from a timing error is simply `speed x time`:
>
> ```
> 7.6146 km/s * 1 s = 7.6146 km = 7614.6 m
> ```
>
> This is the core reason satellite operations are obsessive about clock
> synchronization. At LEO orbital speeds (roughly 7.5-7.7 km/s for a ~500 km orbit), even
> a clock that's off by a single second translates to a position error larger than most
> antenna beamwidths at the relevant slant range - more than enough to cause a
> groundstation to mis-point or lose lock entirely, with no orbital element tampering
> required at all. A forged TLE and a desynchronized clock produce the same operational
> symptom through completely different root causes.

---

**Q10. What groundstation coordinates (latitude/longitude) were used to compute the
pointing angles in this analysis? Where is this located?**

`44.3175 N, 23.7975 E` - **Craiova, Romania**.

> Read `lat_deg` and `lon_deg` directly from `gs_config.json`. Every az/el value in
> `pointing_error.csv` is meaningless without knowing where the observer is standing -
> the same satellite orbit produces a completely different antenna pointing plan
> depending on the groundstation's location, which is why this file is shipped alongside
> the pointing data rather than assumed.

---

## What Did I Just Do?

### What a TLE actually is

A **Two-Line Element set (TLE)** is a compact, fixed-width text encoding of a satellite's
orbit at a specific moment in time (the epoch). It is not a continuously updated live
feed - it is a snapshot, plus a mathematical model (SGP4/SDP4) that lets you propagate
that snapshot forward or backward in time to predict where the satellite will be at any
other moment, within the model's accuracy limits.

Six numbers fully describe a Keplerian orbit's shape and orientation in space:

- **Inclination** - tilt of the orbital plane relative to the equator
- **RAAN** - where the orbital plane crosses the equator, going north (the orbital
  plane's "compass heading" in inertial space)
- **Eccentricity** - how elliptical the orbit is (0 = perfectly circular)
- **Argument of perigee** - orientation of the ellipse within the orbital plane
- **Mean anomaly** - where the satellite is along its orbit at the epoch
- **Mean motion** - how fast the satellite completes orbits (revolutions per day)

A TLE packs these six elements, plus the epoch and some drag/perturbation terms, into
exactly two 69-character lines. Every field lives in a fixed column range - there are no
delimiters - which is both the format's biggest strength (trivial to parse with fixed
offsets, unambiguous) and its biggest weakness (a single misplaced digit silently shifts
into the wrong field with no syntax error to catch it).

### Why mean motion and RAAN are the attacker's tool of choice

Out of the six orbital elements, tampering with **mean motion** and **RAAN** produces
the largest pointing error for the smallest, least-noticeable numeric change:

- **Mean motion** controls how fast the satellite moves along its orbit. A tiny error
  here doesn't matter much one orbit later, but it compounds: after N orbits, the
  position error grows roughly linearly with N. A few hours after the TLE epoch, even a
  mean motion error in the fifth decimal place produces a satellite that arrives at any
  given point in its orbit several seconds early or late - which translates directly to
  a wrong az/el prediction from the ground.
- **RAAN** rotates the entire orbital plane around the Earth's axis. A RAAN error
  doesn't grow over time the way a mean motion error does, but it produces an immediate,
  constant angular offset in where the orbital plane appears to be - which shows up as a
  near-fixed bias in azimuth from any single groundstation.

Combined, the two together is exactly what you saw in this challenge: an error that is
already present at the start of the window (the RAAN contribution) and that grows as the
pass progresses (the mean motion contribution).

### Why this matters operationally

A groundstation antenna - especially at higher frequencies like X-band or Ka-band used
for high-rate downlink - typically has a beamwidth measured in single-digit degrees or
less. The 14.46-degree peak error you found in this challenge is large enough to put the
satellite completely outside the main beam for a meaningful chunk of the pass. The
operational consequences range from degraded link margin (weaker signal, more dropped
packets) to a complete loss of signal if the antenna is following a fixed open-loop
pointing plan rather than actively tracking the actual signal (autotrack/conscan).

This is why Lab 5 (TheDrift) frames TLE tampering as an attack on the **distribution
channel**, not on cryptography or the satellite itself. The orbital mechanics are public
knowledge and the math (SGP4) is a published, open standard - there is nothing to break.
The entire attack surface is "can the attacker get a slightly wrong text file accepted as
if it were the real one." This is precisely why operational TLE distribution increasingly
relies on authenticated, signed catalog sources (such as Space-Track's authenticated API)
rather than plain-text file drops or unauthenticated mirrors - the defense is not making
the orbit harder to compute, it's making the file's origin harder to fake.

### Why checksums alone aren't a security control

The checksum you recomputed in Q7 catches **accidental corruption** - a dropped byte in
a file transfer, a copy-paste error, a truncated download. It does **not** catch a
deliberate forgery, because the forger simply recomputes the correct checksum for their
tampered values before distributing the file, exactly as this CTF's own
`ODYSSEY-2-forged.tle` does. A valid checksum tells you the file is internally
self-consistent; it tells you nothing about whether the orbital elements inside it are
true. That distinction - integrity versus authenticity - is the same one that came up in
the Replay Detection CTF, just applied to a different kind of file.



***                                                                 

<b><i>Looking for a different CTF/Lab? </br>[Lab Directory](/navigation.md)</i></b>
