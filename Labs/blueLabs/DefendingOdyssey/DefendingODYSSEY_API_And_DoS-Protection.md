![image](/Assets/Attachments/blueantisyphon.png)

# Blue Lab 1 - Defending ODYSSEY-1: API & DoS Protection

#### This lab requires the use of the **Hacking & Defending Satellite Infrastructure w/ John Strand** VM.<br>
If you do not have this VM, please contact us!

<hr>

## Lab Scenario

In this lab, your are **blue team** for ODYSSEY-1.  
Red team has:
- Replayed stale telemetry into the groundstation
- Flooded `/login` and `/cmd` on your groundstation service

<hr>

## Lab Overview

In this lab you will:

1. Use **Wireshark** to understand the replay and HTTP floods  
2. Use **Docker**, **Nginx**, **Fail2ban**, and **Suricata** to harden and monitor the groundstation

You already have under `~/Desktop/DefendingODYSSEY`:
- `groundstation/`

---

## Network Forensics: Replay & Flood Detection

### Observe normal traffic with Wireshark

Before we begin, we need to open a terminal.



Next, we need to create our groundstation by running the following commands:

```bash
cd /home/ubuntu/Desktop/DefendingODYSSEY/groundstation
```

```bash
sudo docker compose up --build
```

Now we need to open a browser and navigate to `http://localhost:5000`

![image](/Assets/BLab1/BLab1-6.png)

If you've made it this far, it means that the docker was successfully built.<br>
Next, we need to launch Wireshark.<br>
Open a new terminal and run the following:

```bash
sudo -E wireshark &
```

Once the window is up, we want to capture on `Loopback: lo`<br>
Go ahead and **Double-Click** on it.

![image](/Assets/BLab1/BLab1-8.png)

Click any of the bigger **packets**.

![image](/Assets/BLab1/BLab1-9.png)

![image](/Assets/BLab1/BLab1-10.png)

<br>

### Watch replay attack in Wireshark

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

## Part C - Hardening & Detection with Standard Tools

### Rate-limit with Nginx

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
