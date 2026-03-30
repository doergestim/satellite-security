# Telemetry & Data Handling




## 1. Overview // what telemetry is and why it matters?

**Telemetry** is the automated transmission of measurements and state information from a remote platform (the satellite) to a receiver (ground station). Telemetry provides the operational picture (housekeeping), science measurements (payload data), and status information needed to operate, diagnose, and protect spacecraft.

From a security standpoint telemetry is:
- **Evidence** (logs, signals, state transitions) that can be used to detect anomalies, intrusions, or misuse.
- **A reconnaissance target** for adversaries. //unprotected telemetry may reveal vulnerabilities or operational details
- **A control feedback source** — correctly understanding and protecting telemetry is essential to prevent false positives/negatives in failure detection and to avoid spoofing.

---

## 2. Telemetry frame structure & metadata 

A telemetry frame is the encapsulation used to carry one or more measurements and control/status fields. Different missions use different frame formats (AX.25, CCSDS Space Packet, custom formats), but common conceptual components exist across formats.

### Typical frame components 
- **Preamble / sync pattern:** enables bit synchronisation and framing at the receiver. It may be a fixed bit pattern or a symbol sequence.
- **Header (frame-level):** contains metadata about the frame such as: version/type, spacecraft ID, packet/stream ID, sequence number, timestamp (or time tag), priority/urgency flags, and length fields.
- **Subsystem/virtual channel header:** for multiplexed streams, a second-level header indicates which onboard subsystem or virtual channel the payload belongs to.
- **Payload:** the application data. Can bundle multiple telemetry words, sensor packets, or compressed binary blobs.
- **Trailer / FCS (CRC):** checksum/CRC or larger error-detection codes to validate transmission integrity.
- **Optional FEC parity / ECC fields:** forward error correction appended to help correct bit errors without retransmission.

### Metadata you must preserve when capturing frames
When you save a telemetry capture, also save the following context metadata — without it, decoding later is hazardous:
- **Center frequency and capture bandwidth** used by the SDR.
- **Sample rate and sample format** (float32, int16, interleaved IQ, channel order).
- **UTC timestamp of capture start** and ideally a synchronized high-resolution timestamp per sample/block.
- **Demodulation parameters used (modulation, symbol rate, sample offset, filters)** if you recorded processed audio or demodulated bit streams.
- **Receiver gain/attenuation and antenna used** (gains and front-end status).
- **Any Doppler compensation applied** or frequency offsets used.

>[!IMPORTANT]
>
>Without these, reproducing or verifying a capture's contents becomes error-prone.

---

## 3. Housekeeping data (HK) // content, semantics, and handling

**Housekeeping (HK)** telemetry carries spacecraft health and status: power buses, battery voltage and current, solar-panel currents, battery state-of-charge estimates, temperatures, attitude and reaction-wheel speeds, fault flags, reset counters, and software/firmware version identifiers.

### Characteristics of HK data
- **Low data rate, frequent reporting:** many HK channels are sampled at modest rates but sent periodically to monitor trends.
- **High operational importance:** operators rely on HK for safe operations; missing or corrupted HK can force conservative safe modes.
- **Usually numerical, calibrated, and scaled:** raw ADC counts are converted to engineering units via calibration constants (scale, offset) — preserving calibration metadata is mandatory to interpret values correctly.

### HK semantics & classification
- **Critical vs informative:** separate channels into critical (fast alerts, safety) and informative (science housekeeping, long-term trends). Critical HK often has higher reporting priority and additional protections.
- **State flags and bitfields:** many status values are encoded as bitfields indicating mode, error conditions, or subsystem states. 

### Best practices in handling HK
- **Time-tag every HK packet with a reliable time source.** Prefer mission time (spacecraft epoch) plus ground cross-reference.
- **Preserve raw frames and also store decoded values with conversion metadata. //cal constants, units**
- **Track provenance for calibration tables** — change of calibration must be logged and versioned to avoid misinterpretation of historical telemetry.

---

## 4. Science payload data //variety, volume, and structure

Science payloads vary wildly: imagery, spectrometry, radio science, magnetometry, or custom experimental sensors. Payloads differ from HK in volume, structure, and processing needs.

### Common payload traits
- **High data volume:** imagery and raw science streams can generate large data that require compression, segmentation, or on-board buffering and prioritized downlink scheduling.
- **Application-level framing:** payload data often carries its own internal framing, meta-headers describing mode, exposure, instrument parameters, timestamps, and checksums.
- **Event-driven bursts:** some instruments produce bursty traffic (ex: when an instrument is triggered), requiring flexible telemetry multiplexing.

### Payload processing pipeline
- **Onboard preprocessing:** compression, packetization, FEC application, prioritization.
- **Downlink transport:** multiplexed into spacecraft TM frames according to priority and link capacity.
- **Ground processing:** deframing, decompression, instrument-specific calibration and geo-registration, archiving, and distribution to science teams.

