![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 3 - The Takeover


**Scenario:** You’ve successfully intercepted ODYSSEY-1’s downlink. Inside the captured traffic, you spotted telemetry and even an ACK packet that contains a mysterious `auth` field

Your task is to **reverse engineer the uplink authentication scheme, craft a forged control command, and trick the ground station into accepting it**

## Provided
- ``assets/takeover_pass.iq`` - synthetic downlink (48 kS/s cf32)
- ``groundstation/`` - control app (verifies your forged uplink)
- ``tools/craft_uplink_template.py`` - OPTIONAL helper

Protocol recap (from Lab 1):
- SYNC=0x1ACFFC1D
- Frame: [SYNC(4)][VER(1)][SEQ(2)][TYPE(1)][LEN(2)][PAY(JSON)][CRC16(2)]
- CRC16-CCITT over [VER..PAY]

Auth rule (discover/confirm from pass): auth = sha1(str(epoch) + sat + "-BLUE")[:8]


---

# Start
## Part A - Decode the Downlink

1. **Open GNU Radio Companion (GRC)** and take the flow from [Lab 1](../TheIntercepter/TheIntercepterLab.md)

2. Run it. This will output raw bits into `takeover_pass_BPF.txt`.

3. **Parse the frames** (reuse the Lab 1 parser).
   - Extract JSON payloads
   - Find the telemetry packet with `epoch` and `sat`
   - Find the ACK packet containing `auth=xxxxxxx` (this is your clue)

Just replace the ``File Source`` block with our new `/home/ubuntu/Desktop/TheTakeover/assets/takeover_pass.iq` and the `File Sink` blocks to their respective names( `/home/ubuntu/Desktop/TheTakeover/assets/takeover_pass.bits` and `/home/ubuntu/Desktop/TheTakeover/assets/takeover_pass_BPF.txt` ) and run it

![image](/Assets/RLab3/RLab3-1.png)

4. The **assets** folder should now look like this

```bash
cd ~/Desktop/TheTakeover/
```

```bash
ls assets/
```

![image](/Assets/RLab3/RLab3-2.png)

5. View `takeover_pass_BPF.txt`

```bash
xxd -g1 assets/takeover_pass_BPF.txt
```

![image](/Assets/RLab3/RLab3-3.png)


6. **Work out the auth rule**:  
   The hint in telemetry says:  
   ```
   auth = sha1(str(epoch) + sat + "-BLUE")[:8]
   ```

---

## Part B - Forge the Uplink

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

   
![image](/Assets/RLab3/RLab3-4.png)

### If you want to learn and build the payload from scratch, go [here](./Build_Payload_From_Scratch.md)

---

## Part C - Send It

1. Start the control app:  
   ```bash
   cd groundstation
   sudo docker compose up --build
   # Visit http://localhost:5000 in your browser
   ```

2. Submit your packet:

- In another terminal:

```bash
cd ~/Desktop/TheTakeover/
```

- **Binary upload:**

```bash
curl -F "packet=@uplink.bin" http://localhost:5000/uplink | jq .
```

- **Hex JSON:**

```bash
HEX=$(xxd -p uplink.bin | tr -d '\n')
```
  
```bash
 curl -H "Content-Type: application/json" -d "{\"hex\":\"$HEX\"}" http://localhost:5000/uplink | jq .
```

![image](/Assets/RLab3/RLab3-5.png)


3. If correct you’ll see:  
   ```json
   {
     "ok": true,
     "ack": "SET_MODE CAL accepted",
     "flag": "FLAG_TAKEOVER{mode_set_CAL}"
   }
   ```

4. Check the dashboard (`/` endpoint) - it should now display your new mode and auth tag



***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/redLabs/TheRelay/TheRelayLab.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/TheDenial/TheDenialLab.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
