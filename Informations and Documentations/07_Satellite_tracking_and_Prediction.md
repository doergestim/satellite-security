# Satellite Tracking & Prediction


## Two-Line Elements (TLEs) & NORAD catalogs 

**What a TLE is:**
A Two-Line Element set (TLE) is a compact textual representation of an orbit. Each TLE encodes a set of orbital elements and small fitted parameters intended for use with analytic propagators like SGP4/SDP4. TLEs are compact, easy to exchange, and have been the de facto standard for tracking objects in Earth orbit for decades.

**Where they come from:**
National or commercial tracking organizations (historically NORAD / US Space Force, plus many observatories and commercial operators) produce and distribute element sets and maintain catalogues (NORAD Catalog / USSF Catalog). Public services (ex: Celestrak, Space-Track) re-publish or provide access to subsets of these catalogs.

**TLE structure:**
A TLE consists of two lines of ASCII text. Each line uses fixed columns to store fields. Important fields and their meaning:
- **Satellite catalog number (NORAD ID)** — unique identifier for the object in the catalog.
- **Epoch** — the timestamp (year and fraction-of-day) at which the elements are valid.
- **Mean motion** — the number of orbits per day (from which orbital period is derived).
- **Eccentricity (in compact form)** — small decimal fields in fixed format describing orbit shape.
- **Inclination, RAAN, argument of perigee, mean anomaly** — elements describing orbit orientation and satellite position at epoch.
- **First derivative of mean motion, second derivative, BSTAR drag term** — fitted coefficients representing short-term perturbations (atmospheric drag, long-period effects) used by SGP4.
- **Checksum digits** at the end of each line to detect transcription errors.

**Important TLE caveats:**
- TLEs are *mean elements fitted to a specific propagation model* (SGP4), not instantaneous osculating Keplerian elements. Using them in higher-fidelity numeric integrators without appropriate conversions will produce inconsistent results.
- The epoch is critical: TLE accuracy is highest near the epoch and degrades over time as perturbations accumulate.
- TLEs can be updated frequently for tracked objects — catalog updates are essential for maintaining accurate predictions, especially for objects affected by high drag or maneuvers.


>[!IMPORTANT]
>
>TLEs and catalog entries are observable public metadata about objects. Adversaries can use them to plan observations, jamming, or signal-intelligence activities. Conversely, discrepancies between observed and catalog positions can reveal hidden maneuvers, collisions, or catalog errors.

---

## SGP4 / SDP4 propagators //how tracking works

**What SGP4/SDP4 do:**
SGP4 (Simplified General Perturbations 4) is an analytic propagator designed to work with TLEs and produce time-tagged satellite positions and velocities quickly and with modest accuracy suitable for many tracking tasks. SDP4 is the complementary model for deep-space objects where longer period and lunar/solar perturbations matter.

**Key characteristics:**
- **Analytic approximation:** SGP4 encodes a set of approximations for Earth's gravity field (including main oblateness effects), tidal-like terms, and empirically fitted drag/resonance corrections. It is intentionally fast and designed to match the mean elements in TLEs.
- **Input form:** SGP4 expects TLE-format elements (mean motion, eccentricity, inclination, etc.). The propagator applies periodic corrections and computes the spacecraft position/velocity at requested epochs.
- **Output:** the propagator yields Earth-centered position and velocity in a defined inertial frame, suitable for conversion to azimuth/elevation, range, and Doppler.

**Limitations to keep in mind:**
- **Model fidelity:** SGP4 approximates perturbations; it cannot capture high-frequency or small-scale effects (ex: variable atmospheric density, SRP for unusual shapes, small maneuvers) with high precision.
- **TLE fitting origin:** TLEs themselves often are produced by fitting observed tracking data into the SGP4 model. Thus SGP4+TLE is self-consistent, but not necessarily physically exact.

>[!TIP]
>
>when you see a mismatch between predicted SGP4 positions and observed RF signals, consider: TLE age, unmodeled maneuvers, sensor errors, and catalog misidentification.

