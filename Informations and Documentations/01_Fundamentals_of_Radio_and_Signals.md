# Fundamentals of Radio & Signals



## What is a signal? (Analog vs Digital)

A **signal** is any measurable quantity that conveys information. In RF systems, signals are typically electromagnetic waves whose amplitude, phase, or frequency vary in time.

- **Analog signals:** continuous in time and amplitude. Examples: AM radio carrier, analog telemetry modulated on a subcarrier. Analog signals are modeled as continuous functions x(t).
- **Digital signals:** discrete in time and/or amplitude. Encoded as symbols or bits (0/1). The transmitter maps digital data to a waveform using a modulation scheme (ex: PSK, FSK, QAM).

**Why the distinction matters for satellite security:** analog signals are often easier to intercept but may carry less structured data; digital signals can hide complex frame structures, checksums, and cryptographic payloads. Attack and defense techniques differ — ex: jamming a wideband analog channel vs. selective packet injection against a digital protocol.

---

## Frequency, Wavelength, Bandwidth

**Frequency (f\)** measures cycles per second (Hz). For electromagnetic waves, frequency and wavelength  are related by the speed of light \(c\):



**Bandwidth (BW)** is the range of frequencies a signal occupies. For a carrier-modulated signal, approximate bandwidth depends on modulation: ex: FM deviation and modulation rate, or symbol rate for digital modulations. 

>[!TIP]
>- Satellite links are often bandwidth-limited — spectrum planning and filters are crucial.
>- Narrowband beacons (ex: CW or low-bit-rate telemetry) are easy to monitor at low cost, while wideband payloads (DVB-S2, high-rate telemetry) require higher sampling rates and bandwidth-capable hardware.


---

## Modulation Basics: AM, FM, PM, and digital modulations

### Analog modulation
- **AM (Amplitude Modulation):** message modulates the carrier amplitude. Spectrum contains carrier and sidebands. Susceptible to amplitude noise and fading.
- **FM (Frequency Modulation):** message modulates instantaneous frequency. More robust to amplitude noise; requires larger bandwidth (Carson’s rule).
- **PM (Phase Modulation):** message modulates carrier phase. // related to FM

### Digital modulation 
- **FSK (Frequency Shift Keying):** symbols encoded as different frequencies. Easy to implement and robust in low-SNR.
- **PSK (Phase Shift Keying):** bits encoded as phase changes (BPSK, QPSK, 8PSK). Common in satellite comms (BPSK for telemetry, QPSK/QAM for higher rates).
- **QAM (Quadrature Amplitude Modulation):** combined amplitude+phase constellation (ex: 16-QAM, 64-QAM) — higher spectral efficiency but sensitive to SNR.
- **GMSK (Gaussian Minimum Shift Keying):** continuous-phase FSK variant with spectral efficiency (used in some telemetry links).
- **OFDM (Orthogonal Frequency Division Multiplexing):** multicarrier scheme with many closely spaced subcarriers — used in broadband links but uncommon for smallsat downlinks due to PAPR and complexity.

**considerations for satellite security:**
- Higher-order modulations (ex: 16-QAM) offer more capacity but are fragile under interference; an attacker can degrade performance with modest jamming power.
- Certain legacy satellite telemetry uses simple modulations (AX.25 AFSK, OOK) that are easier to decode and spoof.

---

## Noise, SNR, and Interference


**Signal-to-Noise Ratio (SNR)** is the ratio of signal power to noise power. For digital comms, the relevant metric often is energy per bit to noise density — direct determinant of BER for a given modulation and coding.

**Interference types:**
- **Co-channel interference:** unwanted transmissions in the same channel.
- **Adjacent-channel interference:** leakage from nearby channels due to imperfect filtering.  
- **Out-of-band emissions & harmonics.**

**Jamming models:**
- **Broadband noise jammer** raises noise floor across large bandwidth.  
- **Narrowband or tone jammers** target a specific frequency.  
- **Protocol-aware jammers** transmit on protocol-specific frames or exploit MAC behaviors.


>[!TIP]
>
>**detection tips:**
>- Monitor SNR / RSSI baselines and trigger on sudden jumps in noise floor or unexplained step-changes.  
>- Use spectral occupancy scans (waterfall plots) to spot intermittent interferers.  
>- Correlate across stations to determine if local equipment or environmental interference.


---

## IQ Sampling & Complex Signals (Complex Baseband)


The complex baseband u(t) = I(t) + jQ(t) contains the in-phase (I) and quadrature (Q) components. Sampling the complex baseband at rate f(s) captures amplitude and phase information and is how SDRs represent RF signals in software.

**Why complex sampling?**
- Complex sampling avoids image-frequency aliasing and allows efficient digital downconversion and filtering.  
- A single complex stream represents both amplitude and phase required for modern modulation schemes (PSK/QAM).

**SDR notes: **
- SDR hardware often performs analog downconversion then ADC; software expects complex I/Q samples at a chosen sample rate and format.  
- For reception, choose a sampling rate at least twice the signal bandwidth (Nyquist) after translation to baseband.


---

## Filters (Low-pass, High-pass, Band-pass) 

### Filter types & purpose
- **Low-pass (LP):** passes low frequencies, rejects high — used for anti-aliasing and baseband extraction.  
- **High-pass (HP):** passes high frequencies — used for DC-blocking or AC-coupling.  
- **Band-pass (BP):** passes a band of frequencies — used for channelization and front-end selectivity.

### FIR vs IIR
- **FIR (Finite Impulse Response):** always stable, can have exact linear phase, easier to design for DSP. FIR length relates to filter steepness and stopband attenuation.  
- **IIR (Infinite Impulse Response):** efficient (fewer coefficients) for given sharpness, but non-linear phase and potential stability concerns.

### Windowing & filter
- Common FIR windows: Hamming, Hanning, Blackman, Kaiser. Window choice trades main-lobe width vs side-lobe attenuation.  
- Use Parks-McClellan (Remez) for equiripple FIR designs when optimal stopband performance is needed.

### Polyphase & multirate filtering
- **Polyphase filters** implement efficient resampling (decimation/interpolation) and channelizers (filter banks) — critical for SDR pipelines that need computational efficiency.

### Considerations for satellite signals
- Use steep bandpass filters to isolate a narrow telemetry carrier in a crowded VHF/UHF band.  
- For GNSS and L-band, filter insertion loss and front-end linearity matter — avoid strong blockers saturating the LNA.  
- Apply matched filters in detection chains for known preambles/pulse shapes to maximize SNR.

---

## Putting it together — a SDR receive chain

A typical SDR reception pipeline for a satellite downlink:

1. Antenna → LNA → bandpass filter //analog front-end
2. Mixer / RF front-end → ADC //downconversion to IF or complex baseband
3. Digital anti-aliasing / decimation /FIR polyphase
4. AGC / normalization
5. Channel filtering / demodulator //matched filter, PLL, symbol sync
6. Framing / FEC decoding / telemetry extraction

**Security-specific notes:**
- Monitor upstream steps (RF spectrum & AGC) for signs of jamming or spoofing (unexpected amplitude/power changes, sudden frequency offsets).  
- Preserve raw IQ dumps (SigMF metadata) for later forensic analysis — avoid lossy processing before archiving evidence.


---

## resources & further reading 

- **GNU Radio docs & tutorials** — https://wiki.gnuradio.org/  
 
- **DSP Guide** by Steven W. Smith — free online; great practical DSP primer: https://www.dspguide.com/  

- **Practical SDR tutorials (RTL-SDR.com)** — https://www.rtl-sdr.com  
---
