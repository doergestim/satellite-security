![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)


# Lab 1 — The Intercept

**Scenario**: Your team is hired to test the downlink resilience of a startup's CubeSat, **ODYSSEY-1**
They claim the link is secure. You're given two *historical* baseband captures from a ground station test

> Goal: Intercept → Decode → Reverse → (Simulated) Command

## Learning Outcomes
- Practice SDR recon on *synthetic* captures (recognize FSK, estimate bitrate)
- Recover frames using a CCSDS-like sync word and custom framing
- Parse telemetry, extract hidden intel, and derive a simple auth scheme
- Craft a valid **command packet** and feed it to a **local uplink gateway** (no RF) to receive the final flag

## What You Get
- `assets/pass_01.iq` — complex float32, 48 kS/s baseband (cleaner)
- `assets/pass_02.iq` — complex float32, 48 kS/s baseband (noisier)
- `tools/generate_captures.py` — deterministically regenerates the captures
- `tools/sat_gateway.py` — local validator for your crafted uplink packet
- `assets/*.json` — samplerate + format metadata

---

## Setup ( Only if you are not using the VM )

1. Download the zip for this main folder from [Here](./Lab1.zip)

- Click the Download button

![](./download.png)


- Extract it

2. Optional: Regenerate captures
   ```bash
   python3 tools/generate_captures.py
   ```


### Flags
- FLAG1{...} — first telemetry
- FLAG2{...} — second pass ACK
- FLAG3{...} — returned by local gateway on valid command

# Start
### Part A - Get bits out
- Open [GNU Radio](/Tools%20and%20Frameworks/GNU_radio.md)
```bash
gnuradio-companion &
```

<img width="1920" height="1167" alt="image" src="https://github.com/user-attachments/assets/2ddbdd2b-51b3-4eb5-b4c4-4595666f81fa" />

- First things first let's input our file, that's under ``~/Desktop/Lab1/assets/pass_01.iq``

- To add nodes to the flow press ``Ctrl + f`` to open the search bar on the right

<img width="420" height="110" alt="image" src="https://github.com/user-attachments/assets/08cd9ea9-2eb4-416b-85ac-b1096503d54b" />

- Look for ``File Source`` and drag it into the flow

<img width="128" height="157" alt="image" src="https://github.com/user-attachments/assets/6b0cc087-9a43-451d-a7a6-c1de4c605532" />

- Double click it and at ``File`` put the path to the ``~/Desktop/Lab1/assets/pass_01.iq`` and the rest like in the image, then press **Apply** and **Ok**

<img width="604" height="531" alt="image" src="https://github.com/user-attachments/assets/52f14882-7662-429b-8a92-86f64d60303f" />

>[!NOTE]
>For each block we add don't forget the shortcut is `Ctrl + f`

- Now let's add a ``Throttle`` and connect the 2 nodes by dragging the **out** from `File source` to the **in** of `Throttle`

<img width="377" height="147" alt="image" src="https://github.com/user-attachments/assets/7afb082d-e2dd-4ca0-90a7-e91fb953a013" />

<br>

<img width="548" height="447" alt="image" src="https://github.com/user-attachments/assets/ef271371-dce1-42e7-a949-cb485108d97c" />

- Double click the ``samp_rate`` variable and edit the **Value** to **48000**, then again press **Apply** and **Ok**, as we do with every block when we edit it

<img width="109" height="86" alt="image" src="https://github.com/user-attachments/assets/39e8e7cf-ff0f-437f-bcec-d22e98b82a1c" />

- Add a ``QT GUI Frequency Sink`` and connect it to the ``Throttle``

<img width="604" height="531" alt="image" src="https://github.com/user-attachments/assets/aec248cc-eb4b-4af3-a534-40de0dd6d4ce" />

<img width="681" height="180" alt="image" src="https://github.com/user-attachments/assets/57383e01-09d2-420d-aad2-fa8df8cf9527" />

- Double click the `Options` block on the top-left, in the **Id** field write **Lab1**, and under **Generate Options** select **QT GUI**

