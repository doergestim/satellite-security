![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Blue Lab 1 - Defending ODYSSEY-1: RF & Groundstation Defense

**Scenario:** You are **blue team** for ODYSSEY-1.  
Red team has:
- Jammed your BFSK downlink (`pass_jam_0dB.iq`, `pass_jam_-5dB.iq`)
- Replayed stale telemetry into the groundstation
- Flooded `/login` and `/cmd` on your groundstation service

In this lab you will:

1. Use **SDR tools** to **see and understand** the jamming at RF layer  
2. Use **Wireshark** to understand the replay and HTTP floods  
3. Use **Docker**, **Nginx**, **Fail2ban**, and **Suricata** to harden and monitor the groundstation

You already have under `~/Desktop/DefendingODYSSEY`(the files from **Lab2**):
- `pass_clean.iq`
- `pass_jam_0dB.iq`
- `pass_jam_-5dB.iq`
- `groundstation/`

---

# Part A - RF: Visually Detect & Understand the Jamming

## Compare clean vs jammed signals in Gqrx

1. Open **Gqrx**, run

```bash
gqrx &
```
In another Terminal:

- Get the path of the files:

```bash
cd ~/Desktop/DefendingODYSSEY
```

```bash
realpath pass_clean.iq
# /home/satuser/Desktop/DefendingODYSSEY/pass_clean.iq
```

```bash
realpath pass_jam_0dB.iq 
# /home/satuser/Desktop/DefendingODYSSEY/pass_jam_0dB.iq
```

```bash
realpath pass_jam_-5dB.iq 
# /home/satuser/Desktop/DefendingODYSSEY/pass_jam_-5dB.iq
```

- If this window pops up, go to **File** -> **I/O Devices**

2. Select **Complex Sampled (IQ) File** Device
3. Input rate: **48000**
4. Device string: `file=/home/satuser/Desktop/DefendingODYSSEY/pass_clean.iq,freq=437.5e6,rate=48000,repeat=true,throttle=true`

![image](/Assets/BLab1/BLab1-1.png)

<img width="690" height="469" alt="image" src="https://github.com/user-attachments/assets/0f4434b8-be3b-44ec-acdb-64e84591f8b0" />

- Click **Ok**

5. Observe clean **BFSK tones** by pressing the **Arrow(play)** Button on the top-left

![image](/Assets/BLab1/BLab1-2.png)

- Then repeat for (by replacing `pass_clean.iq` in the **Device string**):
- `pass_jam_0dB.iq`
- `pass_jam_-5dB.iq`

- Do it by going to **File** -> **I/O Devices**

Watch for:
- Raised noise floor  
- Smeared tones  
- Loss of clarity in waterfall  

- Here are the other 2:

![image](/Assets/BLab1/BLab1-3.png)

![image](/Assets/BLab1/BLab1-4.png)

# Clean Signal - Analysis

### Observations
- Two strong vertical stripes indicating clear BFSK tones.
- Noise floor is low, roughly –110 to –100 dBFS.
- Signal stands well above the noise, giving high SNR.
- Waterfall is stable and organized.
- Symbol transitions look sharp and well-defined.

### Interpretation
- This is a healthy downlink.
- The demodulator will lock easily and decode reliably.
- Represents normal, expected operational conditions.


# Jammed Signal (0 dB) - Analysis

### Observations
- BFSK tones are still visible but partially submerged.
- Noise floor is raised significantly, around –90 to –80 dBFS.
- Waterfall shows extra broadband noise.
- Tone edges are less distinct.
- SNR is reduced but not destroyed.

### Interpretation
- This is a partial denial situation.
- Demodulator may still decode some packets but with higher BER.
- Operators would see intermittent or glitchy telemetry.
- Packet loss is likely but not total.


# Jammed Signal (–5 dB) - Analysis

### Observations
- Noise floor is almost equal to or above the signal.
- BFSK tones are faint and hard to distinguish.
- Waterfall is nearly uniformly bright, indicating strong wideband jamming.
- Symbol structure is buried in noise.
- Effective SNR is below 0 dB.

### Interpretation
- This is a total denial scenario.
- Demodulator will not lock or decode correctly.
- Operators would experience complete loss of telemetry.
- Groundstation would show stale or missing frames.

---

## Inspect symbols in Inspectrum

1. Start **Inspectrum**

```bash
inspectrum &
```

2. Set sample rate: **48000**
3. Load `pass_clean.iq` by pressing **Input file** and selecting that file

![image](/Assets/BLab1/BLab1-5.png)

4. What to look for:

### Identify the two BFSK tones
- Look for two horizontal bright lines (top tone and bottom tone).
- These represent the two frequencies used for BFSK (Mark and Space).
- In a clean signal, both tones should be stable and sharp.

### Verify that the tones switch at regular intervals
- BFSK encodes bits by switching between the two tones.
- In Inspectrum, this appears as alternating bright segments between the two lines.
- Good signal = clear, rectangular transitions between tones.

### Locate symbol boundaries
- Use the horizontal time axis.
- Each repeated “block” of patterns corresponds to symbols or frames.
- Clean signals show:
  - Repeating sections
  - Well-defined on/off transitions
  - Minimal blurring

### Look for the preamble structure
- Satellite packets often start with a known pattern (e.g., alternating tones).
- These appear as highly regular, tightly spaced transitions at the start of each frame.
- In your screenshot, these are the short, very crisp segments at the left of each block.

### Check for noise or corruption
- Clean signal:
  - Background is dark blue/green.
  - Tones stand out clearly in bright yellow.
- Jammed or degraded signal:
  - Background becomes grainier.
  - Tones smear or lose sharpness.
  - Transitions look “fuzzy” or inconsistent.

### Confirm symbol rate consistency
- The distance (time) between tone transitions should be constant.
- Clean signal:
  - Blocks are evenly spaced.
  - Symbol spacing is identical throughout.
- Jammed signal:
  - Harder to see transitions.
  - Blocks may look smeared over time.

---

- Repeat for jammed files; observe corruption of symbol structure.

# Part B - Network Forensics: Replay & Flood Detection

## B1 - Observe normal traffic with Wireshark

1. Run groundstation:

```bash
cd groundstation
```

```bash
sudo docker compose up --build
```

2. Open browser -> `http://localhost:5000`

![image](/Assets/BLab1/BLab1-6.png)

3. Open a new terminal by pressing the button on the **Top-Left** of your already open **terminal** and open **Wireshark**

![image](/Assets/BLab1/BLab1-7.png)

```bash
sudo wireshark &
```

4. Capture on `Loopback: lo`

![image](/Assets/BLab1/BLab1-8.png)

- Double **Click** on that

- Click any bigger **packet**

![image](/Assets/BLab1/BLab1-9.png)

![image](/Assets/BLab1/BLab1-10.png)

---

## B2 - Watch replay attack in Wireshark

1. Trigger replay (from a terminal):

```bash
seq 1 500 | xargs -I{} -P 20 sh -c \
 'curl -s -X POST http://localhost:5000/ingest \
   -H "Content-Type: application/json" \
   --data-binary @stale.json'
```

2. In Wireshark:
   - Apply filter with `Ctrl + /` and paste this: `frame contains "ingest"`
   - See many POSTs to `/ingest`

![image](/Assets/BLab1/BLab1-11.png)


3. On the top part of your window, go to **Statistics** -> **IO Graphs** -> **identify spike in rate**

![image](/Assets/BLab1/BLab1-12.png)


---

# Part C - Hardening & Detection with Standard Tools

## C1 - Rate-limit with Nginx

### What we are going to do

- Groundstation listens on **127.0.0.1:5000**
- Nginx will listen on **port 80**
- All requests will be forwarded to **127.0.0.1:5000**
- Nginx will apply **rate limits** to protect the groundstation

- Create the Nginx site config

```bash
sudo nano /etc/nginx/sites-available/groundstation
```

- Paste:

```nginx
limit_req_zone $binary_remote_addr zone=odysseyratelimit:10m rate=10r/s;

server {
    listen 80;
    server_name _;

    location / {
        limit_req zone=odysseyratelimit burst=20 nodelay;
        proxy_pass http://127.0.0.1:5000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- To save and exit do `Ctrl + x` and `y` and `Enter`

- Disable default site

```bash
sudo rm /etc/nginx/sites-enabled/default 2>/dev/null || true
```

- Enable your site

```bash
sudo ln -s /etc/nginx/sites-available/groundstation \
          /etc/nginx/sites-enabled/groundstation
```

- Test config

```bash
sudo nginx -t
```

![image](/Assets/BLab1/BLab1-13.png)

- Reload Nginx

```bash
sudo systemctl reload nginx
```

- Test access

```bash
curl -v http://localhost/
```

![image](/Assets/BLab1/BLab1-14.png)


- Trigger rate limiting

```bash
seq 1 200 | xargs -I{} -P 50 sh -c \
 'curl -s -o /dev/null -X POST http://localhost/ingest \
   -H "Content-Type: application/json" \
   --data "{\"test\":{}}"' 
```

- Watch Nginx logs:

```bash
sudo head -n 20 /var/log/nginx/error.log
```

![image](/Assets/BLab1/BLab1-15.png)


***                                                                 

<b><i>Continuing the course? </br>[Next Lab](/Labs/blueLabs/SatDump/SatDump.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/Lab5/The_Drift.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---
