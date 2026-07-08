![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)


# The Slow Exfil CTF

**Capture file:** `slow_exfil.pcap`

**Tools you will need:** Wireshark

## Download the files [here](./slow_exfil.pcap)

**Scenario:** A security analyst reviewing HTTP traffic logs flagged that one internal
host is making unusually regular outbound GET requests to the same external IP. You have
been handed a packet capture. Determine what is being exfiltrated, how, and from where.

---

**Q1. What is the IP address of the host performing the exfiltration?**

---

**Q2. What destination IP address receives the exfiltrated data?**

---

**Q3. How many HTTP requests does the attacker send as part of the exfiltration?**

---

**Q4. Which HTTP header field carries the hidden data?**

---

**Q5. What encoding scheme is used to embed data in that header field?**

---

**Q6. What substring pattern in the User-Agent header identifies an exfil request
and distinguishes it from normal traffic?**

---

**Q7. What is the 5th character (1-indexed) of the exfiltrated message?**

---

**Q8. What is the full exfiltrated message?**

---

**Q9. How many seconds elapse between the first and last exfil request?**

---

**Q10. How many requests does the decoy host send using the same User-Agent pattern?**


## If stuck or want to check your answers, here is the [writeup](./CTF_Slow_Exfil_Writeup.md)

***                                                                 
<b><i>Continuing the CTF? </br>[Next Lab](../CTF_6/)</i></b>

<b><i>Looking for a different CTF/Lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the CTFs?***
