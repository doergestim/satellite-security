![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)



# Ephemerista

>[!IMPORTANT]
>DISCLAIMER - This lab is more of a documentation and quiz, not a hands-on experience, you don't need to run any of the commands, all of the commands presented are examples

## Overview

This lab introduces **Ephemerista**, a library for working with satellite ephemeris data

You will learn how to load, inspect, analyze, and validate orbital data, then explore how ephemeris manipulation or misuse can affect satellite operations and security-critical systems

## Installation

```bash
pip install ephemerista
```

>[!NOTE]
> Usually, you would use it under a **python environment**, this is how you would do that in bash:


```bash
python3 -m venv venv
```

```bash
source venv/bin/activate
```

```bash
pip install ephemerista
```


Quick verification:

```bash
python3 - << 'EOF'
import ephemerista
print(ephemerista)
EOF
```


<img width="987" height="92" alt="2026-03-19_12-32" src="https://github.com/user-attachments/assets/c344a902-ee94-441a-89c1-7a15be12792e" />


---

## Ephemerista Data Fundamentals

Ephemerista data describes the predicted state of a satellite as a function of time. At minimum, this includes:

- **Position** (X, Y, Z)
- **Velocity** (Vx, Vy, Vz)
- **Reference frame**
- **Epoch** (time reference)

This data is **foundational** to:
- **Satellite** tracking
- **Ground station** pointing
- **Signal** timing and **Doppler** correction
- **Navigation** and **synchronization** systems

Small errors in ephemerista data often scale into large operational failures

---

## Initialization

Before using any Ephemerista functionality, you **must** initialize the library with three required data files:

- **Earth Orientation Parameters (EOP)** from the IERS
- **Planetary ephemerides** in SPK format (e.g. `de440s.bsp` from NASA JPL)
- **Orekit data package** for force models and time transforms

```python
import ephemerista

ephemerista.init(
    eop_path="finals2000A.all.csv",
    spk_path="de440s.bsp",
    orekit_data="orekit-data-main.zip"
)
```

>[!IMPORTANT]
> This initialization must be called **once per Python session** before anything else. Skipping it will cause errors in all downstream calculations.

>[!NOTE]
> The required data files can be downloaded from:
> - EOP: https://datacenter.iers.org/data/csv/finals2000A.all.csv
> - SPK: https://naif.jpl.nasa.gov/pub/naif/generic_kernels/spk/planets/de440s.bsp
> - Orekit: https://gitlab.orekit.org/orekit/orekit-data/-/archive/main/orekit-data-main.zip

---

## Loading Ephemeris Data

### Using Two-Line Element (TLE) Data

>[!IMPORTANT]
>Everytime you run a python code, either use an IDE like **Visual Studio**, or run it in console like this: `nano script.py` -> **Ctrl + X** and **Y** and **Enter** to save and exit -> `python3 script.py` to run it -> `rm script.py` to delete it to then make another one

TLEs are loaded as strings and passed directly to the `SGP4` propagator:

```python
import ephemerista
from ephemerista.propagators.sgp4 import SGP4
from ephemerista.time import TimeDelta

ephemerista.init(
    eop_path="finals2000A.all.csv",
    spk_path="de440s.bsp",
    orekit_data="orekit-data-main.zip"
)

iss_tle = """ISS (ZARYA)
1 25544U 98067A   24187.33936543 -.00002171  00000+0 -30369-4 0  9995
2 25544  51.6384 225.3932 0010337  32.2603  75.0138 15.49573527461367"""

propagator = SGP4(tle=iss_tle)
print(propagator.time)   # epoch parsed from the TLE
```

>[!NOTE]
> You don't have to run these python scripts necessarily, we are using them as examples to understand the logic behind **Ephemerista**

You should always inspect the epoch before performing calculations - `propagator.time` exposes the TLE epoch directly.

---

## Querying Satellite Position

### Position at a Specific Time

Propagate to a single point in time by passing a `Time` object:

```python
from ephemerista.propagators.sgp4 import SGP4
from ephemerista.time import Time, TimeDelta

propagator = SGP4(tle=iss_tle)

t = Time.from_iso("UTC", "2026-01-01T12:00:00")
state = propagator.propagate(time=t)

print(state.position)   # vector in km, Earth-centered
```

Returned position values are vectors, typically in kilometres.

### Position Over a Time Window

```python
start_time = propagator.time
end_time = start_time + TimeDelta.from_hours(24)
times = start_time.trange(end_time, step=float(TimeDelta.from_minutes(1)))

trajectory = propagator.propagate(time=times)
```

This pattern is commonly used for pass prediction and trajectory analysis. The returned `trajectory` object holds all state vectors across the time window.

---

## Velocity and Orbital Dynamics

### Querying Velocity

Each propagated state contains both position and velocity:

```python
state = propagator.propagate(time=t)

print(state.position)   # km
print(state.velocity)   # km/s
```

**Velocity** magnitude for **LEO satellites** typically ranges between **7–8 km/s**

### Sanity Check Example

```python
import numpy as np

speed = np.linalg.norm(state.velocity)
assert 6.5 < speed < 8.5, f"Unexpected speed: {speed} km/s"
```

Sanity checks like this are simple but effective for catching corrupted or stale data

---

## Coordinate Frames

Ephemerista supports multiple coordinate frames, commonly:

- **ECI** (Earth-Centered Inertial) - inertial, fixed to stars
- **ECEF** (Earth-Centered Earth-Fixed) - rotates with the Earth

The propagated trajectory carries its frame as an attribute:

```python
trajectory = propagator.propagate(time=times)
print(trajectory.frame)           # shows the current reference frame
print(trajectory.frame.abbreviation)
```

### Transforming Between Frames

Frame transformations are handled via Ephemerista's `coords` module:

```python
from ephemerista.coords.twobody import Cartesian

# The trajectory is natively in ECI; inspect its frame
print(trajectory.frame.abbreviation)   # e.g. "TEME" for SGP4 output

# Convert a single state to Cartesian for manual frame work
state = propagator.propagate(time=t)
print(state.position)
print(state.velocity)
```

**Frame mismatches** are a frequent source of subtle but serious errors - always verify `trajectory.frame` before mixing data from different sources.

---

## Doppler Effects and Signal Impact

### Estimating Doppler Shift

```python
def doppler_shift(relative_velocity_km_s, carrier_frequency_hz):
    c = 299_792.458   # km/s
    return carrier_frequency_hz * (relative_velocity_km_s / c)

import numpy as np
state = propagator.propagate(time=t)
speed = np.linalg.norm(state.velocity)   # km/s

shift = doppler_shift(speed, 2.2e9)
print(f"Max Doppler shift: {shift:.0f} Hz")
```

Incorrect ephemeris data directly translates to incorrect Doppler compensation, often resulting in degraded or lost links

---

## Comparing Ephemeris Sources

Using multiple ephemeris sources is common in real systems.

```python
from ephemerista.propagators.sgp4 import SGP4
from ephemerista.time import TimeDelta
import numpy as np

propagator_a = SGP4(tle=tle_source_a)
propagator_b = SGP4(tle=tle_source_b)

t = propagator_a.time
state_a = propagator_a.propagate(time=t)
state_b = propagator_b.propagate(time=t)

delta = np.linalg.norm(
    np.array(state_a.position) - np.array(state_b.position)
)
print(f"Position difference: {delta:.3f} km")
```

Interpretation:
- **Tens of metres**: normal variation
- **Hundreds of metres**: investigate
- **Kilometres**: likely stale or incorrect data

---

## Time Sensitivity and Error Propagation

### Introducing a Timing Offset

```python
from ephemerista.time import Time, TimeDelta
import numpy as np

t_nominal = propagator.time
t_offset  = t_nominal + TimeDelta.from_seconds(1)

state_nominal = propagator.propagate(time=t_nominal)
state_offset  = propagator.propagate(time=t_offset)

error = np.linalg.norm(
    np.array(state_nominal.position) - np.array(state_offset.position)
)
print(f"1-second offset causes {error*1000:.1f} m position error")
```

Even sub-second **offsets** can cause errors large enough to:
- **Miss** acquisition windows
- **Break** tracking filters
- **Desynchronize** receivers

### Progressive Offset Analysis

```python
for seconds in [0.1, 0.5, 1, 2]:
    t_test = t_nominal + TimeDelta.from_seconds(seconds)
    s_test = propagator.propagate(time=t_test)
    err = np.linalg.norm(
        np.array(state_nominal.position) - np.array(s_test.position)
    )
    print(f"Offset {seconds}s → {err*1000:.1f} m error")
```

---

## Defensive Validation Patterns

### Basic Bounds Checking

```python
import numpy as np

state = propagator.propagate(time=t)

pos_km = np.linalg.norm(state.position)
vel_km_s = np.linalg.norm(state.velocity)

assert pos_km > 6_000,  f"Position too low: {pos_km:.1f} km"
assert vel_km_s < 9.0,  f"Velocity too high: {vel_km_s:.3f} km/s"
```

### Rate-of-Change Monitoring

```python
t1 = propagator.time
t2 = t1 + TimeDelta.from_minutes(1)

s1 = propagator.propagate(time=t1)
s2 = propagator.propagate(time=t2)

delta_km = np.linalg.norm(
    np.array(s2.position) - np.array(s1.position)
)
assert delta_km < 600, f"Unexpected position jump: {delta_km:.1f} km"
```

---

## Quick quiz

### 1. What does ephemeris data primarily represent?
A. Satellite hardware configuration  
B. Satellite position and velocity over time  
C. Encryption parameters  
D. Ground station layout  


---

### 2. Why are timing errors dangerous?
A. They only affect logging  
B. They cause cosmetic visualization issues  
C. They scale into large positional errors  
D. They only affect deep-space missions  



---

### 3. Why perform ephemeris sanity checks?
A. To improve visualization quality  
B. To reduce CPU usage  
C. To detect corruption or manipulation  
D. To increase orbital altitude  



---

### 4. Which frame mismatch is most problematic?
A. ECI vs ECEF  
B. CSV vs JSON  
C. ASCII vs binary  
D. UDP vs TCP  



---

### 5. Why compare multiple ephemeris sources?
A. Redundancy and anomaly detection  
B. Faster computation  
C. Larger datasets  
D. Easier plotting  



---

Scroll for the **Answers**

<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>


1. **Correct answer: B**

2. **Correct answer: C**

3. **Correct answer: C**

4. **Correct answer: A**

5. **Correct answer: A**





***                                                                 

<b><i>Want to go back? </br>[Previous Lab](/Labs/blueLabs/SatDump/SatDump.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
