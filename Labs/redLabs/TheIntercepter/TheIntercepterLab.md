![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)


# Lab 1 - The Intercept

**Scenario**: Your team is hired to test the downlink resilience of a startup's CubeSat, **ODYSSEY-1**

They claim the link is secure. You're given two *historical* baseband captures from a ground station test

> Goal: Intercept -> Decode -> Reverse -> (Simulated) Command

## Learning Outcomes
- Practice SDR recon on *synthetic* captures (recognize FSK, estimate bitrate)
- Recover frames using a CCSDS-like sync word and custom framing
- Parse telemetry, extract hidden intel, and derive a simple auth scheme
- Craft a valid **command packet** and feed it to a **local uplink gateway** (no RF) to receive the final flag

## What You Get
- `assets/pass_01.iq` - complex float32, 48 kS/s baseband (cleaner)
- `assets/pass_02.iq` - complex float32, 48 kS/s baseband (noisier)
- `tools/generate_captures.py` - deterministically regenerates the captures
- `tools/sat_gateway.py` - local validator for your crafted uplink packet
- `assets/*.json` - samplerate + format metadata


### Flags
- FLAG1{...} - first telemetry
- FLAG2{...} - second pass ACK
- FLAG3{...} - returned by local gateway on valid command

# Start
### Part A - Get bits out

- Open a terminal

- Open [GNU Radio Companion](/Tools%20and%20Frameworks/GNU_radio.md)
```bash
gnuradio-companion &
```

![](/Assets/RLab1/Lab1-1.png)

- First things first, let's input data from our file that's under ``/home/satuser/Desktop/TheIntercepter/assets/pass_01.iq``

- To add blocks to the flow press ``Ctrl + f`` to open the search bar on the right

![](/Assets/RLab1/Lab1-2.png)

- Look for ``File Source`` and drag it into the flow. We are using the ``File Source`` block because the signal is already captured

The `File Source` just replays the **IQ samples** so we can analyze them reliably without **live RF**

![](/Assets/RLab1/Lab1-3.png)

- Double click the ``File Source`` block to open the configuration settings. In the ``File`` field, put the path to the ``/home/ubuntu/Desktop/TheIntercepter/assets/pass_01.iq`` capture file. Fill out the rest of the fields to match what you see in the image. Press **Apply** and then **Ok**


<img src="/Assets/RLab1/Lab1-4.png" width="700">

>[!IMPORTANT]
>For each block we add, don't forget that the shortcut to search for blocks is `Ctrl + f`

- Now let's add a ``Throttle`` block and connect the 2 blocks by dragging the **out** from `File Source` to the **in** of `Throttle`

>[!NOTE]
>Why do we use ``Throttle``?
>``Throttle`` limits how fast samples flow through the graph
>
>Without a ``Throttle``, GNU Radio will run as fast as your CPU allows, spike usage, and make the GUI unusable when there’s no real hardware clock to otherwise limit the input rate

![](/Assets/RLab1/Lab1-5.png)

<br>

![](/Assets/RLab1/Lab1-6.png)

- Double click the ``samp_rate`` variable block at the top of the graph and edit the **Value** to **48000**. Press **Apply** and then **Ok**, as we do with every block when we edit it

![](/Assets/RLab1/Lab1-7.png)

>[!NOTE]
>What is **Sample rate**?
>It’s how many **samples per second** the signal contains
>Here, **48 kS/s** tells every block how fast the signal was captured so **timing**, **filters**, and **symbol recovery** work correctly


- Add a ``QT GUI Frequency Sink`` block and connect it to the ``Throttle``

>[!NOTE]
>What is `QT GUI Frequency Sink`?
>It shows the **signal** in the **frequency domain**.
>We use it to see where energy sits in the **spectrum** so we can quickly identify things like **bandwidth**, **offsets**, and **FSK (Frequency-Shift Keying) tones**

![](/Assets/RLab1/Lab1-8.png)

![](/Assets/RLab1/Lab1-9.png)

