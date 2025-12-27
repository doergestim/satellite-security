![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 6: Exploiting an Unauthenticated SLE Management Plane

## Context
This lab demonstrates a **realistic vulnerability class**:  
**unauthenticated exposure of a management/control plane** in a protocol-heavy system.

You are assessing a **VisionSpace SLE Provider** deployed with insecure defaults.  
Your goal is to **enumerate, validate impact, and remediate** the issue using professional techniques.

This is **not** a beginner lab. You are expected to understand TCP/IP, REST, and basic threat modeling.

---

## Assumptions
- You are already in the `sle-provider` directory
- Docker is installed and working
- The provider is started with:

```bash
sudo docker compose up --build -d
```

---

## 1. Establish Baseline Exposure

### 1.1 Identify exposed ports
```bash
sudo docker ps --format "table {{.Names}}\t{{.Ports}}"
```

Expected exposure:
- TCP 2048 → Management REST API
- TCP 55529 → SLE user port
- UDP 16887–16888 → frame data

This confirms a **remotely reachable management interface**.

---

## 2. Management Plane Discovery

### 2.1 Confirm REST service
```bash
curl -i http://127.0.0.1:2048/
```

Expected:
- HTTP 404
- `Server: TwistedWeb`

This proves the service is alive.

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

### Finding
The server **advertises sensitive management endpoints and HTTP verbs without authentication**.

This alone constitutes a **high-severity vulnerability**.

---

## 3. Exploitation: Unauthorized Enumeration

> The following actions are **read-only** and safe.

### 3.1 Enumerate service instances
```bash
curl -s http://127.0.0.1:2048/api/service-instances | jq .
```

**Impact:**
- Attacker learns live operational state
- Service identifiers can be reused for destructive actions

---

### 3.2 Enumerate runtime configuration
```bash
curl -s http://127.0.0.1:2048/api/sle-config | jq .
```

**Impact:**
- Disclosure of antenna IDs, ports, routing parameters
- Enables targeted sabotage or stealthy interference

This is a **confidentiality breach**.

---

## 4. Exploitation Capability Validation (Non-Destructive)

You will **prove** modification is possible without executing it.

### 4.1 Check allowed methods
```bash
curl -i -X OPTIONS http://127.0.0.1:2048/api/service-instances/test
```

Expected:
- `Allow:` header including `POST`, `PATCH`, or `DELETE`

### 4.2 Interpretation
If modification verbs are allowed **without authentication**, then:

- Integrity is compromised
- Availability is compromised
- Full mission impact is possible

No further exploitation is required to prove risk.

---

## 5. Threat Modeling (Required Exercise)

Answer the following:

- What happens if `/api/service-instances/<si>` is deleted?
- What happens if frame routing parameters are modified?
- Why is this worse than a typical web app vulnerability?

Document answers.

---

## 6. Evidence Collection

Collect the following artifacts:

```bash
curl -i http://127.0.0.1:2048/api/
curl -s http://127.0.0.1:2048/api/service-instances
curl -s http://127.0.0.1:2048/api/sle-config
```

Optional traffic capture:
```bash
sudo tcpdump -i any -nn tcp port 2048 -w sle-management-abuse.pcap
```

---

## 7. Root Cause Analysis

Inspect `docker-compose.yml`:

```yaml
ports:
  - "2048:2048/tcp"
```

### Root cause
- Management plane bound to **0.0.0.0**
- No authentication enabled
- Intended for trusted lab use, deployed unsafely

This is a **deployment vulnerability**, not a code bug.

---

## 8. Remediation

### 8.1 Restrict management plane exposure
Modify:
```yaml
- "2048:2048/tcp"
```

To:
```yaml
- "127.0.0.1:2048:2048/tcp"
```

Restart:
```bash
sudo docker compose down
sudo docker compose up --build -d
```

---

## 9. Verification (Post-Fix)

### 9.1 Attacker POV (remote)
```bash
nc -vz <VM_IP> 2048
```
Expected: **connection refused**

### 9.2 Admin POV (local)
```bash
curl http://127.0.0.1:2048/api/
```
Expected: still accessible locally

---

## 10. Final Assessment

### Severity: **High**
- Unauthenticated management interface
- Full CRUD operations exposed
- Mission-impacting consequences

***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/blueLabs/BlueLab/BlueLab.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/Lab5/The_Drift.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---
