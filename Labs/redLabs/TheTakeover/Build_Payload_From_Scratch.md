# Build the Uplink Payload **from scratch** (no helper)

This shows exactly how to craft the bytes for the forged uplink **without** using the helper script

## 0) Values to plug in
From your decode (Part A), write these down:
- `epoch` (e.g., `1713372600`)
- `sat` name: `ODYSSEY-1`

Compute the short auth tag:
```bash
printf "%s" "1713372600ODYSSEY-1-BLUE" | sha1sum | cut -c1-8
# -> e.g.  a1b2c3d4
```

## 1) Create the JSON payload bytes
Payload (TYPE=3):
```json
{"cmd":"SET_MODE","mode":"CAL","epoch":1713372600,"sat":"ODYSSEY-1","auth":"a1b2c3d4"}
```
Note: The **CRC is over the exact bytes** you send here. JSON whitespace matters because it changes bytes and length

To generate compact JSON consistently, use `json.dumps(..., separators=(',',':'))` or write the string exactly as above (no spaces)

## 2) Build the frame header
Big-endian fields:
```
SYNC(4)  = 0x1ACFFC1D
VER(1)   = 0x01
SEQ(2)   = any (e.g., 0x1092)   # you choose
TYPE(1)  = 0x03                 # command
LEN(2)   = payload length in bytes
PAYLOAD  = the JSON bytes
CRC16(2) = CCITT over [VER..PAYLOAD], init=0xFFFF, poly=0x1021
```
CRC does **not** include SYNC. CRC covers bytes starting at `VER` and ending at the last byte of the payload.

## 3) Pure-Python “from scratch” builder
Create `tools/uplink_from_scratch.py` with this content and edit the two variables at the top:
```python
#!/usr/bin/env python3
import struct, hashlib

# === EDIT THESE TWO FROM YOUR DECODE ===
epoch = 1713372600
sat   = "ODYSSEY-1"
# =======================================

SYNC = 0x1ACFFC1D

def crc16_ccitt(data: bytes, init=0xFFFF) -> int:
    # CCITT-FALSE: poly 0x1021, init 0xFFFF
    crc = init
    for b in data:
        crc ^= (b << 8)
        for _ in range(8):
            if crc & 0x8000:
                crc = ((crc << 1) ^ 0x1021) & 0xFFFF
            else:
                crc = (crc << 1) & 0xFFFF
    return crc

def be16(x): return struct.pack(">H", x)
def be32(x): return struct.pack(">I", x)

auth = hashlib.sha1(f"{epoch}{sat}-BLUE".encode()).hexdigest()[:8]

payload = (
    '{"cmd":"SET_MODE","mode":"CAL","epoch":'
    + str(epoch)
    + ',"sat":"'
    + sat
    + '","auth":"'
    + auth
    + '"}'
).encode("utf-8")

ver  = 0x01
seq  = 0x4242         # pick any 16-bit sequence
ptype= 0x03
ln   = len(payload)

header_wo_sync = bytes([ver]) + be16(seq) + bytes([ptype]) + be16(ln)
crc = crc16_ccitt(header_wo_sync + payload)

packet = be32(SYNC) + header_wo_sync + payload + be16(crc)
open("uplink.bin", "wb").write(packet)

print("epoch:", epoch, "sat:", sat, "auth:", auth)
print("len(payload):", ln, "crc16:", f"0x{crc:04X}")
print("first 32 bytes:", packet[:32].hex())
```

Run it:
```bash
python3 tools/uplink_from_scratch.py
xxd -g1 uplink.bin
```

## 4) Submit
```bash
cd groundstation
docker compose up --build
# in another terminal
curl -F "packet=@../uplink.bin" http://localhost:5000/uplink | jq .
```

You should get:
```json
{"ok":true,"ack":"SET_MODE CAL accepted","flag":"FLAG_TAKEOVER{mode_set_CAL}"}
```

## 5) Common mistakes
- CRC includes SYNC -> **wrong** (exclude SYNC)
- JSON spacing differs -> CRC mismatch (CRC must match the bytes you actually send)
- Wrong TYPE (must be `0x03`) or wrong LEN
- Auth computed from wrong `epoch`/`sat`