- Double click the `Options` block on the top-left, in the **Id** field write **Lab1**, and under **Generate Options** select **QT GUI**

![](/Assets/RLab1/Lab1-10.png)


- Run the flow by pressing ``F6``, you will be prompted to save the file, let's save it with the name **Lab1_GNU.grc** on **Desktop**, you might also get a warning, ignore it!

>[!IMPORTANT]
>Here is a [Checkpoint File](/Assets/RLab1/TheIntercept_1.grc)
>
>If you need to use this, just **download** it into your **VM** and **double click** on it

![](/Assets/RLab1/Lab1-11.png)

- You’re looking at **raw baseband**. You will notice two energy blobs near **±2 kHz**, indicating that this signal was likely encoded using **Binary FSK** a.k.a. **2-FSK** or **2FSK**

![](/Assets/RLab1/Lab1-12.png)

- Close that and let's go on, we are going to keep that for reference and testing purposes

- Add a ``Quadrature Demod`` block and connect it to the ``Throttle``, this is a **Frequency discriminator** that can convert **FSK** into a **1-D float** representation. **FSK** encodes data as instantaneous frequency. ``Quadrature Demod`` converts frequency shifts into a float that swings **high**/**low** for **1**/**0**, respectively. For the sake of simplicity, open the settings for the ``Quadrature Demod`` block and change **fsk_deviation_hz** to **fdev**

![](/Assets/RLab1/Lab1-13.png)


- The ``Quadrature Demod`` block references the **fdev** variable, but we haven't added that variable yet to the graph. Let's add that variable plus a few more. Search for ``Variable`` and drag a total of three ``Variable`` blocks onto the graph.

![](/Assets/RLab1/Lab1-14.png)

<br>

![](/Assets/RLab1/Lab1-15.png)

<br>

![](/Assets/RLab1/Lab1-16.png)

<br>

![](/Assets/RLab1/Lab1-17.png)

What do they mean?

1. **sym_rate** (**1200**): The symbol rate - how many **bits per second** the **satellite** is transmitting

2. **sps** (**samp_rate / sym_rate**): The samples per second - how many **samples** represent one **symbol**. This tells **clock recovery** where bit boundaries are

3. **fdev** (**2000**): **Frequency deviation** (in **Hz**) of the **FSK tones** - how far the signal shifts for a **0** vs **1**


- Add a ``Low Pass Filter`` block and connect it to the ``Quadrature Demod`` block. The ``Low Pass Filter`` block removes high-frequency noise so that the clock recovery locks faster. Open the settings for the ``Low Pass Filter`` block and change them to match what is shown in the image below

![](/Assets/RLab1/Lab1-18.png)

- Add a ``QT GUI Time Sink`` block, connect it to the ``Low Pass Filter`` block, and change the **Type** setting to **Float**

![](/Assets/RLab1/Lab1-19.png)

![](/Assets/RLab1/Lab1-20.png)


- Run it again by pressing ``F6`` to visualize this

![](/Assets/RLab1/Lab1-21.png)

>[!IMPORTANT]
>Here is a [Checkpoint File](/Assets/RLab1/TheIntercept_2.grc)
>
>If you need to use this, just **download** it into your **VM** and **double click** on it

- Add a ``Clock Recovery MM`` block and connect it to the ``Low Pass Filter`` block. Our float stream is oversampled at 48 kS/s. The ``Clock Recovery MM`` block finds the optimal sample per symbol every 40 samples to align to bit boundaries. Open the ``Clock Recovery MM`` block and change the settings to match what is seen in the image below

![](/Assets/RLab1/Lab1-22.png)

- Add a ``Binary Slicer`` block and connect it to the ``Clock Recovery MM`` block. Then add a ``UChar To Float`` block and connect it to the ``Binary Slicer`` block. The ``Binary Slicer`` converts each symbol into a byte 0x00 or 0x01

- Add a ``QT GUI Time Sink`` block and connect it to the ``UChar To Float`` block. Open the settings for the ``QT GUI Time Sink`` block and set the **Type** setting to **Float**

