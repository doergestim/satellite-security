![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Replay Attack Detection CTF

**Assets provided:**
- `groundstation_ingest.pcap` - capture of traffic to a groundstation's `/ingest` telemetry endpoint

**Tools you will need:** Wireshark (or tshark)

## Download the file [here](./groundstation_ingest.pcap)

**Scenario:** You are reviewing a packet capture pulled from a groundstation's network
tap after operators reported the live telemetry dashboard appeared to "freeze" and loop
the same readings. Determine what happened.

---

**Q1. What HTTP endpoint receives the suspicious traffic in this capture?**

---

**Q2. What HTTP method is used for every request to that endpoint?**

---

**Q3. How many legitimate telemetry posts appear before the suspicious traffic begins?
A post is "legitimate" if its JSON body has `"mode":"NOMINAL"`.**

---

**Q4. How many requests make up the replay burst? A replay request has
`"mode":"SAFE"` in its JSON body.**

---

**Q5. What is the `epoch` value inside the JSON body of the replayed requests?
Is it the same for every replayed request, or does it change?**

---

**Q6. What was the `epoch` value of the last legitimate post before the replay burst
began?**

---

**Q7. How many seconds elapse between the last legitimate post and the first replayed
request?**

---

**Q8. What HTTP response code does the server return to every replayed request?**

---

**Q9. How many distinct TCP source ports does the attacker use across the entire
replay burst?**

---

**Q10. What check is missing from the server's handling of `/ingest` that would have
prevented this attack? Name the specific defensive mechanism.**


## If stuck or want to check your answers, here is the [writeup](./CTF_Replay_Detection_Writeup.md)


***                                                                 
<b><i>Continuing the CTF? </br>[Next Lab](../CTF_3/CTF_TLE_Orbital_Forensics.md)</i></b>

<b><i>Looking for a different CTF/Lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the CTFs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)
