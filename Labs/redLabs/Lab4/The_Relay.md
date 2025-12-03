![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 4 - The Relay

**Scenario:** Show how an attacker can **record** valid satellite command traffic and **replay** it later to a naive groundstation that lacks freshness checks. You’ll plan a pass, review/parse captured frames, then replay them into a local vulnerable groundstation emulator and measure impact. Finally, you’ll turn on defenses and re-test.

**You’ll practice:** pass planning (gpredict), KISS framing basics, safe replay, and blue-team mitigations (timestamps, nonces, dedup)

---

## Setup 

Download the zip for this main folder from [Here](./lab-4-the-relay.zip) ( Only if you are not using the VM )

- Click the Download button

<img width="330" height="177" alt="image" src="https://github.com/user-attachments/assets/df15f9ee-985a-4f6a-af65-32698e1aa337" />


- Extract it

```bash
# enter the lab folder
cd ~/Desktop/Lab4
```

```bash
# Python virtual env for reproducibility
python3 -m venv .venv
```

```bash
# activate it
source .venv/bin/activate
```

```bash
# upgrade pip and install our few dependencies
pip install --upgrade pip
pip install pycrc
```

**Why:** The emulator and helpers are plain Python, `pycrc` is only used in one optional check

Find the documentation for [gpredict here](/Tools%20and%20Frameworks/gpredict.md)

---

## Assets overview

```bash
tree -L 2
```
Expected:
```
assets/
  DEMO-SAT.tle            # synthetic TLE to import into gpredict
  captured_kiss.hex       # two KISS frames (AUTH, EXEC MODE=OP) as hex
emulators/
  groundstation_vuln.py   # naive KISS command sink on TCP 52001 + HTTP /status
tools/
  packet_tools.py         # decode/inspect KISS frames, show parsed fields
  replay.py               # send KISS frames to a host:port
  verify_success.py       # check emulator state via HTTP
```

**Why:** You can run the full lab without any RF gear using `assets/captured_kiss.hex`.

---

## Plan a pass with gpredict

Import the synthetic TLE and look at when **DEMO‑SAT** would be visible from your location.

```bash
# Start gpredict from a terminal for logs (optional)
gpredict
```
In gpredict:
- **Edit -> Update TLE data from local files**: choose `assets/` and *open*

![image](/Assets/RLab4/RLab4-1.png)
<img width="315" height="170" alt="image" src="https://github.com/user-attachments/assets/e39205f1-89bf-4ea8-ad78-a72ada1f3c7b" />

- **File -> New Module**: create a new module (“DEMO-Lab”)

![image](/Assets/RLab4/RLab4-2.png)

- **Satellites -> Search for our DEMO-SAT**: pick `assets/DEMO-SAT.tle`

![image](/Assets/RLab4/RLab4-3.png)

- Observe **AOS/LOS** and nominal UHF center frequency you’d monitor (synthetic example: 437.500 MHz)

![image](/Assets/RLab4/RLab4-4.png)


**Why:** See how real operators plan captures, we’ll still use a synthetic capture next

---

## Inspect the provided capture

Open and parse the KISS frames we provide

This simulates “post‑demod” packets an attacker or operator might export

>[!IMPORTANT]
>Make sure you are at `~/Desktop/Lab4/`

```bash
cd ~/Desktop/Lab4
```

```bash
# Show frames and parsed fields
python3 tools/packet_tools.py --show assets/captured_kiss.hex
```

![image](/Assets/RLab4/RLab4-5.png)


**What you’ll see (explanation):**
- `timestamp`: embedded epoch the sender claimed when the frame was created
- `counter`: a simplistic sequence value
- `data_utf8`: command payload string
- `crc_ok`: integrity over the plaintext payload passes (not a cryptographic MAC)

**Why:** Know exactly what you’re about to replay

---

## Start the vulnerable groundstation emulator

The emulator listens for **KISS** frames on **TCP 52001** and exposes a JSON status at **http://127.0.0.1:8000/status** 
It intentionally lacks freshness and replay protections by default

```bash
# run the emulator
python3 emulators/groundstation_vuln.py
```
Keep it running in its terminal. Open a second terminal (or tab) for the replay steps, activate the venv again:

```bash
# in a new terminal tab, back in the lab folder
cd ~/Desktop/Lab4
source .venv/bin/activate
```

Quick health check:
```bash
# read status
python3 tools/verify_success.py --url http://127.0.0.1:8000/status
```
Expected JSON:
```json
{
  "owned": false,
  "mode": "SAFE",
  "last_seen": null,
  "nonce": null
}
```
**Why:** Confirm the target is reachable and in a known-good state

---

## Replay the capture

Send the two frames (AUTH followed by EXEC) to the emulator. Pace them at ~0.8 s so logs are readable.

```bash
python3 tools/replay.py --kiss assets/captured_kiss.hex --host 127.0.0.1 --port 52001 --pace 0.8
```
**What happens:**
- Emulator logs **AUTH OK**, then **EXEC ACCEPTED → mode=OP owned=true**.
- No crypto broken: we simply **replayed** previously valid traffic lacking freshness binding.

Verify impact:
```bash
python3 tools/verify_success.py --url http://127.0.0.1:8000/status
```
Expected:
```json
{ "owned": true, "mode": "OP", ... }
```
**Why:** This mirrors a real record‑and‑replay against poorly designed command links or naive endpoints.

---

## Turn on defenses (blue-team experiments)

Stop the emulator (Ctrl‑C) 

Toggle each protection in `emulators/groundstation_vuln.py` and retry the replay to see which control blocks it.

Open the file and set:
```python
REQUIRE_FRESH_TS = True     # reject stale timestamps (±300s window)
REJECT_REPLAY = True        # keep recent frame hashes and drop duplicates
REQUIRE_AUTH_NONCE = True   # bind AUTH to a fresh server-provided nonce
```

Restart emulator:
```bash
python3 emulators/groundstation_vuln.py
```

### Freshness check
Replay again:
```bash
python3 tools/replay.py --kiss assets/captured_kiss.hex --host 127.0.0.1 --port 52001 --pace 0.8
```
Expected log: `STALE TS - dropping`.  
**Why:** Old frames should not be accepted as “fresh.”

### Replay cache
If you disable `REQUIRE_FRESH_TS` but keep `REJECT_REPLAY = True`, the first run may pass and **second** identical replay will be dropped as duplicate.  
**Why:** Caches block identical frame hashes even if timestamps look fine.

### Nonce-bound AUTH
With `REQUIRE_AUTH_NONCE = True`, the client must ask for a server nonce and echo it in `AUTH`.

Send a nonce request (a special synthetic command we added) and then AUTH with that nonce using the provided helper:

```bash
# Generate a fresh AUTH sequence using helper (writes to temp file)
python3 tools/packet_tools.py --show assets/captured_kiss.hex  # (just to remind format)
```

```bash
# Use the prebuilt nonce+auth demo:
python3 - <<'PY'
from tools.packet_tools import FEND,FESC,TFEND,TFESC,parse_payload,unkiss
import binascii,struct,time,sys
def kiss(payload):
    out=bytearray([0xC0,0x00])
    for b in payload:
        if b==0xC0: out+=bytes([0xDB,0xDC])
        elif b==0xDB: out+=bytes([0xDB,0xDD])
        else: out.append(b)
    out.append(0xC0); return bytes(out)
def build(cmd, ts=None, ctr=500):
    SYNC=0x1ACFFC1D
    if ts is None: ts=int(time.time())
    body=struct.pack(">I",SYNC)+struct.pack(">I",ts)+struct.pack(">H",ctr)
    p=cmd.encode(); body+=struct.pack(">H",len(p))+p
    crc=binascii.crc32(body)&0xFFFFFFFF; body+=struct.pack(">I",crc)
    return kiss(body)
# 1) Ask for nonce
open("nonce_then_auth.hex","w").write(build("CHALLENGE?").hex()+"
")
# 2) (Manually check nonce via /status), then craft AUTH with that nonce value:
print("Now run emulator; check nonce at http://127.0.0.1:8000/status and edit the next line.")
PY
```


- Visit `http://127.0.0.1:8000/status`, copy the `nonce` value, then craft AUTH:

```bash
python3 - <<'PY'
import binascii,struct,time
def kiss(payload):
    out=bytearray([0xC0,0x00])
    for b in payload:
        if b==0xC0: out+=bytes([0xDB,0xDC])
        elif b==0xDB: out+=bytes([0xDB,0xDD])
        else: out.append(b)
    out.append(0xC0); return bytes(out)
def build(cmd, ts=None, ctr=501):
    SYNC=0x1ACFFC1D
    if ts is None: ts=int(time.time())
    body=struct.pack(">I",SYNC)+struct.pack(">I",ts)+struct.pack(">H",ctr)
    p=cmd.encode(); body+=struct.pack(">H",len(p))+p
    crc=binascii.crc32(body)&0xFFFFFFFF; body+=struct.pack(">I",crc)
    return kiss(body)

nonce = input("Paste nonce from /status: ").strip()
open("nonce_then_auth.hex","a").write(build(f"AUTH token=abcd1234 nonce={nonce}").hex()+"
")
print("Wrote nonce_then_auth.hex")
PY
```

Send the two freshly crafted frames:
```bash
python3 tools/replay.py --kiss nonce_then_auth.hex --host 127.0.0.1 --port 52001 --pace 0.8
```
**Why:** You’re now seeing how **challenge‑response** thwarts naive replay of stale captures.


***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/redLabs/Lab5/The_Drift.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/Lab3/The_Takeover.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