### Data quality & calibration
- **Instrument calibration** (radiometric, geometric, spectral) is essential. Calibration coefficients and processing recipes must be preserved as part of data provenance.
- **Metadata richness** (exposure times, instrument modes, onboard processing steps) determines the science usability of archived data.

---

## 5. Parsing & deframing techniques

Successful decoding of telemetry requires reliable bit- and byte-level synchronization and robust handling of imperfect links.


### Preprocessing: synchronization and symbol alignment
- **Bit synchronization** (or symbol synchronization) recovers the correct sampling instants for symbol decisions; this is typically handled in the demodulation stage but is essential input for deframing.
- **Byte alignment**: once a bitstream is recovered, you must find the correct byte boundaries — preambles or sync words are used for this.

### Framing detection
- **Sync word detection:** scan for the preamble/sync pattern, allow for error tolerance (ex: correlation-based detection with thresholding), then align bits/bytes to the expected frame boundary.
- **Handling bit slip:** imperfect clocks and channel noise may cause bit slippage; robust receivers monitor sync loss and re-acquire using sliding correlation windows.

### Escape sequences & byte/bit-stuffing
- **Frame delimiters vs payload contents:** many protocols use a reserved delimiter (ex: 0x7E in HDLC/AX.25) which requires escaping or bit-stuffing in payloads to avoid accidental delimiter occurrence. A deframer must implement un-stuffing logic.
- **Error-resilient parsing:** do not assume perfect frames — detect malformed frames (bad length fields, invalid CRC) and gracefully resynchronize rather than discarding large data chunks.

### Checksum and CRC validation
- **CRC check as gatekeeper:** compute CRC/FCS over the received frame and compare to trailer — reject frames with mismatched CRC, but log them for forensic analysis (errors themselves can be diagnostic).
- **FEC decoding:** if FEC parity was included, perform error correction prior to CRC checking; track corrected-bit counts as a quality metric.

### Multiplexing & virtual channels
- **Demultiplexing:** many satellites multiplex multiple logical channels into a single physical frame; extract channel IDs from headers and route payloads to appropriate handlers.
- **Ordering & reassembly:** frames may carry sequence numbers and fragmentation flags — deframing must reassemble segmented payloads reliably.

### Dealing with partial captures & noise
- **Partial frames:** when captures start mid-frame, look for the next valid sync and resync gracefully.
- **Noise spikes & false syncs:** employ conservative correlation thresholds and secondary validation steps (lengths, CRC) before accepting a sync lock.

---

## 6. KISS protocol// telemetry over serial/TCP 

**KISS (Keep It Simple, Stupid)** is a lightweight framing convention used by TNCs and some ground station software to wrap packet-radio frames (often AX.25) over serial or TCP links. KISS is simple and widely used for passing raw frames between a radio modem and a host application.

### KISS basics 
- **Frame delimiter:** KISS uses the special byte `FEND` (0xC0) to mark frame boundaries.
- **Escaping:** `FESC` (0xDB) and escape sequences (ex: `FESC` `TFEND`) are used to encode literal control bytes that appear in payloads.
- **Command byte:** a single command/control byte follows FEND to indicate port or command type; after that comes the payload and a closing FEND.

### Why KISS and telemetry pipelines?
- KISS allows a TNC or radio interface to present raw layer-2 frames (ex: AX.25 packets) to host software reliably over a byte-stream transport.
- When capturing telemetry over TCP/serial using KISS, store the raw KISS-wrapped frames (or the unwrapped payload) and record timestamps at the earliest point possible (on receive) to preserve timing provenance.

### Pitfalls
- **Byte-order & escaping errors:** improper un-escaping corrupts frames — use robust KISS parsers that tolerate partial frames.
- **Timing ambiguity:** serial streams may buffer — ensure timestamping occurs at read time to avoid ambiguous event ordering.

---

## 7. Data formats, storage & archival 

Telemetry and payload data must be stored in a way that preserves raw evidence, supports reproducible analysis, and enables efficient retrieval.

### Two-tier storage approach
1. **Raw evidence store**: preserve raw IQ captures, demodulated bitstreams, and original frames *unchanged*. These are the forensic source-of-truth.
2. **Processed & indexed store**: decoded telemetry, calibrated values, metadata, and indices for quick search and analytics.

### File formats & metadata
- **Use open, documented formats** where possible (ex: SigMF for IQ captures, CCSDS or mission-defined binary for frames). Always include a metadata sidecar (JSON) that records capture parameters and provenance.
- **Store checksums & hashes** (SHA-256) for every file to detect corruption and enable chain-of-custody proofs.
- **Version control for decoder rules & calibration**: decoders, lookup tables, and calibration files should be versioned (ex: git or artifact repository) so that processed results can be reproduced precisely.

### Retention & archival
- **Retention policy:** raw IQ and raw frames can be large — define retention tiers (keep critical event captures long-term, rotate routine captures to long-term compressed archives or summaries).
- **Off-site backups & immutable archives:** for chain-of-custody, use write-once or WORM-capable archives and maintain off-site replication.