![](/Assets/RLab1/Lab1-23.png)

<br>

![](/Assets/RLab1/Lab1-24.png)

- Run again with ``F6`` to see what we got

![](/Assets/RLab1/Lab1-25.png)

>[!IMPORTANT]
>Here is a [Checkpoint File](/Assets/RLab1/TheIntercept_3.grc)
>
>If you need to use this, just **download** it into your **VM** and **double click** on it

- Add a ``Add Const`` block and connect it to the ``Binary Slicer`` block. We will add 48 to convert the ``Binary Slicer`` output to **ASCII**. Open the settings for the ``Add Const`` block and change them match what is seen in the image below

![](/Assets/RLab1/Lab1-26.png)

- Add a ``File Sink`` block and connect it to the ``Add Const`` block. The `File Sink` block reads the **output** from the ``Add Const`` block and saves it into a **file**. Open the settings for the ``File Sink`` block and change them to match what is seen in the image below


<img src="/Assets/RLab1/Lab1-27.png" width="700">

- Now you can run the flow for 5-10 seconds and check the file that was created. You should get something like the following

```bash
cat /home/ubuntu/Desktop/TheIntercepter/assets/pass_01.bits
```

```
1000101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101010101000110101100111111111100000111010000000100000000000000010000000100000000100101010111101100100010011100110110000101110100001000100011101000100010010011110100010001011001010100110101001101000101010110010010110100110001001000100010110000100010011000110111001100100010001110100010001001001111010001000101100101010011010110010101001100100010001011000010001001100101011100000110111101100011011010000010001000111010001100010011011100110001001100110011001100110111001100010011001100110011001101110010110000100010011000100110000101110100011101000110010101110010011110010010001000111010001101110010111000110110001011000010001001110100011001010110110101110000001000100011101000110010001100010010111000110100001011000010001001100001011100110110110100100010001110100010001000110001010000010100001101000110010001100100001100110001010001000010001000101100001000100110111001101111011101000110010100100010001110100010001001000001010100110100110100100000011100000111001001100101011100110110010101101110011101000010000001100101011101100110010101110010011110010010000001100110011100100110000101101101011001010010001000101100001000100110100001101001011011100111010000100010001110100010001001100011011011110110110001101111011100100011111100100000010000100100110001010101010001010010001001111101101011101011001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000011010110011111111110000011101000000010000000000000010000000100000000010010001011110110010001001110011011101010110001001110011011110010111001100100010001110100111101100100010011000010110010001100011011100110010001000111010001000100110111101101011001000100010110000100010011000110110111101101101011011010111001100100010001110100010001001101111011010110010001000101100001000100111000001100001011110010110110001101111011000010110010000100010001110100010001001101001011001000110110001100101001000100111110100101100001000100111000001101111011100110010001000111010011110110010001001101100011000010111010000100010001110100011001100111000001011100011001000110100001101100010110000100010011011000110111101101110001000100011101000110010001100010010111000110111001100110011010100101100001000100110000101101100011101000101111101101011011011010010001000111010001101010011001100110000011111010010110000100010011100110111010001100001011101000111010101110011001000100011101000100010011011100110111101101101011010010110111001100001011011000011101100100000010001100100110001000001010001110011000101111011011001000110111101110111011011100110110001101001011011100110101101011111011001000110010101100011011011110110010001100101011001000111110100100010011111011111001110110110000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000110101100111111111100000111010000000100000000000000110000000100000000011111010111101100100010011100110110000101110100001000100011101000100010010011110100010001011001010100110101001101000101010110010010110100110001001000100010110000100010011000110111001100100010001110100010001001001111010001000101100101010011010110010101001100100010001011000010001001100101011100000110111101100011011010000010001000111010001100010011011100110001001100110011001100110111001100010011001100110011001101110010110000100010011000100110000101110100011101000110010101110010011110010010001000111010001101110010111000110101001011000010001001110100011001010110110101110000001000100011
```

