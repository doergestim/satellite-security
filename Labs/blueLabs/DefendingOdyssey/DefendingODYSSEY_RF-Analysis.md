![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Blue Lab 1 - Defending ODYSSEY-1: RF Defense

**Scenario:** You are **blue team** for ODYSSEY-1.  
Red team has:
- Jammed your BFSK downlink (`pass_jam_0dB.iq`, `pass_jam_-5dB.iq`)

In this lab you will:

1. Use **SDR tools** to **see and understand** the jamming at RF layer

You already have under `~/Desktop/DefendingODYSSEY`:
- `pass_clean.iq`
- `pass_jam_0dB.iq`
- `pass_jam_-5dB.iq`

---

# Part A - RF: Visually Detect & Understand the Jamming

## Compare clean vs jammed signals in Gqrx

1. Open **Gqrx**, run

```bash
gqrx &
```
In another Terminal:

- Get the path of the files:

```bash
cd ~/Desktop/DefendingODYSSEY
```

```bash
realpath pass_clean.iq
# /home/ubuntu/Desktop/DefendingODYSSEY/pass_clean.iq
```

```bash
realpath pass_jam_0dB.iq 
# /home/ubuntu/Desktop/DefendingODYSSEY/pass_jam_0dB.iq
```

```bash
realpath pass_jam_-5dB.iq 
# /home/ubuntu/Desktop/DefendingODYSSEY/pass_jam_-5dB.iq
```

- If this window pops up, go to **File** -> **I/O Devices**

2. Select **Complex Sampled (IQ) File** Device
3. Input rate: **48000**
4. Device string: `file=/home/ubuntu/Desktop/DefendingODYSSEY/pass_clean.iq,freq=437.5e6,rate=48000,repeat=true,throttle=true`

<img width="658" height="474" alt="image" src="https://github.com/user-attachments/assets/521aa8fe-4b51-4b75-a993-b793efc6b543" />

- Click **Ok**

5. Observe clean **BFSK tones** by pressing the **Arrow(play)** Button on the top-left

<img width="1090" height="698" alt="image" src="https://github.com/user-attachments/assets/0ebea7fd-b7eb-4e34-8da5-794ade284574" />


- Then repeat for (by replacing `pass_clean.iq` in the **Device string**):

- `file=/home/ubuntu/Desktop/DefendingODYSSEY/pass_jam_0dB.iq,freq=437.5e6,rate=48000,repeat=true,throttle=true`

<img width="1089" height="697" alt="image" src="https://github.com/user-attachments/assets/c8471a7d-a1d1-4c90-8934-2a3f1cda786b" />

-  `file=/home/ubuntu/Desktop/DefendingODYSSEY/pass_jam_-5dB.iq,freq=437.5e6,rate=48000,repeat=true,throttle=true`

<img width="1090" height="737" alt="image" src="https://github.com/user-attachments/assets/59dd63f9-5f66-4646-a1e9-66425a652cca" />


- Do it by going to **File** -> **I/O Devices**

Watch for:
- Raised noise floor  
- Smeared tones  
- Loss of clarity in waterfall  


# Clean Signal - Analysis

### Observations
- Two strong vertical stripes indicating clear BFSK tones.
- Noise floor is low, roughly –60 to –50 dBFS.
- Signal stands well above the noise, giving high SNR.
- Waterfall is stable and organized.
- Symbol transitions look sharp and well-defined.

### Interpretation
- This is a healthy downlink.
- The demodulator will lock easily and decode reliably.
- Represents normal, expected operational conditions.


# Jammed Signal (0 dB) - Analysis

### Observations
- BFSK tones are still visible but partially submerged.
- Noise floor is raised significantly, around –40 to –35 dBFS.
- Waterfall shows extra broadband noise.
- Tone edges are less distinct.
- SNR is reduced but not destroyed.

### Interpretation
- This is a partial denial situation.
- Demodulator may still decode some packets but with higher BER.
- Operators would see intermittent or glitchy telemetry.
- Packet loss is likely but not total.


# Jammed Signal (–5 dB) - Analysis

### Observations
- Noise floor is almost equal to or above the signal.
- BFSK tones are faint and hard to distinguish.
- Waterfall is nearly uniformly bright, indicating strong wideband jamming.
- Symbol structure is buried in noise.
- Effective SNR is below 0 dB.

### Interpretation
- This is a total denial scenario.
- Demodulator will not lock or decode correctly.
- Operators would experience complete loss of telemetry.
- Groundstation would show stale or missing frames.

---

## Inspect symbols in Inspectrum

1. Start **Inspectrum**

```bash
inspectrum &
```

2. Set sample rate: **48000**
3. Load `pass_clean.iq` by pressing **Input file** and selecting that file

![image](/Assets/BLab1/BLab1-5.png)

4. What to look for:

### Identify the two BFSK tones
- Look for two horizontal bright lines (top tone and bottom tone).
- These represent the two frequencies used for BFSK (Mark and Space).
- In a clean signal, both tones should be stable and sharp.

### Verify that the tones switch at regular intervals
- BFSK encodes bits by switching between the two tones.
- In Inspectrum, this appears as alternating bright segments between the two lines.
- Good signal = clear, rectangular transitions between tones.

### Locate symbol boundaries
- Use the horizontal time axis.
- Each repeated "block" of patterns corresponds to symbols or frames.
- Clean signals show:
  - Repeating sections
  - Well-defined on/off transitions
  - Minimal blurring

### Look for the preamble structure
- Satellite packets often start with a known pattern (e.g., alternating tones).
- These appear as highly regular, tightly spaced transitions at the start of each frame.
- In your screenshot, these are the short, very crisp segments at the left of each block.

### Check for noise or corruption
- Clean signal:
  - Background is dark blue/green.
  - Tones stand out clearly in bright yellow.
- Jammed or degraded signal:
  - Background becomes grainier.
  - Tones smear or lose sharpness.
  - Transitions look "fuzzy" or inconsistent.

### Confirm symbol rate consistency
- The distance (time) between tone transitions should be constant.
- Clean signal:
  - Blocks are evenly spaced.
  - Symbol spacing is identical throughout.
- Jammed signal:
  - Harder to see transitions.
  - Blocks may look smeared over time.

---

- Repeat for jammed files; observe corruption of symbol structure.



***                                                                 

<b><i>Continuing the course? </br>[Next Lab](./DefendingODYSSEY_API_And_DoS-Protection.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/Lab5/The_Drift.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
