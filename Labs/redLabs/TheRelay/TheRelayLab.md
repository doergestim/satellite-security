![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 4 - The Relay

**Scenario:** See how an attacker can **record** valid satellite command traffic and **replay** it later to a naive groundstation that lacks freshness checks. You’ll plan a pass, review/parse captured frames, then replay them into a local vulnerable groundstation emulator and measure impact. Finally, you’ll turn on defenses and re-test.

**You’ll practice:** pass planning (gpredict), temporal alignment & TLE integrity, KISS framing basics, safe replay, and blue-team mitigations (timestamps, nonces, dedup)

---

## Setup 

- Enter the lab folder

```bash
cd ~/Desktop/TheRelay
```

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

- Start gpredict from a terminal

```bash
gpredict
```

In gpredict:

- **Edit -> Update TLE data from local files**: choose `assets/` and *open*

<img width="320" height="160" alt="image" src="https://github.com/user-attachments/assets/464d589c-809d-4a5c-b9fc-79f58e685e64" />



- **File -> New Module**: create a new module (“DEMO-Lab”)

<img width="656" height="119" alt="image" src="https://github.com/user-attachments/assets/99f37bd5-6d7b-4bb9-bec0-64995b811b04" />


- **Satellites -> Search for our DEMO-SAT**: pick `assets/DEMO-SAT.tle`

<img width="738" height="384" alt="image" src="https://github.com/user-attachments/assets/a81fb622-e20f-4d6a-a06a-363ee7e89ce1" />


**Technical Note:** In Satellite Security, "Freshness" applies to more than just passwords; it applies to orbital data. A TLE is only an approximation. 

The SGP4 algorithm used by Gpredict loses accuracy at a rate of roughly 1–3 km per day. After several months, a satellite will be "predicted" thousands of kilometers away from its actual location. This is a critical lesson for attackers: you cannot intercept or spoof a signal if you don't have up-to-date orbital intelligence! 

**The "Why":** To illustrate the effect of *TLE Drift*, the synthetic example refers to a specific satellite pass in September 01, 2025, at 14:43:00

- Observe the current Gpredict data:
 
 <img width="706" height="730" alt="image" src="https://github.com/user-attachments/assets/85f01a52-ad0d-4170-9c2a-3aae03c358d1" />

- The current (2026) position for DEMO-SAT appears illogical, due to the *TLE Drift* effect. To see exactly what the operator saw during the original attack, we must align the simulation's clock with our "Forensic Evidence."

**Temporal Alignment Configuration Steps:**
- Open Time Controller : Select the date and time
  
<img width="633" height="422" alt="image" src="https://github.com/user-attachments/assets/25ac4885-2c7a-48c3-9b6e-daf7c895c252" />

- Adjust the Time Controller: **Simulated -Real-Time -> Manual Control**
  
<img width="371" height="70" alt="image" src="https://github.com/user-attachments/assets/1417273f-ebac-41b5-acab-0d3dfeae7f57" />
<br />
<img width="371" height="70" alt="image" src="https://github.com/user-attachments/assets/d7af4e3f-bcb9-4d1f-96b7-743188462d5f" />

- Select **September 01, 2025, 14:43:00**
  
<img width="373" height="247" alt="image" src="https://github.com/user-attachments/assets/3f0fe2da-eb57-4d9d-b287-3bc312407e4f" />

>[!IMPORTANT]
>In a real-world tactical scenario, orbital data (TLEs) must be used immediately following their release or capture. The extreme "drift" and map distortion observed in this lab are intentional, used here to demonstrate the impact of TLE Decay over several months.

**Observe the Correction:** The satellite should now display a clean, circular footprint


- Observe **AOS/LOS** and nominal UHF center frequency you’d monitor (synthetic example: 437.500 MHz)

<img width="733" height="731" alt="image" src="https://github.com/user-attachments/assets/38859e4f-0359-4681-b074-93d37f81749b" />


**Why:** See how real operators plan captures, we’ll still use a synthetic capture next

- Close **gpredict** and go back to the terminal

---

## Inspect the provided capture

Open and parse the KISS frames we provide

This simulates “post‑demod” packets an attacker or operator might export

>[!IMPORTANT]
>Make sure you are at `~/Desktop/TheRelay/`

- Show frames and parsed fields

```bash
python3 tools/packet_tools.py --show assets/captured_kiss.hex
```

![image](/Assets/RLab4/RLab4-5.png)


**What you’ll see:**
- `timestamp`: embedded epoch the sender claimed when the frame was created
- `counter`: a simplistic sequence value
- `data_utf8`: command payload string
- `crc_ok`: integrity over the plaintext payload passes (not a cryptographic MAC)

**Why:** Know exactly what you’re about to replay

---

## Start the vulnerable groundstation emulator

The emulator listens for **KISS** frames on **TCP 52001** and exposes a JSON status at **http://127.0.0.1:8000/status** 
It intentionally lacks freshness and replay protections by default

- Run the emulator

```bash
python3 emulators/groundstation_vuln.py
```

Keep it running in its terminal. Open a second terminal (or tab) for the replay steps, activate the venv again:

- In a new terminal tab, back in the lab folder

```bash
cd ~/Desktop/TheRelay
```

Quick health check:

```bash
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

And:

<pre>
[!] Replay did not achieve unauthorized state (check emulator logs).    <-- because we didn't play the replay yet, it is normal!
</pre>

**Why:** Confirm the target is reachable and in a known-good state

---

## Replay the capture

Send the two frames (AUTH followed by EXEC) to the emulator. Pace them at ~0.8 s so logs are readable.

```bash
python3 tools/replay.py --kiss assets/captured_kiss.hex --host 127.0.0.1 --port 52001 --pace 0.8
```

**What happens(check the logs back in the first terminal as well):**
- Emulator logs **AUTH OK**, then **EXEC ACCEPTED -> mode=OP owned=true**.
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

Open the file:

```bash
nano emulators/groundstation_vuln.py
```

and set:
```python
REQUIRE_FRESH_TS = True     # reject stale timestamps (±300s window)
REJECT_REPLAY = True        # keep recent frame hashes and drop duplicates
REQUIRE_AUTH_NONCE = True   # bind AUTH to a fresh server-provided nonce
```

To save and exit do `Ctrl+x` + `y` and `Enter`

Restart emulator by going to the first terminal, do `Ctrl+c` to stop it and run it again:

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

With `REQUIRE_AUTH_NONCE = True`, the client must request a server-issued nonce and echo it back inside `AUTH`. Old captured frames carry no valid nonce, so they're rejected outright.

Rather than running two separate scripts and pasting values by hand, use the provided helper - it handles the full challenge-response flow automatically.

First, create the script:

```bash
cat > tools/nonce_auth_demo.py << 'EOF'
#!/usr/bin/env python3
"""
nonce_auth_demo.py  –  Full challenge-response AUTH flow in one script.

Usage:
    python3 tools/nonce_auth_demo.py [--host HOST] [--port PORT]
                                     [--status-url URL] [--token TOKEN]
                                     [--out FILE]
"""

import binascii, socket, struct, time, urllib.request, json, argparse

# ── KISS helpers ──────────────────────────────────────────────────────────────

FEND, FESC, TFEND, TFESC = 0xC0, 0xDB, 0xDC, 0xDD
SYNC = 0x1ACFFC1D

def kiss_wrap(payload: bytes) -> bytes:
    out = bytearray([FEND, 0x00])
    for b in payload:
        if   b == FEND: out += bytes([FESC, TFEND])
        elif b == FESC: out += bytes([FESC, TFESC])
        else:           out.append(b)
    out.append(FEND)
    return bytes(out)

def build_frame(cmd: str, ctr: int = 0) -> bytes:
    ts   = int(time.time())
    body = struct.pack(">I", SYNC) + struct.pack(">I", ts) + struct.pack(">H", ctr)
    enc  = cmd.encode()
    body += struct.pack(">H", len(enc)) + enc
    crc  = binascii.crc32(body) & 0xFFFFFFFF
    body += struct.pack(">I", crc)
    return kiss_wrap(body)

# ── Network helpers ───────────────────────────────────────────────────────────

def send_frame(frame: bytes, host: str, port: int) -> None:
    with socket.create_connection((host, port), timeout=5) as s:
        s.sendall(frame)

def poll_nonce(status_url: str, retries: int = 10, delay: float = 0.5) -> str:
    """Keep asking /status until the emulator has issued a nonce."""
    for _ in range(retries):
        with urllib.request.urlopen(status_url, timeout=3) as r:
            data = json.loads(r.read())
        nonce = data.get("nonce")
        if nonce:
            return nonce
        time.sleep(delay)
    raise RuntimeError("Emulator never issued a nonce – is it running with REQUIRE_AUTH_NONCE=True?")

# ── Main flow ─────────────────────────────────────────────────────────────────

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host",       default="127.0.0.1")
    ap.add_argument("--port",       default=52001, type=int)
    ap.add_argument("--status-url", default="http://127.0.0.1:8000/status")
    ap.add_argument("--token",      default="abcd1234")
    ap.add_argument("--out",        default="nonce_then_auth.hex")
    args = ap.parse_args()

    frames = []

    # Step 1 – send CHALLENGE? so the emulator generates a nonce
    print("[1/3] Sending CHALLENGE? ...")
    challenge = build_frame("CHALLENGE?", ctr=500)
    send_frame(challenge, args.host, args.port)
    frames.append(challenge)

    # Step 2 – wait for the nonce to appear in /status
    print("[2/3] Polling for nonce ...")
    nonce = poll_nonce(args.status_url)
    print(f"      Got nonce: {nonce}")

    # Step 3 – build AUTH frame that echoes the nonce, write combined hex file
    print("[3/3] Building AUTH frame ...")
    auth = build_frame(f"AUTH token={args.token} nonce={nonce}", ctr=501)
    frames.append(auth)

    with open(args.out, "w") as f:
        for frame in frames:
            f.write(frame.hex() + "\n")

    print(f"\n✓  Wrote {len(frames)} frames to {args.out}")
    print(f"   Replay with:  python3 tools/replay.py --kiss {args.out} "
          f"--host {args.host} --port {args.port} --pace 0.8")

if __name__ == "__main__":
    main()
EOF
```

```bash
python3 tools/nonce_auth_demo.py
```

**What it does, step by step:**

1. Builds a `CHALLENGE?` frame and sends it to the emulator - this triggers nonce generation
2. Polls `http://127.0.0.1:8000/status` until the nonce appears
3. Builds a fresh `AUTH token=abcd1234 nonce=<fetched>` frame embedding that nonce
4. Writes both frames to `nonce_then_auth.hex`

You'll see output like:
```
[1/3] Sending CHALLENGE? ...
[2/3] Polling for nonce ...
      Got nonce: a3f7c2e1
[3/3] Building AUTH frame ...

✓  Wrote 2 frames to nonce_then_auth.hex
   Replay with:  python3 tools/replay.py --kiss nonce_then_auth.hex --host 127.0.0.1 --port 52001 --pace 0.8
```

Once the script is in place, run it:

```bash
python3 tools/nonce_auth_demo.py
```

Then replay the freshly crafted frames:

```bash
python3 tools/replay.py --kiss nonce_then_auth.hex --host 127.0.0.1 --port 52001 --pace 0.8
```

Expected emulator log: `AUTH OK -> EXEC ACCEPTED -> mode=OP`

Now try replaying the **original** stale capture:

```bash
python3 tools/replay.py --kiss assets/captured_kiss.hex --host 127.0.0.1 --port 52001 --pace 0.8
```

Expected log: `NONCE MISMATCH - dropping`

**Why:** The nonce is single-use and server-generated. An attacker who recorded yesterday's traffic has no way to forge a valid nonce for today's session - challenge-response breaks replay even when timestamps are absent or clocks are unsynchronized.




***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/redLabs/TheDrift/TheDriftLab.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/TheTakeover/TheTakeoverLab.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