### Part B - Frame sync + parsing
- Let's turn those 0's and 1's into something more readable. We first need to determine where pieces of information, known as **frames**, begin so that we can perform that conversion. The beginning of **frames** is signified with a **sync word**. For this lab, the **sync word** is **0x1ACFFC1D**, a very common one, which in binary is **00011010110011111111110000011101**

- Let's add a ``Correlate Access Code - Tag`` block and connect it to the ``Binary Slicer`` block. The ``Correlate Access Code - Tag`` block searches the **bitstream** for a known **sync word**. When the block finds that **sync word**, it tags the stream so that downstream blocks know **“a frame starts here”**. Change the settings in the ``Correlate Access Code - Tag`` block to match what is seen in the image below

![](/Assets/RLab1/Lab1-28.png)

- To make sure we are getting hits, add a ``Tag Debug`` block and connect it to the ``Correlate Access Code - Tag`` block. The ``Tag Debug`` block just shows you the tags in the **stream**. Open the ``Tag Debug`` block settings and change the **Type** to **Byte**

![](/Assets/RLab1/Lab1-29.png)

- Run the flow, you should see hits in the debug section in the bottom-left

![](/Assets/RLab1/Lab1-30.png)

>[!IMPORTANT]
>Here is a [Checkpoint File](/Assets/RLab1/TheIntercept_4.grc)
>
>If you need to use this, just **download** it into your **VM** and **double click** on it

- Add a ``Tagged Stream Align`` block and connect it to the ``Correlate Access Code - Tag`` block. The ``Tagged Stream Align`` block realigns the **stream** so data starts exactly at the **tagged frame boundary**. Change the settings in the ``Tagged Stream Align`` block to match those seen in the image below

![](/Assets/RLab1/Lab1-31.png)

- Add a ``Repack Bits`` block and connect it to the ``Tagged Stream Align`` block. The ``Repack Bits`` block groups **individual bits** into **bytes**. After **frame sync**, we need real **bytes** so the data can be dumped, parsed, and read as a **packet**. Open the settings for the ``Repack Bits`` block and ensure they match those seen in the image below

![](/Assets/RLab1/Lab1-32.png)

- Add a ``File Sink`` block, connect it to the ``Repack Bits`` block, and change the settings to those seen in the image below

![](/Assets/RLab1/Lab1-33.png)

- This is how the **final flow** should look:

![](/Assets/RLab1/Lab1-34.png)

>[!IMPORTANT]
>Here is a [Checkpoint File](/Assets/RLab1/TheIntercept_5.grc)
>
>If you need to use this, just **download** it into your **VM** and **double click** on it

- Now let it Run for 5-10 seconds then stop

```bash
cd ~/Desktop/TheIntercepter
```

- Run
```bash
xxd assets/pass_01_BPF.bits
```

- You should see information flowing out

![](/Assets/RLab1/Lab1-35.png)

- Hooray!! It works! We get valuable information:

1. The first flag: **FLAG1{downlink_decoded}**
2. The **sat**: **ODYSSEY-1**
3. The **epoch**: **1713371337**

- We will use these to make a payload to get the last flag, run the same logic for ``pass_02.iq``, basically just change the paths

![](/Assets/RLab1/Lab1-36.png)

- We get an **ACK** response with our second flag: **FLAG2{protocol_reversed}**

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
![](/Assets/RLab1/Lab1-37.png)

- Now we got our 3rd and final flag: **FLAG3{uplink_forged_locally}**





***                                                                 
<b><i>Continuing the course? </br>[Next Lab](/Labs/redLabs/TheDenial/TheDenialLab.md)</i></b>

<b><i>Looking for a different lab? </br>[Lab Directory](/navigation.md)</i></b>

***Finished with the Labs?***

Please be sure to destroy the lab environment!

[Click here for instructions on how to destroy the Lab Environment](/labdestruction.md)

---

> Created By Turcu Știolică Alexandru - Black Hills Information Security
