![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)


# The Slow Exfil CTF - Writeup

---

## Answers

**Q1. What is the IP address of the host performing the exfiltration?**

`192.168.1.50`

> [!NOTE]
> You might note be able to get this from the beggining, these first questions are easier to answer afterwards





> **Step 1 - Find all hosts making HTTP requests.** Apply:
> ```
> http.request
> ```
> Look at the `ip.src` column. Three source IPs appear: `192.168.1.20`,
> `192.168.1.50`, and `192.168.1.77`.
>
> **Step 2 - Identify which hosts look suspicious.** Filter to each host and
> inspect their User-Agent headers. `192.168.1.20` uses realistic browser strings
> (Chrome, Firefox, curl) - normal. Both `192.168.1.50` and `192.168.1.77`
> carry a `probe/NNN;` substring in every User-Agent. Both are candidates.
>
> **Step 3 - Check the IO Graph for beaconing.** Go to **Statistics -> IO Graph**.
> Both suspicious hosts show periodic spikes, but check the destination for each:
> ```
> http.request && ip.src == 192.168.1.50
> http.request && ip.src == 192.168.1.77
> ```
> `192.168.1.50` sends all its requests to the same external IP (`93.184.216.34`).
> `192.168.1.77` sends all its requests to a different IP (`203.0.113.10`).
>
> **Step 4 - Check whether the encoded values make sense.** For each host,
> read the numbers from the `probe/NNN;` fields in order and look them up in
> the ASCII table. The values from `192.168.1.50` (79, 68, 89, 83...) all fall
> in the printable ASCII range and spell readable text. The values from
> `192.168.1.77` (132, 155, 178...) are all above 127 - outside standard
> printable ASCII - and produce garbage. The host encoding coherent, readable
> ASCII is the exfiltrator.

---

**Q2. What destination IP address receives the exfiltrated data?**

`93.184.216.34`

> **Step 1 - Filter to the attacker's requests:**
> ```
> http.request && ip.src == 192.168.1.50
> ```
>
> **Step 2 - Read the `ip.dst` column.** All 11 requests from `192.168.1.50`
> go to the same destination.
>
> **Step 3 - Confirm this differs from the decoy destination.** The decoy
> host (`192.168.1.77`) sends its requests to `203.0.113.10`. Using the wrong
> destination when reconstructing the message gives you garbage values from
> non-printable decimal numbers. The real exfil target is `93.184.216.34`.

---

**Q3. How many HTTP requests does the attacker send as part of the exfiltration?**

`11`

> **Step 1 - Filter to attacker requests going to the correct destination:**
> ```
> http.request && ip.src == 192.168.1.50 && ip.dst == 93.184.216.34
> ```
>
> **Step 2 - Read the packet count in the Wireshark status bar at the
> bottom of the window.** It reads `11 displayed`.
>
> **Step 3 - Cross-check:** this equals the length of the exfiltrated message.
> One request encodes one character.

---

**Q4. Which HTTP header field carries the hidden data?**

`User-Agent`

> **Step 1 - Filter to any single attacker request:**
> ```
> http.request && ip.src == 192.168.1.50
> ```
> Click the first result.
>
> **Step 2 - Expand `Hypertext Transfer Protocol` in the packet detail pane.**
> Compare the headers against a normal request from `192.168.1.20`. All standard
> fields (`Host`, `Accept`, `Connection`) are identical. The only anomalous field
> is `User-Agent`.
>
> **Step 3 - Read the User-Agent value.** It contains the substring
> `probe/79;` - a decimal number embedded in an otherwise plausible
> User-Agent string:
> ```
> Mozilla/5.0 (compatible; probe/79; +http://update.local)
> ```
> Normal User-Agents from `192.168.1.20` contain browser and OS identifiers
> with no numeric `probe/` substring.

---

**Q5. What encoding scheme is used to embed data in that header field?**

`Decimal ASCII - each request encodes one character as its decimal ASCII value`

