![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 6 - Exploiting an Unauthenticated SLE Management Plane

## Context
This lab demonstrates a **realistic vulnerability class**:  
- **unauthenticated exposure of a management/control plane** in a protocol-heavy system

You are assessing a **VisionSpace SLE Provider** deployed with insecure defaults 

Your goal is to **enumerate, validate impact, and remediate** the issue

---

## Start

```bash
cd ~/Desktop/SLEHacking
```

```bash
sudo docker compose up --build -d
```

---

## 1. Establish Baseline Exposure

### 1.1 Identify exposed ports
```bash
sudo docker ps --format "table {{.Names}}\t{{.Ports}}"
```

<img width="1781" height="57" alt="2026-03-19_11-52" src="https://github.com/user-attachments/assets/0ee86895-9ae7-41dd-a5ad-ce8473c3ad63" />


Expected exposure:
- **TCP 2048** -> Management REST API
- **TCP 55529** -> SLE user port
- **UDP 16887–16888** -> frame data

This confirms a **remotely reachable management interface**

---

## 2. Management Plane Discovery

### 2.1 Confirm REST service
```bash
curl -i http://127.0.0.1:2048/
```

Expected:
- HTTP 404
- `Server: TwistedWeb`

This proves the service is **alive**

<img width="1160" height="220" alt="2026-03-19_11-53" src="https://github.com/user-attachments/assets/8067c4ad-24af-4f2e-b405-18c42a0ab3c7" />


---

### 2.2 Enumerate API root
```bash
curl -i http://127.0.0.1:2048/api/
```

Expected output (critical finding):
```text
GET, DELETE /api/service-instances
GET, DELETE, POST, PATCH /api/service-instances/<string:si>
GET /api/sle-config
GET, PATCH /api/sle-config/<string:param>
```


<img width="707" height="218" alt="2026-03-19_11-57" src="https://github.com/user-attachments/assets/99f6930e-b37b-4533-a9f3-1625b99673ca" />


### Finding
The server **advertises sensitive management endpoints and HTTP verbs without authentication**

This alone constitutes a **high-severity vulnerability**

---

## 3. Exploitation: Unauthorized Enumeration

### 3.1 Enumerate service instances
```bash
curl -s http://127.0.0.1:2048/api/service-instances/ | jq .
```

<img width="943" height="94" alt="2026-03-19_11-58" src="https://github.com/user-attachments/assets/a5839274-7272-4f26-8dc8-f4c13081a8d0" />


**Impact:**
- Attacker learns live operational state
- Service identifiers can be reused for destructive actions

---

### 3.2 Enumerate runtime configuration
```bash
curl -s http://127.0.0.1:2048/api/sle-config/ | jq .
```

<img width="883" height="329" alt="2026-03-19_11-59" src="https://github.com/user-attachments/assets/22753345-25fa-4db1-9449-5e6a99c40b92" />



| Parameter | Category | Exploitation Impact | Why It Matters |
|---------|--------|---------------------|---------------|
| `authentication-delay` | Auth / Timing | Expands authentication window; weakens replay and brute‑force resistance | Increases attacker tolerance for failed auth attempts |
| `transmit-queue-size` | Availability | Queue exhaustion → degraded or stalled telemetry | Enables low‑effort DoS without crashing the service |
| `startup-timer` | Availability | Forces repeated session initialization failures | Prevents stable link establishment |
| `allow-non-use-heartbeat` | Session Control | Keeps dead or hijacked sessions alive | Enables stealthy degradation instead of hard failure |
| `min-heartbeat` | Liveness | Premature disconnects under load | Causes false session death |
| `max-heartbeat` | Liveness | Masks dead links for extended periods | Delays operator detection |
| `min-deadfactor` | Reliability | Aggressive link termination | Amplifies minor disruptions |
| `max-deadfactor` | Reliability | Suppresses failure detection | Enables long‑lived degraded states |
| `max-trace-length` | Visibility | Log flooding or truncation | Reduces forensic visibility |
| `min-reporting-cycle` | Telemetry | Forces excessive reporting | Resource exhaustion |
| `max-reporting-cycle` | Telemetry | Suppresses reporting | Operational blind spots |
| `server-types` | Architecture | Reveals supported service roles | Aids targeted attack planning |
| `local-id` | Identity | Enables identity spoofing prep | Breaks trust assumptions |
| `local-password` | Credentials | High‑value credential target | Direct compromise of trust |
| `remote-peers` | Trust | Maps trusted endpoints | Enables impersonation or MITM planning |

---

## 4. Exploitation Capability Validation

You will **prove** modification is possible

### 4.1 Check allowed methods
```bash
curl -i -X OPTIONS http://127.0.0.1:2048/api/service-instances/test
```

<img width="1005" height="234" alt="2026-03-19_12-01" src="https://github.com/user-attachments/assets/36205c0f-74f6-4fa0-aae0-44ac26b6dc25" />


Expected:
- `Allow:` header including `POST`, `PATCH`, or `DELETE`

### 4.2 Interpretation
If modification verbs are allowed **without authentication**, then:

- Integrity is compromised
- Availability is compromised
- Full mission impact is possible

No further exploitation is required to prove risk.

---

## 5. Traffic capture:

```bash
sudo tcpdump -i lo -nn tcp port 2048 -w sle-management-abuse.pcap
```

- Then in another **terminal**, generate traffic:

```bash
curl http://127.0.0.1:2048/api/
```

```bash
curl http://127.0.0.1:2048/api/sle-config/
```

- Go back to the **tcpdump terminal** and press **Ctrl + c** to stop the capture, it will save it into `sle-management-abuse.pcap` under the same folder

- Open it:

```bash
wireshark sle-management-abuse.pcap
```

<img width="1837" height="1057" alt="image" src="https://github.com/user-attachments/assets/03cc85a0-69ab-42ec-910d-371d7c3a2bf6" />


***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/blueLabs/DefendingOdyssey/DefendingODYSSEYLab.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/TheDrift/TheDriftLab.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---
