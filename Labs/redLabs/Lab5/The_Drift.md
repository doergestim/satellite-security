![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 5 — The Drift (Orbital Analysis & Manipulation)

**Goal:** Use **Orekit** to propagate a satellite from TLE, forge small ephemeris changes, compare **real vs forged** ground pointing (az/el), and see how a naive antenna controller would mis-point. Synthetic assets only

> Docker is required for the Orekit container; host Python is used for plotting

---

## Folder layout (what you have)

```
Lab5/
├─ assets/
│  ├─ ODYSSEY-1.tle             # used by Orekit propagation (LEO-like, synthetic)
│  └─ ODYSSEY-1_strict.tle      # strict 69-char TLE; importable into gpredict
├─ config/
│  └─ gs.json                   # groundstation location/time window
├─ docker/
│  └─ Dockerfile                # Orekit + Hipparchus + orekit-data container
├─ scripts/
│  ├─ run_analysis.py           # real vs forged propagation + pointing + error
│  ├─ visualize_pointing.py     # plots: az, el, total pointing error
│  ├─ peek_csv.py               # quick “head” for the output CSVs
│  └─ forge_tle.py              # write an explicitly forged TLE file
├─ emulators/
│  └─ antenna_controller_sim.py # prints naive motor setpoints from a plan
├─ outputs/                     # CSVs written here by run_analysis.py
```

---

## Objective

In this lab you will:

1. Use **Orekit** to propagate a satellite’s orbit from a TLE
2. Compare the “real” ground pointing angles to a **forged TLE**
3. Visualize the divergence in antenna pointing over time
4. Simulate how a naive antenna controller would follow the forged plan
5. Learn why **ephemeris tampering** is dangerous in real satellite operations, and how operators defend against it

---  

## Background

A **TLE (Two-Line Element set)** is a compact format that describes a satellite’s orbit. Groundstations use it to calculate when satellites pass overhead and where to point their antennas 

If an attacker **modifies a TLE** slightly (a few digits in mean motion or RAAN), the orbit prediction shifts. The groundstation may then:

- point its antenna in the wrong direction,
- miss the pass entirely,
- or waste precious downlink time trying to reacquire

This attack is attractive because **TLEs are small text files** — easy to inject or spoof if distribution channels are not authenticated

---

## Setup

Download the zip for this main folder from [Here](./Lab5.zip) ( Only if you are not using the VM )

- Click the Download button

<img width="330" height="177" alt="image" src="https://github.com/user-attachments/assets/df15f9ee-985a-4f6a-af65-32698e1aa337" />

- Extract it

### Create a Python venv for host-side tools and download the files
```bash
cd ~/Desktop/Lab5
```

```bash
python3 -m venv .venv
```

```bash
source .venv/bin/activate
```

```bash
pip install --upgrade pip
```

```bash
pip install numpy matplotlib
```

**Why:** Host Python renders plots and runs the mini controller sim, we keep Java/Orekit in Docker for consistency

### Add ODYSSEY‑1 to gpredict for pass planning

**GUI import:**  
```bash
gpredict
```

- gpredict -> **Edit -> Update TLE From Local Files…** -> choose `assets/`   
- Create a module and add **ODYSSEY-1** to see synthetic passes

<img width="840" height="567" alt="image" src="https://github.com/user-attachments/assets/b4ec4c17-aaed-404b-8cdd-4ac0ccb31419" />


**Why:** Gives context, you’ll still do the rest locally with synthetic data

<img width="854" height="907" alt="image" src="https://github.com/user-attachments/assets/0fb6f7f6-a45a-46e7-9167-a2da04322e1e" />


### Inspect inputs
Open and edit `config/gs.json` if desired:
```json
{
  "lat_deg": 52.5200,
  "lon_deg": 13.4050,
  "alt_m": 35.0,
  "start_iso": "2025-08-30T00:00:00",
  "duration_min": 120
}
```
- **lat/lon/alt**: groundstation position (Berlin)
- **start_iso**: analysis start time
- **duration_min**: propagation window

**Why:** Orbit prediction always depends on **where** you observe from and **when**

>[!IMPORTANT]
>Make sure you are in the `Lab5` directory under `~/Desktop/Lab5`

### Build the Orekit container
```bash
sudo docker build -t orekit-drift -f docker/Dockerfile .
```
**Why:** Encapsulates Java/Orekit/Hipparchus and public `orekit-data` so students don’t install JVM bits

### Run propagation: real vs forged
```bash
sudo docker run --rm -v "$(pwd):/work:Z" orekit-drift   python3 scripts/run_analysis.py   --tle assets/ODYSSEY-1.tle   --gs config/gs.json   --forge-offsets "dn=3e-4,draan=0.02"   --out outputs
```
**What it does:**
- Parses TLE and GS config.
- Propagates **real** track and writes:
  - `outputs/real_track.csv` (time, lat, lon, alt)
  - `outputs/real_pointing.csv` (time, az, el from your groundstation)
- Applies a small **forgery** (Δmean motion, ΔRAAN) and repeats:
  - `outputs/forged_track.csv`, `outputs/forged_pointing.csv`
- Computes **pointing_error.csv** with columns: `delta_az_deg, delta_el_deg, angular_error_deg`.

**Why:** In real ops, this forgery could be delivered by a compromised catalog or MITM attack, operators would then mis-point antennas


Quick peek:
```bash
python3 scripts/peek_csv.py --out outputs -n 5
```

<img width="765" height="443" alt="image" src="https://github.com/user-attachments/assets/ee5b31ff-fe0f-4d4d-b58f-46a3e7e372cd" />


### Visualize az/el and total pointing error
```bash
python3 scripts/visualize_pointing.py --out outputs
```

You’ll see three windows:
- **Azimuth vs time**: real vs forged

<img width="641" height="563" alt="image" src="https://github.com/user-attachments/assets/5382ac40-f30f-4bd4-afa4-8f47b398a50d" />

- **Elevation vs time**: real vs forged

<img width="641" height="563" alt="image" src="https://github.com/user-attachments/assets/66bb8967-c733-4a12-b37a-cdf035f14465" />

- **Total pointing error** (degrees) over time

<img width="641" height="563" alt="image" src="https://github.com/user-attachments/assets/011ecdae-bb75-452d-920e-0c10a0c7ad61" />


**Why:** Small ephemeris biases often produce multi‑degree errors near AOS/LOS—enough to miss a pass.

### Create your own forged TLE (explicit file)
```bash
python3 scripts/forge_tle.py   --in assets/ODYSSEY-1.tle   --out assets/ODYSSEY-1-forged.tle   --dn 5e-4   --draan 0.05
```

- Re-run analysis without inline offsets:
```bash
sudo docker run --rm -v "$(pwd):/work:Z" orekit-drift   python3 scripts/run_analysis.py   --tle assets/ODYSSEY-1-forged.tle   --gs config/gs.json   --forge-offsets ""   --out outputs
```

```bash
python3 scripts/visualize_pointing.py --out outputs
```

**Why:** Makes the columnar edits in line‑2 tangible; students see precisely how a tiny change impacts pointing

### Antenna controller simulator
```bash
python3 emulators/antenna_controller_sim.py --pointing outputs/forged_pointing.csv --rate_hz 2
```
- Prints lines like: `[timestamp] SET az=xxx.xx el=yy.yy`
- **Why:** Shows how blindly trusting a plan steers hardware off‑target

---

## Cleanup
```bash
deactivate || true
```


***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/blueLabs/BlueLab/BlueLab.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/Lab4/The_Relay.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---








