> **Step 1 - Extract the numeric values from several requests.** Apply:
> ```
> http.request && ip.src == 192.168.1.50 && ip.dst == 93.184.216.34
> ```
> For each row, read the number between `probe/` and `;` in the User-Agent.
> The first few values are: `79`, `68`, `89`, `83`, `83`.
>
> **Step 2 - Look up each value in the ASCII table:**
> ```
> 79 -> O
> 68 -> D
> 89 -> Y
> 83 -> S
> 83 -> S
> ```
> Each value is a printable ASCII character code in decimal.
>
> **Step 3 - Rule out other encodings.** The values are not hex (no `0x` prefix,
> and values like `89` are unambiguous as decimal). They are not base64 (single
> integers, not strings). Straight decimal ASCII is the only encoding that maps
> all 11 values to printable characters.

---

**Q6. What substring pattern in the User-Agent header identifies an exfil request
and distinguishes it from normal traffic?**

`` `probe/` `` (followed by a decimal integer and semicolon)

> **Step 1 - Apply the broad filter:**
> ```
> http.request && http.user_agent matches "probe/"
> ```
> This returns 16 rows - 11 from the real attacker and 5 from the decoy host.
> All 16 share the same `probe/NNN;` structure.
>
> **Step 2 - Confirm no normal requests match.** Apply:
> ```
> http.request && ip.src == 192.168.1.20 && http.user_agent matches "probe/"
> ```
> Zero rows returned. The pattern does not appear in any legitimate browser traffic.
>
> **Step 3 - Note the full structure.** It is always `probe/` followed by one
> to three decimal digits, followed by `;`. The surrounding User-Agent string is
> crafted to look like a standard compatibility tag, which is why a casual glance
> at any single packet doesn't raise an alarm.

---

**Q7. What is the 5th character (1-indexed) of the exfiltrated message?**

`S`

> **Step 1 - Filter and sort ascending by frame number:**
> ```
> http.request && ip.src == 192.168.1.50 && ip.dst == 93.184.216.34
> ```
> Click the `No.` column header to sort ascending if not already sorted.
>
> **Step 2 - Read the User-Agent of the 5th row.** It contains `probe/83;`.
>
> **Step 3 - Convert:** decimal `83` in the ASCII table is `S`.
>
> Distractor: `192.168.1.77` also sends a request containing `probe/83;` in its
> User-Agent, but to `203.0.113.10`, not `93.184.216.34`. If you apply
> `http.user_agent matches "probe/"` without filtering by destination, the
> interleaved decoy requests shift your row count and you land on the wrong packet.

---

**Q8. What is the full exfiltrated message?**

`ODYSSEY2KEY`

> **Step 1 - Apply the exfil filter and sort ascending:**
> ```
> http.request && ip.src == 192.168.1.50 && ip.dst == 93.184.216.34
> ```
> Confirm 11 rows.
>
> **Step 2 - Read the decimal value from each row's User-Agent, in order:**
>
> | Row | User-Agent substring | Decimal | Character |
> |-----|----------------------|---------|-----------|
> | 1   | probe/79;            | 79      | O         |
> | 2   | probe/68;            | 68      | D         |
> | 3   | probe/89;            | 89      | Y         |
> | 4   | probe/83;            | 83      | S         |
> | 5   | probe/83;            | 83      | S         |
> | 6   | probe/69;            | 69      | E         |
> | 7   | probe/89;            | 89      | Y         |
> | 8   | probe/50;            | 50      | 2         |
> | 9   | probe/75;            | 75      | K         |
> | 10  | probe/69;            | 69      | E         |
> | 11  | probe/89;            | 89      | Y         |
>
> **Step 3 - Concatenate in frame-number order:** `ODYSSEY2KEY`.
>
> Distractor: the decoy stream (`192.168.1.77` -> `203.0.113.10`) uses values
> `132`, `155`, `178`, `144`, `167` - all above 127 and outside the standard
> printable ASCII range. Attempting to decode them gives garbage, which is the
> signal that you are looking at the wrong stream.

---

**Q9. How many seconds elapse between the first and last exfil request?**

`80.000 seconds`

