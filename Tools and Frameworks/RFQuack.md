# RFQuack

> **Tool:** RFQuack
>
> **Link:** https://rfquack.org
>
> RFQuack is a modular firmware + software toolkit and dongle ecosystem for sniffing, analyzing, manipulating, and transmitting radio-frequency signals using small MCU + radio modules (CCxxxx-family, RFM69, nRF24, etc.). It provides a Python interactive shell, pluggable modules (scanners, decoders, replay, fuzzers), and a CLI/Docker client for integration. RFQuack is designed for RF reverse-engineering, rapid prototyping of RF dongles, and distributed crawling (NetQuack).

---

## Quickstart

```bash
git clone --recursive https://github.com/rfquack/RFQuack
cd RFQuack
pip install -r requirements.pip
vim build.env  # set your parameters and :wq
make clean build flash
```

---

## What RFQuack gives you?

- **Multi-radio dongle platform:** support for several radio chips on the same board (up to 5 radios depending on hardware) so you can monitor multiple bands/protocols concurrently.
- **Automatic frequency & bitrate estimation:** built-in modules that can lock onto unknown carriers and estimate symbol rates, enabling quick capture of unknown signals in the wild.
- **Interactive Python shell & scripting:** `q` shell with tab-completion to configure radios, start sniffers, set packet patterns, and script workflows — excellent for rapid investigation or automation.
- **Sniff / record / replay / fuzz:** modules for passive sniffing, storing raw packets, replaying saved transmissions, and performing parameter fuzzing to test receiver robustness.
- **Pluggable modules & API:** modular design makes it easy to add custom decoders, new radio drivers, or attack modules (ex: roll-jam, mouse-jack style attacks) and to integrate with brokered backends (MQTT) or distributed crawlers (NetQuack).
- **Lightweight CLI & Docker support:** RFQuack-cli allows easy distributed operation (Docker examples) and remote control of dongles for bulk capture or automated windows in a testbed.

---

## Why RFQuack and satellite security?

- **Ground-side reconnaissance & interference detection:** use RFQuack dongles to sweep and fingerprint local RF environments around ground stations to detect unexpected transmitters or interference in satellite bands reachable by compact radios (UHF/VHF/ISM). This helps reveal nearby jammers, unauthorized beacons, or equipment misconfiguration.
- **Peripheral device testing:** many ground-support devices (remote sensors, telemetry beacons, tracking beacons, antenna controllers) use low-power RF; RFQuack helps reverse engineer and test these peripheral links for vulnerabilities (replay, injection, malformed commands) in a controlled lab.
- **Crawling & baselining with NetQuack:** deploy multiple RFQuack nodes to create a distributed crawler that builds a historical baseline of local spectral activity — useful for long-term anomaly detection near critical infrastructure.
- **Rapid prototyping of test dongles:** build custom dongles that emulate specific modems or beacons to validate ground radio stacks, evaluate filtering, or test anti-spoofing heuristics in a shielded environment.
- **Integration into incident response:** when a suspected interference or spoofing incident occurs, a portable RFQuack dongle can quickly gather packets, capture waveforms, and provide immediate intelligence for triage before larger SDR arrays are mobilized.

---

## use cases & workflows 

- **Spectrum crawl & fingerprinting:** run automatic frequency/bitrate estimation modules across bands of interest during satellite passes and store captures with timestamps and GPS/TLE metadata. Use results to build a fingerprint DB for normal vs anomalous transmissions.
- **Incident triage:** on detection of unusual packet types or SNR drops at a ground station, field-operatives can deploy RFQuack units to capture local emissions, attempt local replay to validate whether interference is in-band or local, and forward captures to analysts.
- **Ground equipment hardening checks:** emulate known control packets and malformed frames to test whether local antenna controllers, rotator units, or telemetry gateways validate inputs robustly // in shielded lab 
- **Distributed detection (NetQuack):** create a small fleet of RFQuack crawlers to scan and index signals around a facility; correlate multiple nodes to locate persistent interferers or suspicious transmitters using comparative power/time analysis.

---

## How to use RFQuack practically

1. **Pick hardware:** choose a supported board (Teensy/ESP32 variants with selected radio modules, or purpose-built RFQuack dev boards) and populate required radio modules //CC1101, RFM69, nRF24, etc.
2. **Build & flash firmware:** follow RFQuack’s build instructions (PlatformIO-based); prebuilt images and Docker CLI exist for convenience.
3. **Install RFQuack‑cli or use shell:** connect via serial or MQTT; start the Python `q` shell to explore available modules.
4. **Run automatic scan:** use frequency_scanner or bitrate_estimator modules to detect carriers automatically and store raw packets.
5. **Decode / analyze:** export captured packets and load into analysis tools (Wireshark, GNU Radio, custom Python parsers). Use the RFQuack shell to replay or fuzz parameters in a shielded testbed.
6. **Automate & integrate:** use Docker CLI or MQTT to integrate captures into your ingestion pipeline, or set up NetQuack for distributed aggregation.

---

## Integration with your existing stack

- **GNU Radio / gr-satellites / SatNOGS:** use RFQuack for quick field captures and low-power crawling; move richer IQ captures into GNU Radio for deep demodulation and satellite protocol decoding.
- **SIEM & analytics:** forward JSON/PCAP-style packets via MQTT or push to a central collector for correlation with TLEs, SatNOGS observations, and IDS events.
- **NetQuack / distributed collection:** pair RFQuack nodes with a central indexer to build a searchable dataset of local RF activity for long-term trend analysis.

---

## Operational considerations & legal/ethical guidance

- **Regulatory limits:** transmitting or replaying signals can be illegal and dangerous. Only transmit on authorized bands or inside shielded lab setups (Faraday cages) with explicit permission.
- **Dual-use caution:** RFQuack simplifies attacks like replay, jamming, or injection; document these features carefully and restrict access to authorized personnel only.
- **Supply & hardware footprint:** RFQuack nodes are small and portable — great for field ops but also easy to misplace; secure physical custody and inventory.
- **Data provenance:** tag captures with node ID, firmware version, GPS/timestamp, and operator to maintain chain-of-custody for forensic use.

---

## Threats & abuse scenarios 

- **Local reconnaissance & mapping:** attackers can deploy RFQuack crawlers to map ground-station peripheries and discover weakly-protected control links.
- **Replay/injection against ground devices:** compromised or naively designed ground peripherals can be replayed or injected in the wild if permissive..
- **Data poisoning of distributed crawlers:** attackers may try to flood public indices (if any) or corrupt NetQuack datasets; use authenticated collectors and provenance checks.

---

## Mitigations & recommendations

- **Shielded testing for active features:** always perform transmit/replay/fuzz testing within a Faraday cage or isolated lab.
- **Access control:** restrict who can flash firmware, run replay modules, or access historic captures; use role-based controls where possible.
- **Harden ground peripherals:** require cryptographic authentication for control links; use rolling codes or challenge-response schemes where feasible to avoid trivial replay attacks.
- **Provenance & corroboration:** require cross-node corroboration for alerts derived from RFQuack crawlers; apply sanity checks to detect poisoned data.

---

## Practical resources & links

- RFQuack main site: https://rfquack.org
- RFQuack modules overview: https://rfquack.org/modules/overview/  
- RFQuack GitHub (firmware + tools): https://github.com/rfquack/RFQuack  
- RFQuack-CLI (Docker client): https://github.com/rfquack/RFQuack-cli  
- RFQuack architecture notes: https://rfquack.org/architecture/  

---


