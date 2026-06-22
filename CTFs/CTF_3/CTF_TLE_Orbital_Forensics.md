# Orbital Forensics CTF

**Assets provided:**
- `ODYSSEY-2.tle` - TLE pulled from the operator's authoritative catalog (the "real" orbit)
- `ODYSSEY-2-forged.tle` - a TLE intercepted on the groundstation's update channel,
  suspected of being tampered with
- `gs_config.json` - groundstation location and analysis window used to generate the
  evidence below
- `pointing_error.csv` - antenna pointing error (real vs forged) computed across the
  analysis window, columns: `time, delta_az_deg, delta_el_deg, angular_error_deg`

**Tools you will need:** a text editor, a calculator or Python, and (optional) an SGP4
library such as `sgp4` for Python if you want to reproduce the propagation yourself

**Scenario:** A groundstation operator flags that their antenna missed part of a
scheduled ODYSSEY-2 pass. They pulled the TLE their system used from the update channel
log and compared it against the catalog. You have both files and the pointing error
analysis already run for you. Work out what happened.

---

**Q1. Comparing `ODYSSEY-2.tle` and `ODYSSEY-2-forged.tle` line by line, which two
orbital elements differ between them?**

---

**Q2. What is the difference in mean motion (in revolutions/day) between the real and
forged TLE? Give your answer to 5 decimal places.**

---

**Q3. According to `pointing_error.csv`, what is the angular pointing error at the very
start of the analysis window (the first row)?**

---

**Q4. At what point in the analysis window does the angular pointing error first exceed
5 degrees? Give your answer as minutes:seconds from the start of the window.**

---

**Q5. What is the maximum angular pointing error recorded anywhere in
`pointing_error.csv`?**

---

**Q6. In the TLE format, which line contains the Right Ascension of the Ascending Node
(RAAN), and roughly which columns is it found in?**

---

**Q7. `ODYSSEY-2.tle` (the catalog copy) has a corrupted checksum on line 2. What is the
correct checksum digit, and what was the file shipped with instead?**

---

**Q8. Using the real TLE and an SGP4 propagator, what is the satellite's altitude (in
km) at the start of the analysis window defined in `gs_config.json`?**

---

**Q9. Using the satellite's orbital speed at that same moment, estimate how many metres
of position error a 1-second timing error would introduce. Show your working.**

---

**Q10. What groundstation coordinates (latitude/longitude) were used to compute the
pointing angles in this analysis? Where is this located?**
