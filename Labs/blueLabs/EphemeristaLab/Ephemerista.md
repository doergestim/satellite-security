![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)



# Ephemerista

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
python - << 'EOF'
import ephemerista
print(ephemerista)
EOF
```

<img width="1088" height="29" alt="image" src="https://github.com/user-attachments/assets/548067b2-cbec-4ffa-852c-f04098b25400" />


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

## Loading Ephemerista Data

### Using Two-Line Element (TLE) Data

>[!IMPORTANT]
>Everytime you run a python code, either use an IDE like **Visual Studio**, or run it in console like this: `nano script.py` -> **Ctrl + X** and **Y** and **Enter** to save an exit -> `python3 script.py` to run it -> `rm script.py` to delete it to then make another one

```python
from ephemerista import Ephemeris

ephem = Ephemeris("satellite.bsp")
print(ephem)
```

>[!NOTES]
> You don't have to run these python scripts necessarily, we are using them as examples to understand the logic behind **Ephemerista**

Typical attributes available:

```python
ephem.epoch
ephem.frame
ephem.metadata
```

You should always inspect the epoch and frame before performing calculations

---

## Querying Satellite Position

### Position at a Specific Time

```python
from datetime import datetime, timezone
from ephemerista import SPK

spk = SPK("example.bsp")

t = datetime(2026, 1, 1, 12, 0, tzinfo=timezone.utc)
state = spk.state(
    target="SATELLITE",
    observer="EARTH",
    time=t,
    frame="J2000"
)

print(state.position)
```

Returned values are vector objects, usually in meters.

### Position Over a Time Window

```python
positions = []
for hour in range(0, 24):
    t = datetime(2026, 1, 1, hour, 0, tzinfo=timezone.utc)
    positions.append(ephem.position_at(t))
```

This pattern is commonly used for pass prediction and trajectory analysis.

---

## Velocity and Orbital Dynamics

### Querying Velocity

```python
velocity = ephem.velocity_at(t)
print(velocity)
```

**Velocity** magnitude for **LEO satellites** typically ranges between **7–8 km/s**

### Sanity Check Example

```python
speed = velocity.magnitude()
assert 6500 < speed < 8500
```

Sanity checks like this are simple but effective for catching corrupted data

---

## Coordinate Frames

Ephemerista supports multiple coordinate frames, commonly:

- **ECI** (Earth-Centered Inertial)
- **ECEF** (Earth-Centered Earth-Fixed)

### Transforming Between Frames

```python
eci = ephem.position_at(t, frame="ECI")
ecef = ephem.transform(eci, to="ECEF")
```

**Frame mismatches** are a frequent source of subtle but serious errors

---

## Doppler Effects and Signal Impact

### Estimating Doppler Shift

```python
def doppler_shift(relative_velocity, carrier_frequency):
    c = 299_792_458
    return carrier_frequency * (relative_velocity / c)

shift = doppler_shift(7500, 2.2e9)
print(shift)
```

Incorrect ephemeris data directly translates to incorrect Doppler compensation, often resulting in degraded or lost links

---

## Comparing Ephemeris Sources

Using multiple ephemeris sources is common in real systems.

```python
ephem_a = Ephemeris.from_tle("source_a.tle")
ephem_b = Ephemeris.from_tle("source_b.tle")

delta = ephem_a.position_at(t) - ephem_b.position_at(t)
print(delta.magnitude())
```

Interpretation:
- **Tens of meters**: normal variation
- **Hundreds of meters**: investigate
- **Kilometers**: likely stale or incorrect data

---

## Time Sensitivity and Error Propagation

### Introducing a Timing Offset

```python
tampered = ephem.copy()
tampered.epoch_offset(seconds=1)

error = tampered.position_at(t) - ephem.position_at(t)
print(error.magnitude())
```

Even sub-second **offsets** can cause errors large enough to:
- **Miss** acquisition windows
- **Break** tracking filters
- **Desynchronize** receivers

### Progressive Offset Analysis

```python
for offset in [0.1, 0.5, 1, 2]:
    test = ephem.copy()
    test.epoch_offset(seconds=offset)
    err = test.position_at(t) - ephem.position_at(t)
    print(offset, err.magnitude())
```

---

## Defensive Validation Patterns

### Basic Bounds Checking

```python
pos = ephem.position_at(t).magnitude()
vel = ephem.velocity_at(t).magnitude()

assert pos > 6_000_000
assert vel < 9_000
```

### Rate-of-Change Monitoring

```python
p1 = ephem.position_at(t)
p2 = ephem.position_at(t.replace(minute=t.minute + 1))

delta = (p2 - p1).magnitude()
assert delta < 600_000
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
