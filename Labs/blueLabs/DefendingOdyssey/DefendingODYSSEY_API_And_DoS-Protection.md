![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Blue Lab 1 - Defending ODYSSEY-1: API & DoS Protection

**Scenario:** You are **blue team** for ODYSSEY-1.  
Red team has:
- Replayed stale telemetry into the groundstation
- Flooded `/login` and `/cmd` on your groundstation service

In this lab you will:

1. Use **Wireshark** to understand the replay and HTTP floods  
2. Use **Docker**, **Nginx**, **Fail2ban**, and **Suricata** to harden and monitor the groundstation

You already have under `~/Desktop/DefendingODYSSEY`:
- `groundstation/`

---

# Part B - Network Forensics: Replay & Flood Detection

## B1 - Observe normal traffic with Wireshark

1. Run groundstation:

```bash
cd /home/ubuntu/Desktop/DefendingODYSSEY/groundstation
```

```bash
sudo docker compose up --build
```

2. Open browser -> `http://localhost:5000`

![image](/Assets/BLab1/BLab1-6.png)

3. Open a new terminal. Then run:

```bash
sudo -E wireshark &
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
seq 1 200 | xargs -I{} -P 50 sh -c \
 'curl -s -o /dev/null -X POST http://localhost:5000/ingest \
   -H "Content-Type: application/json" \
   --data "{\"test\":{}}"' 
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
curl -v http://localhost:5000/
```

![image](/Assets/BLab1/BLab1-14.png)


- Trigger rate limiting

```bash
seq 1 200 | xargs -I{} -P 50 sh -c \
 'curl -s -o /dev/null -X POST http://localhost:5000/ingest \
   -H "Content-Type: application/json" \
   --data "{\"test\":{}}"' 
```

- Watch Nginx logs:

```bash
sudo head -n 20 /var/log/nginx/error.log
```

![image](/Assets/BLab1/BLab1-15.png)

---

***

<b><i>Continuing the course? </br>[Next Lab](/Labs/blueLabs/SatDump/SatDump.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](./Defending_Odyssey_RF-Analysis.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---


> Created By Turcu Știolică Alexandru - Black Hills Information Security
