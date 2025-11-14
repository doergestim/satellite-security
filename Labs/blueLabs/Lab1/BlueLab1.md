
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


## Inspect symbols in Inspectrum

1. Start **Inspectrum**
2. Load `pass_clean.iq`
3. Set sample rate: **48000**
4. Zoom in until symbol timing is visible
5. Use *symbol period markers* to estimate symbol rate

Repeat for jammed files; observe corruption of symbol structure.

## Verify demodulator failure in GNU Radio Companion

1. Load your Lab1/2 flowgraph.
2. Set File Source → `pass_clean.iq`
3. Run: note consistent access-code hits and decoded packets.
4. Switch to `pass_jam_0dB.iq` and then `pass_jam_-5dB.iq`.

Observe:
- Tag Debug shows fewer or zero hits  
- Output becomes empty or corrupted  

---

# Part B — Network Forensics: Replay & Flood Detection

## B1 — Observe normal traffic with Wireshark

1. Run groundstation:

```bash
cd groundstation
sudo docker compose up
```

2. Open browser → `http://localhost:5000`
3. Open **Wireshark**, capture on `lo` or `docker0`.
4. Filter:

```
http
```

Observe:
- `/stream` SSE  
- Normal telemetry POSTs  

---

## B2 — Watch replay attack in Wireshark

1. Trigger replay (from red lab):

```bash
seq 1 500 | xargs -I{} -P 20 sh -c \
 'curl -s -X POST http://localhost:5000/ingest \
   -H "Content-Type: application/json" \
   --data-binary @stale.json'
```

2. In Wireshark:
   - See many POSTs to `/ingest`
   - Follow TCP stream
   - Observe identical JSON bodies (`epoch`, `counter` repeated)

3. IO Graphs → identify spike in rate.

---

## B3 — Optional: Inspect `/ingest` & `/login` with Burp Suite

1. Open Burp.
2. Set browser proxy to `127.0.0.1:8080`.
3. Interact with:

- `/login`
- `/ingest`
- `/cmd`

4. Replay `/ingest` via Burp Repeater — observe acceptance of identical stale telemetry.

---

# Part C — Hardening & Detection with Standard Tools

## C1 — Rate-limit with Nginx

1. Install/enable nginx.
2. Create:

```nginx
limit_req_zone $binary_remote_addr zone=odysseyratelimit:10m rate=10r/s;

server {
    listen 80;
    server_name _;

    location / {
        limit_req zone=odysseyratelimit burst=20 nodelay;
        proxy_pass http://127.0.0.1:5000;
    }
}
```

3. Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/groundstation \
          /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

4. Rerun replay & flood through port **80**.
   - Check:

```bash
sudo tail -f /var/log/nginx/error.log
```

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

---

## C3 — Detect floods with Suricata IDS

Add to `/etc/suricata/rules/local.rules`:

```text
alert http any any -> any any (
  msg:"ODYSSEY-1 Telemetry Replay / Ingest Flood";
  http.method; content:"POST"; nocase;
  http.uri; content:"/ingest"; nocase;
  detection_filter:track by_src, count 50, seconds 5;
  sid:1000001; rev:1;
)
```

Run IDS:

```bash
sudo suricata -i lo -c /etc/suricata/suricata.yaml
```

Trigger replay again; watch:

```bash
sudo tail -f /var/log/suricata/fast.log
```

---

## C4 — Monitor container resource impact

```bash
sudo docker stats
```

Then run `/cmd` flood again to see saturation.

Optionally throttle container:

```bash
sudo docker update --cpus 0.2 <container_name>
```

Send a normal `/cmd` request to observe latency changes:

```bash
curl -s -H "Content-Type: application/json" \
     --data '{"mode":"SAFE"}' http://localhost/cmd
```
