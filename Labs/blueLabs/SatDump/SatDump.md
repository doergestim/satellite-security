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

Now, let's make the following changes:

1. Pipepline: `Analog Demodulation`
2. Input File: `/home/ubuntu/Desktop/SatDumpLab/pass_clean.iq`
3. Output Directory: `/home/ubuntu/Desktop/SatDumpLab/output_clean/`
4. Baseband Format: `cs8`
5. Samplerate: `48000`
6. Doppler Correction: `Unchecked`


<img width="961" height="695" alt="2026-03-19_15-23_1" src="https://github.com/user-attachments/assets/8cd7f961-e0a7-4c49-b2e8-52e95935972e" />


- Start

Verify:

```bash
ls -lah output_clean
```

<img width="726" height="91" alt="2026-03-19_14-09" src="https://github.com/user-attachments/assets/d44fee64-3fb0-4859-850c-c7974e94033a" />


---

# Part B - Decode REPLAYED IQ

Repeat **Part A** using:

- IQ file: `/home/ubuntu/Desktop/SatDumpLab/pass_replay.iq`
- Output: `/home/ubuntu/Desktop/SatDumpLab/output_replay/`

Verify:

```bash
ls -lah output_replay
```

<img width="723" height="93" alt="2026-03-19_15-27" src="https://github.com/user-attachments/assets/b33cd213-55d2-4bc5-a163-3eb262b8c8c0" />



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

<img width="1049" height="111" alt="2026-03-19_15-29" src="https://github.com/user-attachments/assets/2ccabd9b-3ea9-44ac-958c-3c3296387c65" />


- BOOM! The hashes are different, the replay has been detected!

## Prove Replay With Spectograms

```bash
sudo apt install sox -y
```

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

<b><i>Want to go back? </br>[Previous Lab](/Labs/blueLabs/DefendingOdyssey/DefendingODYSSEY_API&DoS-Protection.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
