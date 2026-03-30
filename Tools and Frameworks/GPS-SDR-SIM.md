# GPS-SDR-SIM

> **Tool:** GPS-SDR-SIM 
>
> **Link:** https://github.com/osqzss/gps-sdr-sim
>
>  GPS‑SDR‑SIM is an open-source GPS baseband signal simulator that generates digitized I/Q samples for GPS L1 (and variants) from user-specified satellite ephemerides, receiver trajectories, and timing. The generated baseband files can be transmitted using SDR hardware (ADALM‑Pluto, HackRF, BladeRF, USRP) or used offline for testing receivers.

---

## Overview 

GPS‑SDR‑SIM numerically generates GNSS baseband signals (pseudorandom codes modulated on carriers) and creates I/Q sample files you can feed to a transmitter or use in analysis tools. Practically, this enables:

- Controlled, repeatable generation of GNSS signals for testing receiver behavior.
- Reproducing specific spoofing scenarios. //time offsets, gradual takeovers, time jumps
- Creating test vectors and known-good captures for receiver validation, regression tests, and training ML detectors.


>This tool is widely used in academic and hobbyist GNSS research and appears across many community projects and forks that add real-time streaming, expanded signal types, or enhanced transmitter toolchains.

---

## Key capabilities

- **Generate baseband I/Q** for configurable sets of satellites and receiver trajectories. //static or moving 
- **Specify simulation time & position:** choose start time, receiver position (lat/lon/alt), and receiver clock bias for repeatability.  
- **Multiple output formats:** produces raw I/Q files that can be converted for use with different SDRs (HackRF, BladeRF, ADALM‑Pluto, USRP).  
- **Real-time / streaming add-ons exist:** community projects wrap gps‑sdr‑sim to stream to SDRs in real time (Pluto wrappers, hackrf_transfer pipelines).  
- **Parameter control for spoofing scenarios:** adjust signal power, relative delays, and satellite subsets to emulate partial or full spoofing attacks.

---

## Why this and satellite security?

- **Threat modeling:** GPS spoofing can manipulate positioning and timing — both critical to satellite ground stations (antenna pointing, scheduling) and many infrastructure components (timing for logging, synchronization). Having a simulator lets you model adversary capabilities and realistic attack modes.  
- **Defensive testing:** create repeatable spoofing cases to harden receivers, evaluate anti‑spoofing algorithms (power/time checks, multi‑antenna correlation), and stress-test downstream systems that rely on GNSS.  
- **Forensics & reproducibility:** simulated captures serve as ground-truth evidence to reproduce incidents and validate detection pipelines.  

---

## use  cases

1. **Receiver robustness validation:** generate scenarios with small clock drifts, sudden time jumps, or coherent takeover where spoofed signals pull a receiver off real satellites. Use these to evaluate receiver lock behavior and failover logic.  
2. **Timing infrastructure tests:** feed synthetic GNSS signals to timing servers or NTP/PTP client, leap second handling, and authentication measures.  
3. **Operator training** run realistic spoofing exercises in an RF‑shielded facility to train operators on detection and incident response without risking real-world disruption.  
4. **ML detector training sets:** generate labeled examples of spoofing vs. clean traces for supervised learning of spoofing detectors.  
5. **Integration testing for ground-control systems:** verify that antenna control, scheduling software, or ground-station automation behaves safely when presented with manipulated GNSS-derived telemetry.  

---

## Recommended hardware & setup 

- **Transmitter SDRs:** ADALM‑Pluto (popular), HackRF One (low-cost but less stable oscillator), BladeRF, or USRP (best timing & stability). 

>[!TIP]
>
> Some SDRs require attention to sample rate and format conversions.  

- **Clock stability:** use a high‑quality TCXO/OCXO or external reference where possible; cheap SDR oscillators (ex: stock Pluto or HackRF) introduce frequency offsets/drift that can distort experiments.  
- **Sample rates:** many users run ~2.6 MS/s or similar rates depending on the downstream transmitter pipeline; check SDR-specific tooling for exact required format (signed bytes vs signed 16‑bit).  
- **Shielded lab:** always transmit simulated GNSS only inside a Faraday cage or RF‑shielded room, or use a wired RF loopback, to avoid illegal over‑the‑air spoofing.  
- **Antenna & front-end:** inside labs use attenuators and direct cable feeds; for non‑transmit uses, feed the generated baseband directly into receiver chains or record it for offline analysis.  

