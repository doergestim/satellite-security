# Data Management & Reproducibility



## Capture formats: SigMF, WAV IQ, raw binary

- **Raw binary //interleaved IQ**
  - Description: simplest form — raw interleaved I/Q samples in binary (I0,Q0,I1,Q1,...). No header or metadata inside the file.
  - Pros: smallest overhead, widely supported by low-level tools.
  - Cons: metadata must be tracked externally; endianness/sample-type mistakes are common when sharing raw files.

- **WAV IQ**
  - Description: leverages the WAV container to store IQ as two audio channels (left = I, right = Q) with a known sample format (ex: 16-bit PCM). Works well when tooling can read WAV.
  - Pros: easy playback with audio tools; some simple tools accept WAV out-of-the-box.
  - Cons: channel/format assumptions must be explicit; WAV headers are limited for RF-specific metadata.

- **SigMF (Signal Metadata Format)**
  - Description: an open metadata standard for signal captures. A SigMF capture pairs a raw IQ file with a JSON sidecar describing center frequency, sample rate, start time, sample format, annotations, and more.
  - Pros: preserves essential metadata in a machine-readable way, supports annotations and multiple segments, and is excellent for reproducibility and sharing.
  - Cons: relies on external sidecar management — both files must be kept together; tooling support is growing but not universal.

- **Compressed and containerized formats**
  - Lossless compression (gzip, zstd) is common to reduce storage costs. When compressing, always record pre- and post-compression hashes and preserve the original file mode/timestamps for provenance.

>[!TIP]
>
> adopt SigMF as your canonical capture format for all lab and field work — it enforces metadata discipline and scales to sharing and automation.

---

## Metadata tagging 

What to record with every capture:

- Capture ID (unique UUID)
- Station ID and operator identity
- Antenna description and orientation (model, gain, polarization)
- SDR device & driver + firmware version
- Center frequency and capture bandwidth
- Sample rate and sample format (int8/int16/float32, endianness)
- UTC start timestamp and duration (with explicit time scale — UTC/TAI/GPS)
- Doppler compensation applied (if any) and offsets used
- TLE/ephemeris used for the pass (file ID + epoch) and ephemeris source
- Antenna rotator state (az/el) logs synchronized to capture time
- Demodulation parameters used for any demodulated bitstreams (symbol rate, filter settings, AGC states)
- Hashes of raw capture files (SHA-256) and signature of the manifest file (if available)
- Toolchain versions and flowgraph identifiers (ex: GNU Radio flowgraph hash)

Why metadata matters: without accurate context, reproducing a decoding attempt may fail or produce different results; metadata is the difference between evidence and noise.

>[!IMPORTANT]
>
>include the time source and time synchronization status in the metadata (ex: GPS-disciplined yes/no, NTP stratum) to expose potential time-spoofing or drift attacks.

---

## Replay & Simulation workflows

Replaying captures and running simulations are critical for debugging, reproducibility, testing anti-spoofing algorithms, and training.

Typical replay workflow

1. Validate the raw capture integrity (compute and compare SHA-256).
2. Load associated SigMF metadata or sidecar to configure demodulators and offsets.
3. Optionally apply pre-processing steps (DC removal, resampling, filtering) in a containerized flowgraph environment matching the recorded processing versions.
4. Record results (decoded frames, logs, timestamps) as new artifacts with a provenance link to the original capture.

>[!TIP]
>
>when performing replays for testing, sign and timestamp the test manifests and maintain a separate test-only repository of keys and certificates to avoid accidental cross-use in operational systems.

---

## Forensics

Comparing captures from multiple ground stations is a powerful technique for validating observations, detecting spoofing, local interference, or equipment faults.

Core steps for multi-station comparison

- Ensure **time alignment**: convert all timestamps to the same time scale and correct for known offsets; if possible use per-file GPS-derived timestamps.
- Normalize sample rates or resample to a common rate before cross-correlation where applicable.
- Compute cross-correlation of baseband signals or demodulated preambles to estimate relative time-of-arrival (TOA) and validate simultaneity. Use phase-coherent comparisons if stations share a frequency/time reference.
- Compare Doppler profiles estimated independently at each station to a trusted ephemeris prediction: mismatches indicate spoofing or ephemeris issues.
- Compare angle-of-arrival (AOA) or amplitude ratios across stations to triangulate source location — this can separate local interference from true satellite-origin signals.

Quality metrics to use

- CRC pass/fail rates, corrected-bit counts, SNR time-series, constellation-error metrics, and timing residuals against predicted ephemeris-based Doppler curves.

>[!IMPORTANT]
>
>require at least two independent stations and matching time/Doppler signatures before elevating an event from anomaly to incident. Single-station claims are fragile.

---

## Chain-of-custody in satellite forensics

A defensible chain-of-custody documents exactly how evidence moved and who handled it, ensuring admissibility and reliability for investigations.

Chain-of-custody recommended procedure

- At capture: assign a unique ID, compute and record cryptographic hashes (SHA-256) of all raw artifacts, and write an append-only log entry recording the capture event and operator.
- During transfer: use encrypted authenticated channels (TLS with mutual auth, signed transfer manifests) and re-hash files at the destination to confirm integrity.
- During analysis: always work on copies of raw evidence. Record all processing steps, tools, and parameters in an immutable provenance manifest. If derived files are created (filtered IQ, demodulated bitstreams), link them to the original artifact and record their hashes.
- Storage & archival: store originals in a write-once or access-controlled archive and keep off-site backups. Record retention policies, user access logs, and periodic integrity re-validation.

Tools and practices

- Use hardware-backed keys (HSM) to sign manifests where legal chain-of-custody is required.
- Maintain an audit trail with timestamps, operator IDs, and justification for each action.

---

## Automation: batch decoding, pipelines, containers

Automation turns repeatable processes into scalable, auditable workflows — but automation must preserve provenance and security.

Design principles for reproducible pipelines

- Immutable inputs: treat raw captures as immutable artifacts. Pipelines consume artifacts and produce versioned outputs; never modify raw data in place.
- Versioned processing: record exact versions of decoders, flowgraphs, libraries, and container images used for each processing job.
- Deterministic runs: fix random seeds where relevant and record non-deterministic sources (ex: CPU architecture differences) in manifests.
- Monitoring and alerting: pipelines should emit structured logs, metrics (throughput, error rates), and alerts when anomalies exceed expected thresholds.

CI/CD for decoders and analysis

- Create unit tests and integration tests that run sample captures through decoders and compare outputs to canonical expected results.
- Use container images to ensure consistent runtime environments; pin base images and use reproducible build systems.

Workflow orchestration

- Use workflow engines (Airflow, Prefect, or simple cron-based schedulers) to manage batch decoding, metadata enrichment, and archival steps. Each workflow run should produce a run-manifest linking inputs, outputs, and environment specs.


---

### Glossary // quick terms

- SigMF: Signal Metadata Format (IQ sidecar JSON standard).
- AOS/LOS: Acquisition and Loss of Signal (from earlier chapters).
- TOA: Time-of-Arrival.
- HSM: Hardware Security Module.
- CI: Continuous Integration / CD: Continuous Delivery.

