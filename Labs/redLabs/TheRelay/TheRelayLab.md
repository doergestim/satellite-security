![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 4 - The Relay

**Scenario:** Show how an attacker can **record** valid satellite command traffic and **replay** it later to a naive groundstation that lacks freshness checks. You’ll plan a pass, review/parse captured frames, then replay them into a local vulnerable groundstation emulator and measure impact. Finally, you’ll turn on defenses and re-test.

**You’ll practice:** pass planning (gpredict), KISS framing basics, safe replay, and blue-team mitigations (timestamps, nonces, dedup)

---

## Setup 

```bash
# enter the lab folder
cd ~/Desktop/TheRelay
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
>Make sure you are at `~/Desktop/TheRelay/`

```bash
cd ~/Desktop/TheRelay
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
cd ~/Desktop/TheRelay
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

> [!TIP]
> You can override defaults with flags if your setup differs:
> ```bash
> python3 tools/nonce_auth_demo.py --host 127.0.0.1 --port 52001 \
>     --status-url http://127.0.0.1:8000/status --token abcd1234
> ```


***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/redLabs/TheDrift/TheDriftLab.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/TheTakeover/TheTakeoverLab.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
