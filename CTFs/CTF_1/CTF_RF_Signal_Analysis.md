![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# RF Signal Analysis CTF

**Assets provided:**
- `pass_ctf.iq` - complex float32 baseband IQ recording
- `pass_ctf.json` - metadata for the IQ recording

## Download the files [here](./ctf_1_files.zip) and **unzip** them

**Tools you will need:** GNU Radio Companion, Python 3 (numpy, hashlib, struct, binascii)

**Scenario:** Your team has intercepted a downlink pass from an unknown satellite.
You have the raw IQ recording and nothing else. Decode it, extract what is inside,
and derive the uplink authentication token.

---

**Q1. What is the sample rate of the `pass_ctf.iq` recording?**

---

**Q2. What sync word marks the start of each frame in the bitstream?
Give your answer in hexadecimal.**

---

**Q3. What modulation scheme was used to transmit this signal?**

---

**Q4. What is the symbol rate of this downlink in bits per second?**

---

**Q5. What is the FSK frequency deviation in Hz?**

---

**Q6. How many IQ samples represent exactly one transmitted symbol?
Show your working.**

---

**Q7. What is the satellite name embedded in the telemetry payload?**

---

**Q8. What Unix epoch timestamp is embedded in the telemetry payload?**

---

**Q9. Using the satellite name and epoch you extracted, compute the uplink
authentication token. What is it?**

---

**Q10. What integrity algorithm protects each frame, and which bytes does it cover?
Be specific about what is included and what is excluded.**


## If stuck or want to check your answers, here is the [writeup](./CTF_RF_Signal_Analysis_Writeup.md)


***                                                                 
<b><i>Continuing the CTF? </br>[Next Lab](../CTF_2/)</i></b>

<b><i>Looking for a different CTF/Lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the CTFs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)