<img width="598" height="533" alt="image" src="https://github.com/user-attachments/assets/d37cf9ce-5ab0-42fc-b9f2-f0d10c1eb968" />


- Run the flow by pressing ``F6``, you will be prompted to save the file, let's save it with the name **Lab1_GNU.grc** on **Desktop**

<img width="828" height="50" alt="image" src="https://github.com/user-attachments/assets/4c5d6795-700a-44c0-b0a5-73c4673fca6f" />

- You’re looking at raw baseband. Two energy blobs near ±2 kHz = 2-FSK

<img width="410" height="374" alt="image" src="https://github.com/user-attachments/assets/bde68567-0911-4910-8942-106f2afbcd20" />

- Close that and let's go on, we are going to keep that for reference and testing purposes

- Add a ``Quadrature Demod`` and connect it to the ``Throttle``, this is a **Frequency discriminator** (turn FSK into a 1-D float), FSK encodes data as instantaneous frequency. Quadrature Demod converts frequency shifts into a float that swings high/low for 1/0

<img width="598" height="533" alt="image" src="https://github.com/user-attachments/assets/c92d319c-f07a-4e1e-8cd7-e9b58c0f060c" />


- Now let's add those variables and some more, search for ``Variable`` and drag into the flow for each one, we need 3 more

<img width="550" height="160" alt="image" src="https://github.com/user-attachments/assets/2b887f6a-4718-4287-a40e-63a069c8d060" />

<br>

<img width="550" height="160" alt="image" src="https://github.com/user-attachments/assets/1be2275b-c6df-4fa8-a1e3-0fad05235a53" />

<br>

<img width="550" height="160" alt="image" src="https://github.com/user-attachments/assets/d2de6b95-2d09-4af8-b056-e2f8034e90d3" />

<br>

<img width="398" height="90" alt="image" src="https://github.com/user-attachments/assets/5a5e9617-56ca-474d-9f03-33a198f2d0da" />


- Add a ``Low Pass Filter`` and connect it to the ``Quadrature Demod``, it removes high-frequency noise so the clock recovery locks faster

<img width="603" height="539" alt="image" src="https://github.com/user-attachments/assets/1605b263-c539-439f-902a-82ccc945d6e5" />

- Add a ``QT GUI Time Sink`` and connect it to the ``Low Pass Filter`` and set it to **float**

<img width="603" height="539" alt="image" src="https://github.com/user-attachments/assets/763cfd8c-91d1-4366-987a-6e552d22e605" />

<img width="1149" height="434" alt="image" src="https://github.com/user-attachments/assets/2f31fbef-ded6-4e09-ab65-1ca5934442d7" />


- Run it again by pressing ``F6`` to visualize this

<img width="595" height="584" alt="image" src="https://github.com/user-attachments/assets/c3d8f4f6-5456-459a-9d0b-9b99a608d8fd" />

- Add a ``Clock Recovery MM`` and connect it to the ``Low Pass Filter``, our float stream is oversampled at 48 kS/s. This block finds the optimal sample per symbol every 40 samples to align to bit boundaries

<img width="602" height="536" alt="image" src="https://github.com/user-attachments/assets/3bd1d575-b145-4e39-91dc-a80607990b4e" />

- Add a ``Binary Slicer`` and connect it to the ``Clock Recovery MM`` and then a ``UChar To Float`` and connect it to the ``Binary Slicer``, the binary slicer converts each symbol into a byte 0x00 or 0x01

- Add a ``QT GUI Time Sink`` and connect it to the ``UChar To Float``, set that to float

<img width="602" height="536" alt="image" src="https://github.com/user-attachments/assets/8fb45293-404b-4a67-bdc0-503e526fe79a" />

<br>

<img width="1094" height="527" alt="image" src="https://github.com/user-attachments/assets/012f2ea2-ba62-467d-9356-d7209617585a" />

- Run again with ``F6`` to see what we got

