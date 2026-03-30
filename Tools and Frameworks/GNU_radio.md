# GNU Radio


>  Free, open-source software-defined radio (SDR) toolkit and signal-processing framework used to build receivers, transmitters, and monitoring systems. Widely used with SDR hardware (HackRF, RTL-SDR, USRP family, Funcube, LimeSDR) for satellite signal experimentation, demodulation, and research.


https://github.com/gnuradio/gnuradio
---

## 1. Overview

GNU Radio is a modular software toolkit for building signal-processing flowgraphs. It provides:

- A large set of DSP building blocks (filters, modulators/demodulators, decoders, synchronization blocks).
- A visual development environment (GNU Radio Companion — GRC) and Python/C++ APIs for programmatic flowgraphs.
- An ecosystem of out-of-tree (OOT) modules adding satellite-specific blocks (ex: gr-satellites, gr-dvbs2rx).

This makes GNU Radio a primary choice for prototyping satellite receivers, experimenting with modulation schemes used on satellites (AX.25, GMSK, BPSK, QPSK, QAM, DVB-S/S2), and building active monitoring or research tools relevant to satellite security.

---

## 2. Why GNU Radio matters for satellite security

- **Flexibility:** implement or modify demodulators/decoders to study proprietary or legacy satellite links.
- **Reproducibility:** full flowgraphs can be shared, versioned, and replayed against captured IQ recordings.
- **Interoperability:** works with many SDR front-ends (RTL-SDR, HackRF, USRP, LimeSDR), enabling both low-cost and research-grade setups.
- **Research & defense:** used in academic work (spoofing/jamming research, GNSS spoofing detection), and by hobbyists for telemetry/imagery decoding — making the same techniques available to both defenders and potential attackers.

---

## 3. Common satellite use-cases 

- **Weather satellite reception (NOAA APT, HRPT, GOES XRIT/XRIT2):** FM/AM demod → image decoding pipelines.
- **Amateur radio satellites (CubeSats, OSCAR):** AX.25 telemetry decoding (APRS), GMSK/BPSK demod, telemetry parsing.
- **DVB-S / DVB-S2 (satellite TV links):** demod → FEC → MPEG-TS extraction (useful for research or testing receiver resilience).
- **GNSS experimentation:** GNSS signal monitoring, spoofing detection research, and software GNSS receivers.
- **Telemetry and command (T&C) research:** building receivers for telemetry downlinks and experimenting with capture and decoding of frames (responsibly, within legal boundaries).
- **Doppler compensation:** real-time or offline frequency correction for LEO passes using TLE-derived frequency vs. time.

---

## 4. Hardware commonly paired with GNU Radio

- **Low-cost / hobbyist:** RTL-SDR (receive only), FUNcube Dongle, SDRplay
- **Mid-range:** HackRF One (transmit & receive, limited bandwidth), LimeSDR
- **Research / high-end:** Ettus USRP family (B200/B210/USRP X300/X310) — full duplex, wideband, and professional timing options

https://www.ettus.com/all-products/ub210-kit/

When working on satellite signals, consider front-end frequency coverage (L-band, S-band, VHF/UHF, LEO downlink frequencies), antenna gain, and the need for Doppler-tracking (frequency agility / rapid tuning).

---

## 5. Notable out-of-tree (OOT) modules & related projects

- **gr-satellites:** collection of decoders, doppler-correction helpers, and satellite utilities (popular for amateur satellites and beacons).
- **gr-dvbs2rx / gr-dvbs2:** DVB-S/S2 tx/rx implementations (useful for research on satellite broadcast signals).
- **gr-gps / gnss-sdr integrations:** components used in experimentation with GNSS.
- **Various community flowgraphs:** NOAA APT, GOES, Meteor-M2 decoding examples hosted in community repos and blog posts.

---

## 6. Satellite security considerations (threats enabled by GNU Radio)

GNU Radio is a dual-use tool: powerful for defenders and researchers, but it also lowers the barrier to perform attacks when combined with suitable RF hardware.

**Threat types:**

- **Interception / eavesdropping:** capture of unencrypted telemetry, imagery, or broadcast streams.
- **Spoofing:** generating false navigation (GNSS) or telemetry signals to mislead receivers; software-defined transmitters make adaptive spoofing prototypes easier to build.
- **Jamming / denial-of-service:** wideband or protocol-aware interference to disrupt satellite links.
- **Replay / relay attacks:** record-and-replay of signals to cause incorrect receiver behavior (timing-sensitive GNSS or telemetry protocols are susceptible).
- **Protocol manipulation / fuzzing:** modifying flowgraphs to fuzz satellite protocols and discover weaknesses in ground software.