---

## Orbit prediction accuracy & limitations 

**Factors that reduce prediction accuracy:**
- **Age of the TLE**: predictions diverge with time from the TLE epoch as unmodeled effects accumulate.
- **Atmospheric variability**: drag in LEO depends strongly on solar activity and atmospheric models; unpredictable variations cause decay and perigee changes that TLEs must be frequently updated to track.
- **Maneuvers & control actions**: any intentional thrusting changes orbital elements abruptly and invalidates older TLEs until a new fit is published.
- **High area-to-mass ratio objects**: lightweight objects (frames, debris, thin panels) are especially sensitive to SRP and atmospheric drag, producing larger prediction errors.
- **Resonances & tesseral terms**: certain altitudes and inclinations can produce sensitive long-term behavior not fully captured by simple analytic models.

**What accuracy actually means:**
- **Position error** might be expressed in kilometers; for many LEO objects, a TLE+SGP4 prediction may be accurate within a few km near the epoch but could drift to tens of km or more over weeks without updates.
- **Timing error** in pass predictions (AOS/LOS) depends on geometry and elevation mask; small positional errors map into seconds-to-minutes of timing discrepancies for low-elevation passes.

**Mitigation strategies:**
- Use frequent catalog updates for targets of interest.
- Complement TLE/SGP4 with local tracking observations to update ephemerides (orbit determination) when higher accuracy is needed.


>[!TIP]
>
>For critical correlation tasks (ex: matching RF captures across stations), use precise time synchronization and, if possible, higher-fidelity ephemerides (ex: precise ephemeris products or numerical propagation with measured initial states).

---

## Pass predictions: AOS, LOS, and maximum elevation //how they are found


- **AOS (Acquisition of Signal)**: the time when the satellite rises above the horizon/elevation mask of a ground station and a link becomes possible.
- **LOS (Loss of Signal)**: the time when the satellite drops below the elevation mask.
- **Maximum elevation (El_max)**: the highest elevation angle reached during a pass — often coincident with the moment of closest slant range.

**How pass times are computed\:**
1. **Propagate the satellite position** from its orbital elements to a set of candidate times over the desired window.
2. **Transform positions to the ground station’s local frame** and compute topocentric elevation angle at each time.
3. **Find crossing events** where elevation crosses the elevation mask threshold (ex: from below 5° to above 5°). A simple approach samples at short intervals and finds sign changes; more precise methods use interpolation or root-finding between samples to refine the AOS/LOS times.
4. **Locate maximum elevation** by finding the time when the elevation derivative crosses zero (or when range is minimal). Often a dense time-sampling followed by interpolation yields sufficient precision.

**Algorithmic & numerical notes:**
- **Sampling rate matters:** coarse sampling can miss short high-elevation passes or mis-estimate AOS/LOS times. Typical implementers sample at 10–30 second intervals and then refine around transition regions.
- **Root-finding precision:** bisection or Brent’s method applied to elevation(t)−mask = 0 gives robust crossing times.
- **Elevation masks & obstructions:** local terrain and antenna elevation masks must be taken into account; a satellite may be geometrically above the horizon but obscured by obstacles.

>[!IMPORTANT]
>
> timing of AOS/LOS is used to schedule command uplinks, correlate signals from different observers, and plan intercept or monitoring activities — errors in AOS/LOS can shift these operations significantly.

---

## Doppler shift prediction & correction

**Physical basis (plain formula):**
The observed instantaneous carrier frequency shift (Doppler) is approximately proportional to the relative radial velocity between transmitter and receiver:

Δf ≈ (v_radial / c) × f_c

where Δf is the frequency shift (Hz), v_radial is the radial velocity toward the receiver (m/s, positive when closing), c is the speed of light, and f_c is the carrier frequency (Hz).

**Example:**
- Suppose a satellite has a radial closing velocity of 7,000 m/s at closest approach, and the downlink carrier is 437.5 MHz.
- Compute v/c = 7,000 / 299,792,458 = 0.000023345... (about 2.3345×10⁻⁵).
- Δf = 437,500,000 × 0.000023345... ≈ 10,215.4 Hz.

