
# Blue Lab 1 — Defending ODYSSEY-1: RF & Groundstation Defense

**Scenario:** You are **blue team** for ODYSSEY-1.  
Red team has:
- Jammed your BFSK downlink (`pass_jam_0dB.iq`, `pass_jam_-5dB.iq`)
- Replayed stale telemetry into the groundstation
- Flooded `/login` and `/cmd` on your groundstation service

In this lab you will:

1. Use **SDR tools** to **see and understand** the jamming at RF layer  
2. Use **Wireshark** to understand the replay and HTTP floods  
3. Use **Docker**, **Nginx**, **Fail2ban**, and **Suricata** to harden and monitor the groundstation

You already have under `~/Desktop/BlueLab`(the files from **Lab2**):
- `pass_clean.iq`
- `pass_jam_0dB.iq`
- `pass_jam_-5dB.iq`
- `groundstation/`

---

# Part A — RF: Visually Detect & Understand the Jamming

## Compare clean vs jammed signals in Gqrx

1. Open **Gqrx**, run

```bash
gqrx &
```

- Get the path of the files:

```bash
realpath pass_clean.iq
# /home/satuser/Desktop/BlueLab/pass_clean.iq
```

```bash
satuser@satvm:~/Desktop/BlueLab$ realpath pass_jam_0dB.iq 
# /home/satuser/Desktop/BlueLab/pass_jam_0dB.iq
```

```bash
satuser@satvm:~/Desktop/BlueLab$ realpath pass_jam_-5dB.iq 
# /home/satuser/Desktop/BlueLab/pass_jam_-5dB.iq
```

- If this window pops up, go to **File** -> **I/O Devices**

2. Select **Complex Sampled (IQ) File** Device
3. Input rate: **48000**
4. Device string: `file=/home/satuser/Desktop/BlueLab/pass_clean.iq,freq=437.5e6,rate=48000,repeat=true,throttle=true`

<img width="866" height="513" alt="image" src="https://github.com/user-attachments/assets/b163115f-ba48-46c5-b11f-7eff47539484" />

- Click **Ok**

5. Observe clean **BFSK tones** by pressing the **Arrow(play)** Button on the top-left

<img width="738" height="687" alt="image" src="https://github.com/user-attachments/assets/02486823-47e6-48ff-a6d0-16189bf05595" />

- Then repeat for (by replacing `pass_clean.iq` in the **Device string**):
- `pass_jam_0dB.iq`
- `pass_jam_-5dB.iq`

- Do it by going to **File** -> **I/O Devices**

Watch for:
- Raised noise floor  
- Smeared tones  
- Loss of clarity in waterfall  

- Here are the other 2:

<img width="738" height="684" alt="image" src="https://github.com/user-attachments/assets/d25ea8b9-71ca-4209-9fd5-5207870fbd05" />

<img width="738" height="684" alt="image" src="https://github.com/user-attachments/assets/408cace7-d781-4662-9374-61cc3ba4158a" />

# Clean Signal — Analysis

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


# Jammed Signal (0 dB) — Analysis

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


# Jammed Signal (–5 dB) — Analysis

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

<img width="1573" height="550" alt="image" src="https://github.com/user-attachments/assets/a13daa62-054e-4a52-a36d-0754327f7b9c" />

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

# Part B — Network Forensics: Replay & Flood Detection

## B1 — Observe normal traffic with Wireshark

1. Run groundstation:

```bash
cd groundstation
```

```bash
sudo docker compose up --build
```

2. Open browser -> `http://localhost:5000`

<img width="889" height="420" alt="image" src="https://github.com/user-attachments/assets/cceaeb7b-2a3f-4692-9d57-5b835dfd290b" />

3. Open a new terminal by pressing the button on the **Top-Left** of your already open **terminal** and open **Wireshark**

<img width="99" height="55" alt="image" src="https://github.com/user-attachments/assets/c1fef67e-bba5-4a00-b6a0-9c9bb3c3cf44" />

```bash
sudo wireshark &
```

4. Capture on `Loopback: lo`

<img width="771" height="140" alt="image" src="https://github.com/user-attachments/assets/604cf65c-0531-4771-9535-c97ee5b576ad" />

- Double **Click** on that

- Click any bigger **packet**

<img width="1024" height="292" alt="image" src="https://github.com/user-attachments/assets/b5bdc6f8-6b63-4f6e-8780-8349e7031082" />

<img width="558" height="204" alt="image" src="https://github.com/user-attachments/assets/bd98e5ee-d448-4e1a-b1e3-17911da2291f" />

---

## B2 — Watch replay attack in Wireshark

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

<img width="1042" height="341" alt="image" src="https://github.com/user-attachments/assets/a4833513-8ca5-407b-bf4c-0ab41da44e4a" />


3. On the top part of your window, go to **Statistics** -> **IO Graphs** -> **identify spike in rate**

<img width="536" height="416" alt="image" src="https://github.com/user-attachments/assets/e4c9e6e3-ebd4-4bda-b918-79b8372d27e0" />


---

# Part C — Hardening & Detection with Standard Tools

## C1 — Rate-limit with Nginx

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

<img width="600" height="44" alt="image" src="https://github.com/user-attachments/assets/81b2cc3e-f8ba-4587-b241-b83387e19c29" />

- Reload Nginx

```bash
sudo systemctl reload nginx
```

- Test access

```bash
curl -v http://localhost/
```

<img width="1385" height="515" alt="image" src="https://github.com/user-attachments/assets/1f6640e2-bbbe-47cf-a7f2-a2d2372b8614" />


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

<img width="1690" height="332" alt="image" src="https://github.com/user-attachments/assets/220369d3-9129-463f-9906-383a6fedd0da" />

---

## C2 — Auto-ban with Fail2ban

Create jail:

```ini
[groundstation-login]
enabled = true
port    = http,https
filter  = nginx-http-auth
logpath = /var/log/nginx/access.log
maxretry = 10
findtime = 60
bantime  = 600
```

Restart:

```bash
sudo systemctl restart fail2ban
```

Trigger login flood:

```bash
seq 1 1000 | xargs -I{} -P 50 sh -c \
  'curl -s -X POST http://localhost/login \
    -d user=admin -d pass=pwd{}'
```

Check status:

```bash
sudo fail2ban-client status groundstation-login
```