---

## 8. Data provenance & chain-of-custody

Maintaining a trustworthy chain-of-custody for telemetry is essential when using captures for security incident investigations. The goal is to demonstrate that the data you analyze are unaltered and were handled according to procedures.

### Provenance elements to capture
- **Who** performed the capture (operator, system process).
- **When** the capture occurred (UTC timestamp, time source) and where it was stored first.
- **What** equipment and settings were used (SDR model, antenna, gains, filters, firmware versions, sample rates, and any applied Doppler offsets).
- **How** the data were transferred, processed, and who had access.
- **Hashes & digital signatures** of raw files at the point of capture.

### Chain-of-custody workflow 
1. **Capture & immediate hashing:** compute and store cryptographic hashes (SHA-256) of raw captures immediately after acquisition.
2. **Record metadata & secure logging:** write an append-only audit record (timestamped) documenting capture parameters and operator actions.
3. **Controlled transfer:** when moving data (network transfer, USB), use authenticated channels (TLS, signed transfer manifests) and re-hash upon receipt to prove integrity.
4. **Access controls & audit trails:** restrict who can access raw evidence and record every access with purpose and timestamp.
5. **Preserve originals:** do analysis on copies; never modify raw evidence files. If modifications are needed (ex: filtering), store the modified file as derived data with links to the original and the exact processing steps.
6. **Chain-of-custody ledger:** maintain a human- and machine-readable ledger that records each custody step for each evidence item.

### Cryptographic sealing & notarization
- **Digital signatures / HSMs:** signing captures with a long-lived key managed by an HSM (Hardware Security Module) provides non-repudiation.
- **Time-stamping authorities (TSA):** combining hashes with a trusted timestamp authority helps defend against claims that timestamps were altered.

---

## 9. Security of telemetry links // authentication, integrity, confidentiality

Many legacy telemetry links are unauthenticated and unencrypted. For security-critical systems, consider layered protections.

### Authentication & integrity
- **Message authentication codes (MACs):** append cryptographic MACs (HMAC, CMAC, or modern AEAD tags) to authenticated messages. Include key-management processes.
- **Digital signatures:** for non-repudiation and auditability, sign telemetry samples or critical state reports (requires onboard crypto capability and key provisioning).
- **Sequence numbers & timestamps:** prevent replay by including monotonic counters and time tags verified against trusted clocks.

### Confidentiality
- **Encryption of sensitive payloads:** apply authenticated encryption (ex: AES-GCM) when telemetry reveals sensitive operational state.
- **Key distribution challenges:** satellites are constrained environments — ground-to-space key provisioning, key rotation, and secure onboard key storage must be planned in advance.

### Practical constraints
- **Computational limits:** many small satellites have constrained CPUs and power; crypto choices must match platform capabilities.
- **Bandwidth limits:** adding authentication and encryption increases overhead; balance protection needs with link capacity.

>[!TIP]
>
>treat telemetry protection as a system design requirement, not an afterthought. Plan for secure boot, secure telemetry channels, and authenticated command channels.

---

## 10. Anomaly detection & data quality checks 

Detecting anomalies requires a combination of simple rule checks and statistical/time-series methods.

### Simple sanity checks
- **Range checks:** voltages and temperatures within expected min/max ranges.
- **Delta checks:** sudden large jumps in values (ex: instantaneous battery voltage drop) trigger alerts.
- **CRC/sequence checks:** repeated CRC failures, out-of-order or missing sequence numbers.

### Statistical & model-based techniques
- **Moving-average baselines and trend detection:** detect slow drifts (ex: battery degradation) using trend estimators.
- **Model-based residuals:** create a predictive model (physics-based or learned) and monitor residuals; significant residuals indicate anomalies.
- **Correlation across channels and stations:** cross-validate telemetry readings with redundant sensors or multiple ground observations.

### Forensic signals in telemetry errors
- **Patterned bit-errors:** repeating error patterns can indicate intentional jamming or narrowband interference.
- **Sequence of subtle anomalies** across subsystems may indicate an attack (ex: coordinated resets followed by anomalous attitudes).

---

## 11. Best Practices & Governance

- **Define telemetry classification:** not all telemetry needs the same confidentiality; classify channels and protect accordingly.
- **Document everything:** frame formats, calibration tables, decoder versions must be documented and preserved.
- **Automate reproducible pipelines:** use versioned processing pipelines, containerized software, and recorded configurations to make analyses reproducible.
- **Incident readiness:** maintain playbooks for suspected telemetry compromise, including containment, evidence preservation, and notification procedures.



---

### Glossary//quick terms

- **HK:** Housekeeping telemetry.
- **FCS / CRC:** Frame Check Sequence / Cyclic Redundancy Check.
- **KISS:** Keep It Simple, Stupid (serial framing for passing raw frames).
- **SigMF:** Signal Metadata Format (metadata sidecar for IQ captures).
- **MAC / AEAD:** Message Authentication Code / Authenticated Encryption with Associated Data.



