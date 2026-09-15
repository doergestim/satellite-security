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

In this section, we are going to trigger a replay attack.<br>
From a terminal, run the following:

```bash
seq 1 200 | xargs -I{} -P 50 sh -c \
 'curl -s -o /dev/null -X POST http://localhost:5000/ingest \
   -H "Content-Type: application/json" \
   --data "{\"test\":{}}"' 
```

<br>

Back over in Wireshark, we need to apply a filter by typing `Ctrl + /`
Then, paste the following: 
<pre>frame contains "ingest"</pre>

![image](/Assets/BLab1/BLab1-11.png)

Before we continue, we want to press the red square **STOP** button at the top left of the window.<br>
Next, at the top part of your window, go to **Statistics** -> **IO Graphs**<br>
This will pull up a graph window. Can you identify the spike in rate?

![image](/Assets/BLab1/BLab1-12.png)


---

## Hardening & Detection with Standard Tools

### Rate-limit with Nginx

#### What we are going to do

- Setup Groundstation to listen **127.0.0.1:5000**
- Setup Nginx to listen on **port 80**
- Forward all requests to **127.0.0.1:5000**
- Use Nginx to apply **rate limits** to protect the groundstation


To get started, open a terminal.<br>
Create the Nginx site config by running the following:

```bash
sudo nano /etc/nginx/sites-available/groundstation
```

<br>

Next, paste:

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

To save and exit do `Ctrl + x` and `y` and `Enter`<br>

Now we need to disable the default site:

```bash
sudo rm /etc/nginx/sites-enabled/default 2>/dev/null || true
```

And enable your site instead:

```bash
sudo ln -s /etc/nginx/sites-available/groundstation \
          /etc/nginx/sites-enabled/groundstation
```

Now it's time to test the config

```bash
sudo nginx -t
```

![image](/Assets/BLab1/BLab1-13.png)


Reload Nginx with the following command:

```bash
sudo systemctl reload nginx
```

Before we go any further, let's test access:

```bash
curl -v http://localhost/
```

![image](/Assets/BLab1/BLab1-14.png)


Next, trigger the rate limiting:

```bash
seq 1 200 | xargs -I{} -P 50 sh -c \
 'curl -s -o /dev/null -X POST http://localhost:5000/ingest \
   -H "Content-Type: application/json" \
   --data "{\"test\":{}}"' 
```

Now, watch Nginx logs:

```bash
sudo head -n 20 /var/log/nginx/error.log
```

![image](/Assets/BLab1/BLab1-15.png)



> Created By Turcu Știolică Alexandru - Black Hills Information Security
