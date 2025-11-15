# RFQuack Simulation Lab


## What you’ll do
> Recon -> filter -> replay -> in-flight manipulation -> beat weak anti-replay -> observe jamming effects -> propose defenses

---

## Setup

Download the zip for this main folder from [Here](./RFQuack_Lab.zip)

- Click the Download button

<img width="330" height="177" alt="image" src="https://github.com/user-attachments/assets/df15f9ee-985a-4f6a-af65-32698e1aa337" />

- Extract it

Open three terminals in the lab directory

**Terminal A — Satellite (data + control):**
```bash
python sat_emulator.py --port 5000 --control-port 5001 --beacon 2
```

**Terminal B — (optional) Legit ground station that sends data:**
```bash
python traffic_gen.py --port 5000 --send-every 7
```

**Terminal C — Attacker shell:**
```bash
python rqshim.py
help
```

> RX is OFF by default in this shim, you won’t see frames until you `rx start`.

---

## Frame format (16 bytes)

```
PRE(2) | SYNC(2) | SAT(1) | MODE(1) | SEQ(1) | CMD(1) | PAD(6) | CRC(2)
```

- **SAT**: satellite ID (0xA1)  
- **MODE**: mode flag you’ll flip  
- **SEQ**: weak anti-replay counter (mod 256)  
- **CMD**: 0x02=BEACON_OFF, 0x03=BEACON_ON, 0x04=SET_MODE  
- **CRC**: integrity only (no auth)

---

## 1) Recon

Start capture:
```bash
rq> radio.set_modem_config modulation=2FSK carrierFreq=433.920
rq> rx start
```
You should now see beacons every ~2s

**Why:** You always need structure before attacks

---

## 2) Focus the signal (filter)

Keep only frames for SAT=0xA1:
```bash
rq> packet_filter add ^AAAA5555A1
rq> packet_filter list
```

**Why:** Mirrors RFQuack’s packet filter to kill noise

---

## 3) Replay

Let the ground station send something

Capture and replay:
```bash
rq> dump last
rq> send <paste the 16-byte hex>
```

**Expected:** Emulator toggles state and prints an ACK-style line
**Why:** CRC isn’t auth; replay still lands

---

## 4) In-flight manipulation (bit-flip)

Flip the MODE byte as frames pass and auto-forward:
```bash
rq> packet_manipulator add XOR 5 0x01   # offset 5 = MODE
rq> repeater on
rq> rx start
```

**Why:** On-path attackers can silently change command meaning

---

## 5) Beat weak anti-replay

Only SEQ is checked. Push a future SEQ and resend:
```bash
rq> packet_manipulator add XOR 6 0x10   # bump SEQ by 0x10
rq> send <previously accepted cmd hex>
```
Try other deltas if needed

**Why:** Freshness needs nonces/challenges, not bare counters

---

## 6) Jamming effects (simulated)

Tell the emulator to probabilistically drop inbound frames:
```bash
rq> jam start --duty 0.6
# watch for ~3 seconds
rq> jam stop
```

**Why:** Even short narrow interference disrupts C2

---

## Quick reference (shim commands)

- `rx start|stop` — enable/disable capture/forwarding  
- `packet_filter add <regex> | list | clear`  
- `packet_manipulator add <XOR|OR|AND|NOT> <offset> <0xVAL> | list | clear`  
- `repeater on|off`  
- `send <hex16>` — CRC auto-fixed  
- `dump last|n <count>`  
- `jam start --duty <0..1> | jam stop`  
- `exit`

