# The Whisper Channel CTF

**Assets provided:**
- `whisper_channel.iq` - complex float32 baseband IQ recording
- `whisper_channel.json` - metadata for the IQ recording

**Tools you will need:** GNU Radio Companion, Python 3 (numpy)

**Scenario:** You have intercepted a baseband recording during a satellite pass.
There is an obvious signal. Decode it. Then look harder.

---

**Q1. Load `whisper_channel.iq` into GNU Radio and open a QT GUI Frequency Sink.
How many distinct signals are visible in the spectrum?**

---

**Q2. What is the approximate carrier frequency offset of the stronger (main)
signal, in Hz?**

---

**Q3. What is the approximate carrier frequency offset of the weaker (hidden)
signal, in Hz?**

---

**Q4. By measuring the two FSK tone positions of the hidden signal, what is its
frequency deviation (fdev) in Hz?**

---

**Q5. What is the symbol rate of the hidden signal in bits per second?
Show your working.**

---

**Q6. Approximately how many dB weaker is the hidden signal compared to the
main signal?**

---

**Q7. Decode the hidden signal. What is the flag contained in its payload?**

---

**Q8. The hidden signal's JSON payload contains a field identifying the channel
type. What is its value?**

---

**Q9. What does the main signal's payload contain?**

---

**Q10. What is the sample rate of the recording?**