> **Step 1 - Apply the exfil filter:**
> ```
> http.request && ip.src == 192.168.1.50 && ip.dst == 93.184.216.34
> ```
> Sort ascending by frame number.
>
> **Step 2 - Read the `Time` column** for the first and last (11th) rows.
> Set Wireshark's time display to **View -> Time Display Format -> Seconds Since
> Beginning of Capture** for a clean reading:
> - First exfil request: `1.503000`
> - Last exfil request:  `81.503000`
>
> **Step 3 - Subtract:** `81.503 - 1.503 = 80.000 seconds`.
>
> Note the regular 8-second interval between consecutive requests. A fixed
> beacon interval is itself a detection signal - real slow-exfil tooling
> typically adds jitter (e.g. +-20% randomness) to make the interval appear
> less mechanical.

---

**Q10. How many requests does the decoy host send using the same `probe/` pattern?**

`5`

> **Step 1 - Filter to all `probe/` traffic going to the decoy destination:**
> ```
> http.request && http.user_agent matches "probe/" && ip.dst == 203.0.113.10
> ```
>
> **Step 2 - Read the status bar count:** `5 displayed`.
>
> **Step 3 - Confirm the source.** All 5 originate from `192.168.1.77`. This
> host uses the same User-Agent structure as the attacker but targets a different
> destination, uses non-printable decimal values (`132`, `155`, `178`, `144`,
> `167`), and encodes no coherent message. It exists to waste an analyst's time
> if they find the `probe/` pattern but decode all matching traffic without first
> confirming the destination.

---

## What Did I Just Do?

### The technique: HTTP header steganography

A covert channel over HTTP headers works by hiding data in a field whose content
is not validated by any intermediate device. `User-Agent` is a common choice
because:

- Every HTTP client sends it, so its presence triggers no alert.
- Its content is a free-form string. Proxies, firewalls, and web servers log it
  but rarely inspect it for structural anomalies.
- It is transmitted in plaintext over HTTP/1.1 without TLS, so no encryption is
  needed on the attacker's side.

The encoding here - embedding the decimal ASCII value of one character per request
in a substring that resembles a version compatibility tag - is deliberately
low-tech. It requires no cryptography, no custom protocol, and no modification to
the request body or URL. The entire exfiltration channel is visible in standard
access logs.

### Why "slow" matters

The 8-second interval between requests is the design choice that makes this
hard to detect in real time:

- Volume-based alerts (requests per second) do not fire. 11 requests over 80
  seconds is indistinguishable from normal browsing behavior by rate alone.
- Total data volume is negligible. Each request is a few hundred bytes of
  standard HTTP overhead. No byte-count threshold triggers.
- Each individual packet looks completely benign. Detection requires correlating
  the User-Agent values across multiple requests in sequence - something that
  happens in log analysis, not in real-time packet inspection.

### What the decoy was designed to do

The decoy stream from `192.168.1.77` is an analyst distraction. An analyst who
finds the `probe/` pattern and immediately decodes all matching requests gets
nonsense output (132, 155, 178, 144, 167 are all outside printable ASCII range)
and may conclude the pattern is a coincidence or a benign tool version tag.

The correct discipline is to confirm source IP, destination IP, encoding, and
pattern all together before drawing conclusions from any one field in isolation.

### Detection approaches

- **User-Agent anomaly scoring.** A SIEM rule matching `probe/\d+;` catches this
  immediately. More broadly, User-Agent strings that do not match known browser
  or tool fingerprints can be flagged for review.
- **Beacon detection.** Fixed-interval outbound connections from one host to one
  external IP is a classic C2 or exfil beacon signature. Network analysis tools
  designed for this (looking for consistent inter-packet timing across flows) will
  surface it regardless of what the payload looks like.
- **TLS.** This attack used plaintext HTTP. Over HTTPS the User-Agent is encrypted
  and invisible to network-layer inspection. Detection then requires endpoint
  visibility - monitoring what the process is constructing and sending before TLS
  encapsulation.



***                                                                 

<b><i>Looking for a different CTF/Lab? </br>[Lab Directory](/navigation.md)</i></b>