<img width="661" height="837" alt="image" src="https://github.com/user-attachments/assets/7e95abf6-2a76-47b2-a2b8-9f1348836043" />

- Add a ``Add Const`` and connect it to the ``Binary Slicer``, we will add 48 to convert it to **ASCII**

<img width="600" height="536" alt="image" src="https://github.com/user-attachments/assets/c8b0df5d-cee5-4a10-a264-7b49167ef67e" />

- Add a ``File Sink`` and connect it to the ``Add Const``, save the file into ``~/Desktop/Lab1/assets/pass_01.bits``

<img width="600" height="536" alt="image" src="https://github.com/user-attachments/assets/faddda91-c6a7-46f4-8de7-4c630e596bf2" />

- Now you can run the flow for 5-10 seconds and check the file we created, you should get something like this

```bash
cat ~/Desktop/Lab1/assets/pass_01.bits
```

```
1000101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101000110101100111111111100000111010000000100000000000000010000000100000000100101010111101100100010011100110110000101110100001000100011101000100010010011110100010001011001010100110101001101000101010110010010110100110001001000100010110000100010011000110111001100100010001110100010001001001111010001000101100101010011010110010101001100100010001011000010001001100101011100000110111101100011011010000010001000111010001100010011011100110001001100110011001100110111001100010011001100110011001101110010110000100010011000100110000101110100011101000110010101110010011110010010001000111010001101110010111000110110001011000010001001110100011001010110110101110000001000100011101000110010001100010010111000110100001011000010001001100001011100110110110100100010001110100010001000110001010000010100001101000110010001100100001100110001010001000010001000101100001000100110111001101111011101000110010100100010001110100010001001000001010100110100110100100000011100000111001001100101011100110110010101101110011101000010000001100101011101100110010101110010011110010010000001100110011100100110000101101101011001010010001000101100001000100110100001101001011011100111010000100010001110100010001001100011011011110110110001101111011100100011111100100000010000100100110001010101010001010010001001111101101011101011001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000011010110011111111110000011101000000010000000000000010000000100000000010010001011110110010001001110011011101010110001001110011011110010111001100100010001110100111101100100010011000010110010001100011011100110010001000111010001000100110111101101011001000100010110000100010011000110110111101101101011011010111001100100010001110100010001001101111011010110010001000101100001000100111000001100001011110010110110001101111011000010110010000100010001110100010001001101001011001000110110001100101001000100111110100101100001000100111000001101111011100110010001000111010011110110010001001101100011000010111010000100010001110100011001100111000001011100011001000110100001101100010110000100010011011000110111101101110001000100011101000110010001100010010111000110111001100110011010100101100001000100110000101101100011101000101111101101011011011010010001000111010001101010011001100110000011111010010110000100010011100110111010001100001011101000111010101110011001000100011101000100010011011100110111101101101011010010110111001100001011011000011101100100000010001100100110001000001010001110011000101111011011001000110111101110111011011100110110001101001011011100110101101011111011001000110010101100011011011110110010001100101011001000111110100100010011111011111001110110110000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000110101100111111111100000111010000000100000000000000110000000100000000011111010111101100100010011100110110000101110100001000100011101000100010010011110100010001011001010100110101001101000101010110010010110100110001001000100010110000100010011000110111001100100010001110100010001001001111010001000101100101010011010110010101001100100010001011000010001001100101011100000110111101100011011010000010001000111010001100010011011100110001001100110011001100110111001100010011001100110011001101110010110000100010011000100110000101110100011101000110010101110010011110010010001000111010001101110010111000110101001011000010001001110100011001010110110101110000001000100011
```

### Part B - Frame sync + parsing
- For this lab, the Sync word is **0x1ACFFC1D**, a very common one, which in binary is **00011010110011111111110000011101**

- Let's add a ``Correlate Access Code - Tag`` and connect it to the ``Binary Slicer``

<img width="600" height="536" alt="image" src="https://github.com/user-attachments/assets/730f486d-b076-4dcd-b831-98b4cb62b9bd" />

