# Lab 2 — The Denial

**Mission:**  
As red team, your task is to **disrupt ODYSSEY-1’s ground communications**.  
You will:  
1. Degrade the downlink (simulate jamming).  
2. Confuse operators with replay attacks.  
3. Lock operators out by overwhelming their weak ground station service.  

---

## Requirements
**Having done** [Lab 1](../Lab1/TheInterceptLab.md)

Download the zip for this main folder from [Here](./Lab2_The_Denial_Kit.zip)

- Click the Download button

<img width="330" height="177" alt="image" src="https://github.com/user-attachments/assets/df15f9ee-985a-4f6a-af65-32698e1aa337" />


- Extract it

Make sure you have these installed:

- **GNU Radio Companion (GRC)** (≥ 3.8 recommended)  
- **Python 3.9+** with:  
  - `numpy`  
  - `requests` (for some optional HTTP tasks)  
- **Docker + Docker Compose** (to run the ground station)  
- **curl** (for HTTP attacks)  
- **xargs**, **seq**, **watch**, and optionally **jq** (all standard on most Linux distros; `jq` is useful for JSON pretty-printing)