**Why GNU Radio accelerates these threats:** open DSP blocks, ease of creating custom waveforms, and the ability to test against recorded captures in a lab.

---

## 7. Defensive & research uses //how GNU Radio helps defenders

- **Monitoring & anomaly detection:** build passive receivers to log signal strength, modulation changes, or unexpected carriers on known satellite bands.
- **Spoofing detection experiments:** implement cross-checks, multi-antenna correlation, or GNSS authentication research.
- **Protocol reverse-engineering:** reproduce expected telemetry framing to validate telemetry authenticity and detect injected frames.
- **Jamming localization / fingerprinting:** use time/frequency analysis with multiple receivers to identify interference sources.


---


## 8. Getting started (practical steps)

1. **Install GNU Radio:** use your platform package manager (apt, conda) or prebuilt binaries. Start with GNU Radio Companion (GRC) for visual flowgraph building.

```bash
# Debian/Ubuntu example
sudo apt update
sudo apt install gnuradio
```

2. **Connect an SDR front-end:** RTL-SDR for receive-only beginners; USRP or Lime for full-featured research.
3. **Explore example flowgraphs:** NOAA APT, AX.25 telemetry, or simple FM demod to understand pipeline stages (source → filter → demod → decode).
4. **Capture IQ recordings:** use them to iterate offline and avoid transmitting or interfering with real-world systems.
5. **Use doppler-correction tools:** combine with tracking software (Gpredict) or compute Doppler from TLEs for LEO passes.

---

## 9. Focused resources & further reading

 (official project pages, OOT modules, tutorial hubs, community networks, and academic work) 
 
 relevant to **GNU Radio** in the context of **satellite security**.  // installation, example flowgraphs, decoders, community ground-stations, and key research on GNSS spoofing/detection.

- **GNU Radio — official site**
  - Main: https://www.gnuradio.org
- **GNU Radio (GitHub)** — source, issue tracker, releases: https://github.com/gnuradio/gnuradio
- **GNU Radio Companion (GRC)** — visual flowgraph editor repo: https://github.com/gnuradio/gnuradio-companion
- **gr-satellites (OOT)** — telemetry decoders, doppler/time blocks, many amateur satellite decoders: https://github.com/daniestevez/gr-satellites
- **gr-dvbs2rx / gr-dvbs2** — DVB‑S/S2 tx/rx blocks (useful for broadcast/transport-stream research): https://github.com/igorauad/gr-dvbs2rx (docs: https://igorauad.github.io/gr-dvbs2rx/docs/usage.html)
- **SatNOGS (Libre Space)** — open, global network of ground stations and tools for scheduling/observations: https://satnogs.org (network: https://network.satnogs.org)
- **GNSS‑SDR (open GNSS receiver)** — software receiver framework useful for GNSS experimentation and spoofing research: https://gnss-sdr.org (GitHub: https://github.com/gnss-sdr/gnss-sdr)
- **RTL‑SDR.com (tutorial hub)** — practical tutorials (NOAA APT, quick starts, examples): https://www.rtl-sdr.com (NOAA tutorial: https://www.rtl-sdr.com/rtl-sdr-tutorial-receiving-noaa-weather-satellite-images/)
- **Ettus Research (USRP vendor & guides)** — USRP hardware and UHD/GNU Radio integration guides: https://www.ettus.com (UHD build/install KB: https://kb.ettus.com/Building_and_Installing_the_USRP_Open-Source_Toolchain_%28UHD_and_GNU_Radio%29_on_Linux)
- **Key academic references (GNSS spoofing & defenses)**
  - *GNSS Spoofing and Detection* — Psiaki & Humphreys (overview PDF): https://radionavlab.ae.utexas.edu/images/stories/files/papers/gnss_spoofing_detection.pdf
  - *GPS Spoofing Detection via Dual-Receiver Correlation* — Psiaki et al. (PDF): https://gps.mae.cornell.edu/psiaki_etal_ieee_taes_submission_dec2011.pdf
  - *GNSS Spoofing Detection using AGC and other methods* (examples & further reading): https://radionavlab.ae.utexas.edu/images/stories/files/papers/Psiaki_WROD_2014.pdf



## 10. Quick reference snippets



**Simple GRC pipeline (conceptual):**

`RTL-SDR source -> Band-pass filter -> Automatic gain control -> FM/PM/QPSK demod -> Symbol sync -> FEC/CRC -> Packet parser/decoder`

**Doppler correction pattern (conceptual):**

`TLE -> compute Doppler vs time -> feed into frequency-translating block or PLL -> demod in corrected baseband`

---