So the carrier can shift by order of ±10 kHz at UHF frequencies for typical high-velocity LEO passes. At higher RF bands (L-band, S-band, X-band), the Doppler magnitude scales linearly with carrier frequency.

**How to correct Doppler:**
- **Precomputed Doppler profile:** use orbital predictions to compute expected Doppler as a function of time for the pass and feed offsets to the receiver to tune automatically.
- **Real-time tracking:** use PLLs or frequency-locked loops inside demodulators to follow residual frequency drift.
- **Coarse + fine approach:** combine coarse precomputed offsets with fine receiver tracking for best results.

**Errors & non-idealities:**
- If the orbital ephemeris is wrong (agey TLE or wrong initial state), the predicted Doppler profile will diverge and tracking will require larger PLL capture ranges or manual correction.
- Non-radial motion components do not contribute to Doppler but can change geometry; reflectors or multipath rarely matter in free-space LEO links but can appear in ground-side complex environments.

>[!IMPORTANT]
>
>mismatched Doppler profiles are a powerful indicator of spoofing or replay attacks. An injected signal that fails to match the predicted Doppler as the satellite moves is easily flagged by a system that cross-checks frequency evolution against a trusted ephemeris.

---

## Automated antenna control & integration 

Automated antenna control systems integrate pass predictions with rotator controllers and optionally with SDR frequency-control systems. the chain is:

1. **Predict pass & Doppler** for the ground station and target.
2. **Send azimuth/elevation commands** to the rotator to follow the satellite in real time. //track
3. **Apply Doppler offsets** to the SDR or receiver to keep the signal centered in the demodulator bandwidth.
4. **Log telemetry, timing, and rotator state** for audit and post-analysis.

**considerations & best practices:**
- **Authentication & access control:** prevent unauthorized actors from commanding rotators or frequency offsets. A misused rotator can prevent authorized reception or help an attacker by pointing to different targets.
- **Verify ephemeris sources:** accept ephemeris updates only from trusted sources to avoid feeding manipulated TLEs that could cause mispointing or conceal maneuvers.
- **Sanity checks:** implement limits on command rates, allowed az/el ranges, and abrupt jumps; log and alert on anomalous commands.
- **Redundancy & manual override:** always provide a secure manual override to retake control if automation fails or appears compromised.

---

## Cross-station correlation and multi-observer strategies

- **Why correlate:** combining observations from multiple geographically dispersed ground stations increases coverage, improves orbit determination, and helps detect deceptive signals (a spoofed signal that appears only at one station but not at others is suspicious).
- **Time & frequency sync:** precise time synchronization (GPS-disciplined clocks, NTP/PTP with authentication) and consistent reference frames are required to meaningfully combine Doppler and TOA data.
- **Data fusion & orbit determination:** collating range, Doppler, and angular measurements feeds orbit-determination filters (ex: Kalman / batch least squares) that can produce refined ephemerides and identify maneuvers.

---

## diagnostic checklist

- If predicted pass is missing or late: check TLE age, recent catalog updates, and whether the object maneuvered.
- If Doppler tracking drifts unexpectedly: verify time synchronization, check for incorrect ephemeris, and consider whether the transmitter has changed frequency or the satellite attitude has altered a directional antenna.
- If AOS/LOS times disagree across stations: check time-scale mismatches (UTC vs GPS), site coordinate errors, and elevation-mask inconsistencies.

---


### Glossary // quick terms

- **TLE:** Two-Line Element set (orbital elements format).
- **NORAD / USSF Catalog:** authoritative catalog of tracked space objects (catalog number / NORAD ID).
- **SGP4 / SDP4:** analytic propagators for near-Earth / deep-space TLE propagation.
- **AOS / LOS:** Acquisition / Loss of Signal.
- **El_max:** Maximum elevation during a pass.
- **v_radial:** radial component of relative velocity used in Doppler computations.