- To make sure we are getting hits, add a ``Tag Debug`` and connect it to the ``Correlate Access Code - Tag``

<img width="600" height="536" alt="image" src="https://github.com/user-attachments/assets/6e9f7888-85af-470e-807c-6819ac15dd78" />

- Run the flow, you should see hits in the debug section in the bottom-left

<img width="414" height="284" alt="image" src="https://github.com/user-attachments/assets/8c99b9c5-2c40-43aa-b5c4-55ffa0cfd946" />

- Add a ``Tagged Stream Align`` and connect it to the ``Correlate Access Code - Tag``

<img width="551" height="451" alt="image" src="https://github.com/user-attachments/assets/7bddfa66-0200-4836-a1e3-b03b2a185136" />

- Add  a ``Repack Bits`` and connect it to the ``Tagged Stream Align``

<img width="551" height="451" alt="image" src="https://github.com/user-attachments/assets/78540910-1542-4d52-a126-7e4b8e59b54b" />

- Add a ``File Sink`` and connect it to the ``Repack Bits``, save it into ``~/Desktop/Lab1/assets/pass_01_BPF.txt``

<img width="549" height="445" alt="image" src="https://github.com/user-attachments/assets/712099df-7632-4d1e-8fe8-62ad97df21f7" />

- This is how the final flow should look:

<img width="1473" height="700" alt="image" src="https://github.com/user-attachments/assets/d7c39993-a93a-468b-a51d-e3844b9ced90" />

- Now let it Run for 5-10 seconds then stop

```bash
cd ~/Desktop/Lab1
```

- Run
```bash
xxd -d assets/pass_01_BPF.txt
```

- You should see information flowing out

<img width="714" height="1096" alt="image" src="https://github.com/user-attachments/assets/bd5d3887-7ed3-4e32-96e1-00bc2608e6b6" />

- Hooray!! It works! We get valuable information:

1. The first flag: **FLAG1{downlink_decoded}**
2. The **sat**: **ODYSSEY-1**
3. The **epoch**: **1713371337**

- We will use these to make a payload to get the last flag, run the same logic for ``pass_02.iq``, basically just change the paths

<img width="618" height="268" alt="image" src="https://github.com/user-attachments/assets/892babed-4127-4c7f-b4dd-e3f237bed2c0" />

- We get an **ACK** response with out second flag: **FLAG2{protocol_reversed}**

- For the final flag, we will build this payload script using the information from ``pass_01.iq`` and save it as ``uplink_craft.py`` in the main directory

```bash
nano uplink_craft.py
```

- Copy-Paste is your friend

```bash
import json, struct, hashlib, binascii
SYNC=0x1ACFFC1D
def crc16_ccitt(b, init=0xFFFF): return binascii.crc_hqx(b, init)

def pkt(ver, seq, ptype, payload):
    pay=json.dumps(payload,separators=(',',':')).encode()
    hdr=struct.pack('>IBHBH', SYNC, ver, seq, ptype, len(pay))
    crc_in=struct.pack('>BHBH', ver, seq, ptype, len(pay))+pay
    crc=crc16_ccitt(crc_in)
    return hdr+pay+struct.pack('>H',crc)

epoch=1713371337
sat="ODYSSEY-1"
auth=hashlib.sha1(f"{epoch}{sat}-BLUE".encode()).hexdigest()[:8]

uplink=pkt(1, 4242, 0x03, {"cmd":"SET_MODE","mode":"CAL","epoch":epoch,"sat":sat,"auth":auth})
open('uplink.bin','wb').write(uplink)
print("uplink.bin written")
```

- To save and exit do `Ctrl + x` and `y` and `Enter`

- Run it

```bash
python3 uplink_craft.py
```

- Now let's feed the local gateway

```bash
python3 tools/sat_gateway.py uplink.bin
```
<img width="538" height="22" alt="image" src="https://github.com/user-attachments/assets/061850ba-c8eb-4c49-98ba-5a6af32375a4" />

- Now we got our 3rd and final flag: **FLAG3{uplink_forged_locally}**












