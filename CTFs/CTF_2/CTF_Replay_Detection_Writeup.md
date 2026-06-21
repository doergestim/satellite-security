![image](https://github.com/user-attachments/assets/068fae26-6e8f-402f-ad69-63a4e6a1f59e)

# Replay Attack Detection CTF - Writeup

---

## Answers

**Q1. What HTTP endpoint receives the suspicious traffic in this capture?**

`/ingest`

> Open the pcap in Wireshark and apply:
>
> ```
> http.request
> ```
>
> Look at the Info column for the request path. Every request in this capture
> targets the same endpoint.

---

**Q2. What HTTP method is used for every request to that endpoint?**

`POST`

> Same filter as Q1 - the Info column shows `POST /ingest HTTP/1.1` for each
> request. Telemetry ingestion endpoints are almost always POST, since the
> client is pushing a JSON body, not requesting a resource.

---

**Q3. How many legitimate telemetry posts appear before the suspicious traffic
begins?**

`4`

> Filter for the endpoint and inspect each request body individually:
>
> ```
> http.request.method == "POST" && http.request.uri == "/ingest"
> ```
>
> Click through the matches in order. The first four have a JSON body
> containing `"mode":"NOMINAL"` and increasing `epoch` values roughly 12
> seconds apart - this is what normal telemetry looks like, one post per
> downlink pass update.

---

**Q4. How many requests make up the replay burst?**

`240`

> Use a display filter that matches the request body content:
>
> ```
> http.request.uri == "/ingest" && http contains "SAFE"
> ```
>
> Check the packet count in the status bar at the bottom of the Wireshark
> window, or use **Statistics -> Conversations** and look at the count for
> the relevant TCP streams. All 240 requests carry identical JSON bodies.

---

**Q5. What is the `epoch` value inside the JSON body of the replayed requests?
Is it the same for every replayed request, or does it change?**

`1713440100` - the same value in every single replayed request.

> Open any one of the matched packets from Q4, right-click the HTTP body,
> and choose **Copy -> Bytes -> Printable Text Only** to read the JSON.
> Then spot-check several other replayed packets - the body, including the
> epoch field, is byte-for-byte identical across all 240 requests. That
> identical, frozen value is the signature of a replay: a legitimate sensor
> would never report the exact same timestamp twice.

---

**Q6. What was the `epoch` value of the last legitimate post before the replay
burst began?**

`1713440036`

> Filter for `mode":"NOMINAL"` posts and find the last one before the
> burst starts:
>
> ```
> http.request.uri == "/ingest" && http contains "NOMINAL"
> ```
>
> Read the `epoch` field of the final match. Compare it to the epoch from
> Q5 - they are different values, which is the first clue that the later
> traffic isn't a continuation of normal reporting; it's old data being
> replayed.

---

**Q7. How many seconds elapse between the last legitimate post and the first
replayed request?**

`59 seconds`

> Find the frame number of the last `NOMINAL` post and the first `SAFE`
> post. Right-click each frame -> **Time Reference** or simply subtract the
> values in the Time column (set Wireshark's time display format to
> **Seconds Since Previous Displayed Packet**, or just read the absolute
> timestamps and subtract).
>
> This gap matters: normal telemetry arrives at a steady ~12 second cadence.
> A 59-second silence followed by a sudden burst is itself a detection
> signal, independent of the payload content.

---

**Q8. What HTTP response code does the server return to every replayed
request?**

`200 OK`

> Filter for the responses on the same TCP stream as the replay requests:
>
> ```
> http.response.code == 200
> ```
>
> Every single replayed POST is accepted and acknowledged exactly like a
> legitimate one. The server has no way to distinguish a replay from a real
> update, because it performs no freshness or duplicate check at all - it
> just parses the JSON and returns success.

---

**Q9. How many distinct TCP source ports does the attacker use across the
entire replay burst?**

`6`

> Use **Statistics -> Conversations -> TCP** and filter to the replay
> timeframe, or apply:
>
> ```
> http.request.uri == "/ingest" && http contains "SAFE"
> ```
>
> then **Statistics -> Endpoints** to see the distinct source ports
> involved. A small, reused pool of source ports across hundreds of
> requests is consistent with a local automated script (such as
> `xargs -P N` invoking `curl` in parallel) reusing a handful of concurrent
> connections rather than a botnet using many distinct clients.

---

**Q10. What check is missing from the server's handling of `/ingest` that
would have prevented this attack?**

A **freshness / replay-protection check** - specifically, the server has no
way to reject a request whose `epoch` (or a server-issued nonce) has already
been seen, and no maximum-age window to reject stale timestamps.

> There are three independent mechanisms that each would have stopped this,
> any one of which is an acceptable answer:
>
> 1. **Timestamp freshness window** - reject any `epoch` that is more than
>    N seconds old relative to server receive time.
> 2. **Replay cache** - track recently seen `epoch` values (or a hash of
>    the full payload) and reject duplicates.
> 3. **Server-issued nonce** - require each legitimate sender to first
>    request a one-time token from the server and embed it in the payload,
>    so a captured payload cannot be resubmitted later.
>
> The endpoint as captured here implements none of these - it accepts and
> parses any JSON body that fits the expected schema, with no concept of
> "have I seen this before."

---

## What Did I Just Do?

### The attack you just analyzed

A **replay attack** does not require breaking any cryptography or finding a
software bug. It works because the server has no memory of what it has
already accepted. The attacker:

1. Observes (or in this scenario, is handed) one legitimate telemetry
   payload from the live `/stream` feed.
2. Resubmits that exact payload to `/ingest` hundreds of times.
3. The server, having no concept of "this data point already arrived,"
   accepts every copy and updates the dashboard each time.

From the operator's point of view, the dashboard appears to freeze or loop -
the same battery voltage, same temperature, same mode, over and over -
because that is literally what is being pushed to it.

### Why this differs from a man-in-the-middle attack

No interception of live traffic is required. The attacker does not need to
sit between the satellite and the ground station, does not need to defeat
any encryption, and does not need network access to the real downlink at
all. They only need:

- One previously captured valid payload (from logs, a prior session, a
  leaked screenshot of the dashboard, or - as in Lab 2 - directly reading
  the `/stream` SSE feed).
- Network access to the `/ingest` endpoint itself.

This is what makes replay attacks dangerous against systems that conflate
"the message is correctly formatted" with "the message is current and
genuine." The payload here did not need to be forged at all - it is a
perfectly valid, properly structured telemetry frame. The problem is purely
that it is old.

### What evidence pointed to a replay, specifically

Three independent signals, each visible without decoding anything beyond
plain HTTP:

1. **Identical payload across many requests.** A real sensor reporting
   battery voltage and temperature will show small but constant variation
   between readings. Two hundred and forty requests with byte-for-byte
   identical JSON bodies is not how telemetry behaves.

2. **An unnatural request-rate spike.** Legitimate telemetry arrived at a
   steady ~12 second interval. The replay burst delivered 240 requests in
   about 4 seconds - over 50 requests per second, a rate no physical sensor
   reporting periodic telemetry would ever produce. This is exactly the
   kind of pattern **Statistics -> IO Graph** in Wireshark is built to
   surface visually: a flat baseline with a sharp vertical spike.

3. **A timing discontinuity.** The 59-second gap of silence immediately
   before the burst is itself suspicious - it suggests the attacker needed
   time to prepare and launch the replay tooling, breaking the previously
   steady cadence.

### Why "valid JSON" is not the same as "authentic data"

This is the core lesson of the whole CTF. The server in this lab performs
input validation - it checks that the body is well-formed JSON with the
expected fields - but input validation answers the question *"is this
shaped correctly?"*, not *"should I trust this right now?"*. Those are
different questions and require different defenses:

| Defense | Answers | Stops |
|---|---|---|
| Schema / input validation | Is this shaped correctly? | Malformed or malicious payloads |
| Authentication | Did this come from who it claims to? | Forged senders |
| Freshness / replay protection | Is this current, or have I seen it before? | Replay attacks |

A system can have perfect input validation and even strong authentication,
and still be fully vulnerable to replay if it has no concept of freshness.
That is exactly the gap exploited here.

### How real systems close this gap

The mitigations are not exotic - they are standard patterns used anywhere
a server accepts pushed updates from an untrusted or semi-trusted source:

- **Nonces.** The server issues a one-time token before accepting data; the
  token is consumed on first use and rejected on reuse. This requires a
  round trip before every submission, which adds latency but makes
  captured payloads worthless after one use.
- **Timestamp windows.** Reject anything outside a tight tolerance (for
  example +-300 seconds) of server-observed time. Cheap to implement,
  does not require state, but only protects against replays that are
  noticeably delayed - a fast enough replay within the window still
  succeeds.
- **Sequence numbers with a sliding window.** Each sender increments a
  counter; the server tracks the highest seen and rejects anything at or
  below it (with a small window for legitimate out-of-order delivery).
  This is how protocols like IPsec and TLS 1.3's 0-RTT data handle replay
  protection at the transport layer.

In this lab series, Lab 4 (TheRelay) demonstrates all three of these as
configurable toggles on a hardened groundstation, so you can directly
compare the same replay attack against a defended endpoint and see each
mechanism block it in turn.

***                                                                 

<b><i>Looking for a different CTF/Lab? </br>[Lab Directory](/navigation.md)</i></b>