---

## How to use GPS‑SDR‑SIM 

### Offline generation 

1. Prepare satellite ephemerides and navigation data (RINEX or built-in ephemeris support).  
2. Create a receiver trajectory file (NMEA or position/time parameters).  
3. Run gps-sdr-sim to generate a baseband I/Q file for the specified time and location.  
4. Convert/format the I/Q for your target SDR (if needed) and either feed into an SDR transmitter in a shielded lab or into offline decoders for analysis.  

### Real-time streaming to SDR

1. Use community wrappers  to stream generated I/Q to the SDR in real time.  
2. Apply frequency corrections or use external reference clocks to reduce carrier offset.  
3. Monitor receiver under test and collect logs/outputs for analysis.  

*(Exact CLI flags vary by fork/version — consult the project README and wrapper scripts for up-to-date usage patterns.)*

---

## Integration with other tools

- **GNU Radio:** feed simulated captures into GNU Radio for signal‑level inspection, FEC behavior analysis, or to chain into larger simulation flowgraphs.  
- **RTK/receiver firmware testing:** use the simulator to produce controlled conditions for validating firmware responses, NMEA outputs, and position fixes.  
- **SIEM & incident pipelines:** tag simulated incidents and use them to validate detection logic and SOC playbooks.  
- **Hardware-in-the-loop:** pair GPS‑SDR‑SIM with antenna rotator control and ground-station automation tools to validate end-to-end resilience under spoofed GNSS conditions.  

---

## Detection & countermeasures

- **Multi-antenna / spatial correlation:** correlate signals from two or more spatially separated antennas to detect coherent counterfeit signals (spoofers are often single-point transmitters).  
- **Power & consistency monitoring:** monitor sudden jumps in received signal power or implausible satellite geometry (ex: all signals arriving with similar power/phase).  
- **Time/frequency checks:** detect atypical Doppler behavior or inconsistent carrier offsets across satellites that don't match orbital motion.  
- **Cross-check with other sensors:** inertial measurement units (IMUs), odometry, or alternative timing sources (cellular, eLoran) to spot GNSS discrepancies.  
- **Cryptographic / authenticated signals:** where available, favor validated GNSS signals (ex: GPS M-code, Galileo OSNMA) and validate authenticated PVT where possible.  


---

## Threats, dual-use, and ethics

- **Dual‑use:** GPS‑SDR‑SIM is explicitly dual‑use — it simplifies creating spoofing scenarios. Public availability means researchers and malicious actors both have access.  
- **Legal risks:** transmitting spoofed GNSS signals over the air is illegal in many jurisdictions and can endanger safety‑critical services.   
- **Responsible disclosure / research norms:** when discovering receiver vulnerabilities using this tool, follow coordinated disclosure to affected vendors/operators and avoid public release of active exploit scripts that enable misuse.  

---

## Operational/hardening recommendations

- **Laboratory controls:** require lab sign‑offs, use RF shields, document all tests, and restrict transmit-capable hardware to authorized personnel.  
- **Receiver hardening:** implement multi‑antenna checks, signal‑consistency tests, and fallback timing sources for critical ground-station infrastructure.  
- **Monitoring & alerting:** instrument receivers and timing-dependent systems to emit alerts when GNSS-derived values change abruptly or conflict with expected behavior.  
- **Forensics:** preserve generated I/Q, SDR logs, and configuration files (ephemerides, seed parameters) to reproduce test scenarios during investigations.  

---

## Practical pitfalls & troubleshooting

- **Oscillator drift:** cheap SDRs introduce frequency and phase drift — compensate with external reference clocks or calibrate and log offsets.  
- **Sample format mismatches:** tools may expect different I/Q formats (signed 8‑bit, signed 16‑bit); check converters or wrapper scripts.  
- **GPU/CPU load for real-time:** streaming generation to SDRs in real time can be CPU‑intensive on some platforms; consider pre‑generating files where possible.  
- **Compatibility forks:** many community forks add features (real‑time streaming, L2/L5 support); check forks and issues for hardware-specific tips ;)


---



