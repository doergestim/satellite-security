![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Blue Lab 2 - SatDump Telemetry Replay: Decode, Compare, Detect

**Goal:** Use **SatDump** to decode two IQ recordings (clean vs replay) and **prove replay**

Replay in this lab is demonstrated by showing that two passes:

- Both look valid
- Both demodulate cleanly
- Both would be trusted by an operator

But when you compare their derived artifacts:

- The files are not identical (different hashes)

- Yet they show unnatural similarity:
    - Same structure
    - Same spectral fingerprint
    - Same repetition patterns

A real satellite pass changes naturally
A replayed pass changes just enough to look different, but not enough to be real

---

## What you have (provided)

Under `~/Desktop/SatDumpLab/`:

```
SatDumpLab/
├── pass_clean.iq
├── pass_replay.iq
```

---

## Start

```bash
cd ~/Desktop/SatDumpLab
mkdir -p output_clean output_replay results
```

---

# Part A - Decode CLEAN IQ

- Launch SatDump

```bash
satdump-ui
```

<img width="999" height="635" alt="image" src="https://github.com/user-attachments/assets/fa04cdf8-0004-4adb-9faf-a7bcf8818a52" />


1. Pipepline: `Analog Demodulation`
2. Input File: `/home/ubuntu/Desktop/SatDumpLab/pass_clean.iq`
3. Output Directory: `/home/ubuntu/Desktop/SatDumpLab/output_clean/`
4. Baseband Format: `cs8`
5. Samplerate: `48000`

<img width="1001" height="748" alt="2026-03-19_14-08" src="https://github.com/user-attachments/assets/bcaf88d7-540a-4b50-942e-231852e06247" />


- Start

Verify:

```bash
ls -lah output_clean
```

<img width="877" height="84" alt="image" src="https://github.com/user-attachments/assets/d6fd28c5-56c5-4446-b14c-353ac635fa28" />

---

# Part B - Decode REPLAYED IQ

Repeat **Part A** using:

- IQ file: `pass_replay.iq`
- Output: `output_replay`

Verify:

```bash
ls -lah output_replay
```

<img width="877" height="84" alt="image" src="https://github.com/user-attachments/assets/afc5f05b-0f1a-4945-bb7a-08e55a675be7" />


---

# Part C - Replay Detection

## Hash comparison

```bash
sha256sum output_clean/* | sort > clean.hashes.txt
```

```bash
sha256sum output_replay/* | sort > replay.hashes.txt
```

```bash
diff -u clean.hashes.txt replay.hashes.txt
```

<img width="1061" height="103" alt="image" src="https://github.com/user-attachments/assets/fcc276fa-bf90-41ad-9fd4-99e2bd5c5033" />

- BOOM! The hashes are different, the replay has been detected!

## Prove Replay With Spectograms

```bash
sox output_clean/*.wav -n spectrogram -o results/clean.png
```

```bash
sox output_replay/*.wav -n spectrogram -o results/replay.png
```

```bash
xdg-open results/clean.png
```

```bash
xdg-open results/replay.png
```

- Put the newly opened **Spectograms** side by side

<img width="1920" height="908" alt="image" src="https://github.com/user-attachments/assets/55d48b78-7000-41ed-ad9e-69c99579ea2f" />

- Again, suspiciously similar, but not quite...

***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/blueLabs/EphemeristaLab/Ephemerista.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/blueLabs/DefendingOdyssey/DefendingODYSSEYLab.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!
