# Lecture 28: HTTP/3

## Status: In Progress

> ### 📋 Read this before continuing
>
> This lecture covers **HTTP/3** — how QUIC over UDP solves HTTP/2's TCP head-of-line blocking, the QUIC handshake, QUIC features (connection migration, per-stream flow control/congestion control), QPACK vs HPACK, and the cons of HTTP/3 (CPU overhead, UDP blocking by proxies, IP fragmentation).
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. TCP head-of-line blocking & QUIC's per-stream solution | ✅ Documented | Unit 1 |
> | 2. QUIC handshake (secure by default, merged TLS + TCP handshake) | ✅ Documented | Unit 2 |
> | 3. QUIC features (connection migration, per-stream flow control, congestion control) | ✅ Documented | Unit 3 |
> | 4. QPACK vs HPACK (why HPACK breaks without ordered delivery) | ✅ Documented | Unit 4 |
> | 5. Cons of HTTP/3/QUIC (CPU overhead, UDP blocking, IP fragmentation) | ✅ Documented | Unit 5 |
>
> | Unit | Content |
> |---|---|
> | **1** | **TCP head-of-line blocking & QUIC per-stream solution** — lost TCP segment blocks all streams; QUIC orders per stream only |
> | **2** | **QUIC handshake** — merged TCP 3-way + TLS 1.3 into 1-RTT, secure by default |
> | **3** | **QUIC features** — connection ID, connection migration, per-stream flow control, per-stream congestion control |
> | **4** | **QPACK vs HPACK** — static dictionary replaces deflate; HPACK relies on TCP order guarantees |
> | **5** | **Cons of HTTP/3/QUIC** — CPU overhead, enterprise proxies blocking UDP, IP fragmentation (DF flag, PMTU discovery) |

---

## Unit 1 — TCP Head-of-Line Blocking & QUIC's Per-Stream Solution

### Direct Answer

HTTP/2 multiplexes many streams over one TCP connection, but TCP guarantees **ordered byte delivery per connection**. If one segment is lost, all streams stall until it's retransmitted — even streams whose data already arrived. HTTP/3 replaces TCP with QUIC over UDP, enforcing ordering **per stream** so a lost datagram only blocks the affected stream.

### Chunk 1 — HTTP/2 over TCP: one connection, many streams

HTTP/2 tags each request/response with a stream ID, but all streams ride on the **same** TCP byte stream:

```
TCP connection:
Stream 1: GET /main.js   → segments 1, 2
Stream 2: GET /style.css → segments 3, 4
Stream 3: GET /img1.jpg  → segments 5, 6
Stream 4: GET /img2.jpg  → segments 7, 8
```

TCP labels segments by sequence number: 1, 2, 3, 4, 5, 6, 7, 8.

**What it does:** Multiple independent HTTP streams share one TCP connection.

### Chunk 2 — TCP delivers in order, period

TCP guarantees **ordered byte delivery**. If segment 3 is lost, the kernel buffers segments 4–8 but won't deliver them to the application until segment 3 arrives and is acknowledged.

```
Segments 1,2 arrived → Stream 1 (main.js) delivered to app ✓
Segment 3 LOST
Segments 4,5,6,7,8 arrived but BLOCKED by the kernel
  → Stream 2 (style.css) stuck
  → Stream 3 (img1.jpg) stuck
  → Stream 4 (img2.jpg) stuck
```

**Why?** TCP doesn't know about streams. It sees one dumb byte stream. "Missing segment 3? Everything after it waits."

**What it does:** A single lost packet stalls all streams on that connection, even though streams 2, 3, and 4 have all their data already.

### Chunk 3 — Streams are independent; TCP is not

```
Stream 1: segments 1,2   (complete)
Stream 2: segments 3,4   (segment 3 lost)
Stream 3: segments 5,6   (complete, but stuck behind segment 3)
Stream 4: segments 7,8   (complete, but stuck behind segment 3)
```

The app never even knows Stream 2, 3, 4 requests exist — they haven't been parsed yet because TCP won't release the bytes.

### Chunk 4 — QUIC over UDP: per-stream ordering

QUIC replaces TCP with UDP. UDP has no ordering guarantees, so QUIC builds its own reliability **per stream**.

```
UDP datagrams (not TCP segments):
Stream 1: datagrams 1,2  → delivered to app ✓
Stream 2: datagram 3 LOST → only Stream 2 stalls
Stream 3: datagrams 5,6  → delivered to app ✓ (independent of stream 2)
Stream 4: datagrams 7,8  → delivered to app ✓
```

QUIC tracks sequence numbers inside the UDP datagrams and knows which stream each belongs to via a stream ID. It only blocks the specific stream that's missing data.

**What it does:** A lost datagram on stream 2 stalls only stream 2; streams 1, 3, and 4 continue to the application.

### Chunk 5 — The mental model

| | HTTP/2 over TCP | HTTP/3 over QUIC/UDP |
|---|---|---|
| Transport | One TCP connection | One QUIC connection over UDP |
| Ordering guarantee | Per-connection (all streams) | Per-stream only |
| Lost segment effect | All streams stall | Only the affected stream stalls |

---

### Assembly Order

The HTTP/3 head-of-line blocking solution assembles like this:

1. **Chunk 1** — HTTP/2 sends many streams over one TCP connection.
2. **Chunk 2** — TCP forces ordered delivery; one lost segment blocks everything after it.
3. **Chunk 3** — Independent streams get stuck behind a lost segment they don't need.
4. **Chunk 4** — QUIC uses UDP and enforces ordering per stream, so only the affected stream stalls.
5. **Chunk 5** — HTTP/3 solves head-of-line blocking by moving ordering from the connection to the stream.

---

## Unit 2 — QUIC Handshake (Secure by Default)

### Direct Answer

QUIC merges the TCP 3-way handshake and TLS 1.3 handshake into a single 1-RTT handshake over UDP. Encryption is mandatory — there is no plaintext QUIC mode. QUIC builds virtual connection semantics over stateless UDP using connection IDs.

### Chunk 1 — The old way: TCP handshake, then TLS handshake

With HTTPS over HTTP/1.1 or HTTP/2, you pay two separate handshakes:

```
Step 1 — TCP 3-way handshake:
Client → Server: SYN
Server → Client: SYN-ACK
Client → Server: ACK
       (TCP connection established, but plaintext)

Step 2 — TLS handshake:
Client → Server: Client Hello
Server → Client: Server Hello + Certificate + Finished
Client → Server: Finished
       (TLS encrypted channel established)

Total: ~2 round trips before any application data flows
```

**What it does:** Establishes a reliable pipe (TCP), then layers encryption on top (TLS). Two separate protocols, two round trips.

### Chunk 2 — Why two steps? TCP was designed before the web was secure

TCP was designed in the 1970s–80s, when the internet wasn't security-focused. Security wasn't part of TCP's job. So TLS was bolted on afterward as a separate layer.

```
Application data
        ↓
    TLS (encryption)    ← added later
        ↓
    TCP (reliability)   ← original design
        ↓
    IP
```

**What it does:** Explains why HTTPS needs two handshakes — TCP doesn't know about encryption; TLS doesn't know about reliability. They're separate concerns.

### Chunk 3 — QUIC's approach: merge them into one handshake

QUIC owns the whole stack (it's built on UDP, which gives it full control). It merges the TCP 3-way handshake and TLS 1.3 handshake into **one** handshake:

```
QUIC handshake (1 round trip):
Client → Server: Client Initial (includes crypto handshake start)
Server → Client: Server Initial + crypto handshake response
       (connection established AND encrypted after 1 RTT)

Application data can flow immediately after, encrypted.
```

**What it does:** One round trip gives you both connection setup and encryption. No separate TCP + TLS dance.

### Chunk 4 — Secure by default

Because QUIC controls everything from the start, encryption is mandatory — there's no plaintext QUIC mode.

```
QUIC: always encrypted (TLS 1.3 integrated)
TCP+TLS: optional encryption (plaintext TCP possible)
```

**What it does:** You can't accidentally send unencrypted QUIC. Security is built into the protocol, not layered on top as an afterthought.

### Chunk 5 — QUIC is still "connection-based" over stateless UDP

#### Chunk 5.1 — UDP is connectionless

UDP has no concept of a connection. Each datagram is independent:

```
UDP datagram: [source IP:port] [dest IP:port] [payload]
```

The network doesn't remember that datagram 2 belongs to the same "conversation" as datagram 1. There's no handshake, no state, no teardown.

**What it does:** UDP sends independent packets with no relationship between them.

#### Chunk 5.2 — TCP connection = IP + port tuple

A TCP connection is identified by the 4-tuple: source IP, source port, destination IP, destination port. If your IP changes (Wi-Fi → 4G), the 4-tuple breaks and the TCP connection dies.

```
TCP connection identified by:
  192.168.1.50:54321 → 93.184.216.34:443

You switch to 4G → IP becomes 10.20.30.40:
  10.20.30.40:54321 → 93.184.216.34:443   ← different tuple, connection dropped
```

**What it does:** TCP ties connection identity to network addresses, so roaming breaks the connection.

#### Chunk 5.3 — QUIC connection ID

QUIC adds a **connection ID** — a random identifier sent with every datagram. The connection is identified by this ID, not by IP:port.

```
QUIC datagram: [connection ID: 0xAB3F...] [stream ID] [payload]
               [source IP:port] [dest IP:port]
```

**What it does:** The connection ID follows the connection regardless of IP address changes.

#### Chunk 5.4 — Connection migration

When you move from Wi-Fi to 4G, your IP changes. With QUIC, the server sees the same connection ID from a new IP and knows it's you:

```
Wi-Fi:   connection ID 0xAB3F from 192.168.1.50:54321
4G:      connection ID 0xAB3F from 10.20.30.40:55555
         Server: "Same connection ID → same connection, just new address"
```

**What it does:** The connection survives network switches without re-handshaking.

#### Chunk 5.5 — Why connection ID must be plaintext

The connection ID can't be encrypted — the server needs to read it to know which TLS key to use to decrypt the rest of the datagram:

```
Connection ID (plaintext) → tells server which session key to use → decrypt payload
```

This is a security tradeoff: anyone can read the connection ID, which enables connection hijacking attacks (an attacker with the ID can send datagrams pretending to be you).

**What it does:** Enables routing/decryption but introduces hijacking risk — "nothing is free."

---

### Assembly Order

The QUIC handshake assembles like this:

1. **Chunk 1** — HTTPS over TCP needs two handshakes (TCP then TLS), ~2 RTTs.
2. **Chunk 2** — TCP was designed pre-web-security; TLS was added as a separate layer.
3. **Chunk 3** — QUIC merges TCP + TLS 1.3 into one handshake over UDP, ~1 RTT.
4. **Chunk 4** — QUIC is secure by default (always encrypted, no plaintext mode).
5. **Chunk 5** — QUIC builds virtual connection semantics over stateless UDP via connection IDs:
   - 5.1 UDP is connectionless
   - 5.2 TCP identifies connections by IP:port; roaming breaks them
   - 5.3 QUIC connection ID identifies connections independently of IP:port
   - 5.4 Connection ID enables connection migration across networks
   - 5.5 Connection ID must be plaintext (security tradeoff: hijacking risk)

---

## Unit 3 — QUIC Features: Connection Migration, Per-Stream Flow Control, Per-Stream Congestion Control

### Direct Answer

QUIC adds three features beyond solving head-of-line blocking: **connection migration** (survives IP changes via connection ID), **per-stream flow control** (slow stream doesn't throttle fast streams), and **per-stream congestion control** (one abusive stream doesn't hog the connection). All three are per-stream instead of per-connection like TCP.

### Chunk 1 — Connection migration (recap from Unit 2)

QUIC identifies connections by a **connection ID**, not by IP:port. When you switch networks (Wi-Fi → 4G), your IP changes but the connection ID stays the same:

```
Wi-Fi:   connection ID 0xAB3F from 192.168.1.50:54321
4G:      connection ID 0xAB3F from 10.20.30.40:55555
         Server: "Same connection ID → same connection, new address"
```

**What it does:** The connection survives network switches without re-handshaking. TCP would drop it because the 4-tuple changed.

### Chunk 2 — Per-stream flow control

Flow control prevents a fast sender from overwhelming a slow receiver. In QUIC, each stream has its **own flow control window** (WINDOW_UPDATE frames per stream):

```
Stream 1: window = 10 KB   (client reading fast)
Stream 2: window = 1 KB    (client reading slow)
Stream 3: window = 10 KB   (client reading fast)
```

Server can keep sending on streams 1 and 3 at full speed, but slows down stream 2 to match its receiver's window.

**What it does:** A slow consumer on stream 2 doesn't throttle streams 1 and 3. In TCP, flow control is per-connection, so one slow stream drags down everything.

### Chunk 3 — Per-stream congestion control

Congestion control prevents flooding the network. QUIC tracks congestion per stream, so if one stream is pumping lots of data, it backs off independently:

```
Stream 1: sending 100 datagrams/sec → detects loss → cuts to 50/sec
Stream 2: sending 5 datagrams/sec   → unaffected, keeps full speed
Stream 3: sending 10 datagrams/sec  → unaffected
```

**What it does:** One abusive stream doesn't hog the entire connection's congestion budget. In TCP, congestion control is per-connection, so packet loss on one stream slows all streams.

### Chunk 4 — TCP vs QUIC comparison

| Feature | TCP (HTTP/2) | QUIC (HTTP/3) |
|---|---|---|
| Flow control | Per-connection | Per-stream |
| Congestion control | Per-connection | Per-stream |
| Lost packet effect | All streams stall | Only affected stream stalls |
| Network switch (IP change) | Connection dropped | Connection migrates (same ID) |

---

### Assembly Order

The QUIC features assemble like this:

1. **Chunk 1** — Connection ID lets QUIC survive IP changes (connection migration).
2. **Chunk 2** — Per-stream flow control lets fast streams keep going while slow streams throttle independently.
3. **Chunk 3** — Per-stream congestion control lets one stream back off without dragging down others.
4. **Chunk 4** — TCP does all of these per-connection; QUIC does them per-stream, which is the core advantage.

---

## Unit 4 — QPACK vs HPACK: Why HPACK Breaks Without Ordered Delivery

### Direct Answer

HTTP/2 uses **HPACK** for header compression — a dynamic dictionary built up over the connection that relies on TCP's ordered delivery. HTTP/3 uses **QPACK**, which replaces dynamic compression with a static dictionary lookup, so headers can be resolved regardless of which stream arrives first.

### Chunk 1 — HPACK: dynamic dictionary + deflate/Huffman

HTTP/2's HPACK compresses headers by building a shared dynamic dictionary. When a header like `cookie: session=abc123` appears, it's indexed. Later requests reference it by index instead of resending the full value.

```
Request 1:  cookie: session=abc123  → added to dynamic dictionary, index 5
Request 10: cookie: session=abc123  → sent as index 5 (1 byte)
```

**What it does:** Saves bandwidth by referencing previously seen headers instead of repeating them.

### Chunk 2 — The CRIME attack (why HTTP/1.1 disabled compression)

If an attacker can inject chosen plaintext into headers and observe the compressed size, they can guess secret values like cookies:

```
Attacker sends:  cookie: session=a
Compressed size: 10 bytes

Attacker sends:  cookie: session=ab
Compressed size: 9 bytes  ← smaller means "ab" matched something in the dictionary
```

By observing size changes, the attacker can brute-force the cookie value one character at a time.

**What it does:** Demonstrates why dynamic compression over encrypted channels is dangerous — compression side channels leak secrets.

### Chunk 3 — HPACK relies on TCP's ordered delivery

HPACK's dynamic dictionary assumes request 1 is processed before request 10. If request 10 uses an index created in request 1, but request 10 arrives first (possible in QUIC where streams are independent), the decoder doesn't know what index 5 means.

```
HTTP/2 (TCP ordered):
Request 1 arrives → creates index 5
Request 10 arrives → uses index 5  ✓ decoder knows it

HTTP/3 (QUIC unordered):
Request 10 arrives → uses index 5  ✗ decoder has never seen index 5
Request 1 arrives  → creates index 5 (too late)
```

**What it does:** HPACK breaks if streams arrive out of order — the decoder can't resolve references to entries it hasn't seen yet.

### Chunk 4 — QPACK: static dictionary instead

QPACK replaces dynamic deflate compression with a **static dictionary** — a fixed mapping of common header names/values to byte codes, agreed upon by client and server upfront.

```
Static dictionary:
  0x0001 → :method: GET
  0x0002 → :method: POST
  0x0003 → :scheme: https
  ...
  0x0127 → cookie: session=...

Request 10: cookie: session=abc123 → sent as [0x0127] [encrypted value]
```

**What it does:** No dynamic state to build up. Each header is resolved independently via the static dictionary, so stream arrival order doesn't matter.

### Chunk 5 — The dictionary is exchanged and encrypted

The static dictionary is negotiated during the QUIC handshake. Both sides know it. Header values themselves are still encrypted (TLS 1.3), so the compression doesn't leak side-channel information like CRIME.

```
QUIC handshake: negotiate QPACK static dictionary
All headers:    resolved via static dictionary + encrypted values
```

**What it does:** Combines the bandwidth savings of header compression with resistance to CRIME-style attacks and independence from stream ordering.

---

### Assembly Order

The QPACK vs HPACK story assembles like this:

1. **Chunk 1** — HPACK uses a dynamic dictionary built up over the connection (references previous headers by index).
2. **Chunk 2** — Dynamic compression enables CRIME-style side-channel attacks on secrets like cookies.
3. **Chunk 3** — HPACK assumes ordered delivery: request creating an index must arrive before requests using it.
4. **Chunk 4** — QPACK uses a static dictionary agreed upfront, so each header resolves independently.
5. **Chunk 5** — The dictionary is exchanged in the QUIC handshake and values are encrypted, avoiding CRIME while remaining order-independent.

---

## Unit 5 — Cons of HTTP/3/QUIC

### Direct Answer

Nothing is free. HTTP/3/QUIC has three main drawbacks: **CPU overhead** (more parsing per packet), **enterprise proxies blocking UDP**, and **IP fragmentation** (DF flag + PMTU discovery complications).

### Chunk 1 — CPU overhead

QUIC does a lot more work per packet than TCP: parsing connection IDs, stream IDs, per-stream flow control windows, per-stream congestion control, QPACK dictionary lookups, and encryption.

```
TCP backend:    read stream → find request end → serve → done
QUIC backend:   parse datagram → extract connection ID → find session key → decrypt →
                extract stream ID → buffer by stream → reassemble → check flow control →
                check congestion window → QPACK decode → serve → WINDOW_UPDATE → repeat
```

**What it does:** More CPU per request. The lecture notes this is "a lot" — people are still working to minimize QUIC CPU usage, especially at scale.

### Chunk 2 — Enterprise proxies block UDP

Many corporate networks and middleboxes are configured to allow TCP (ports 80/443) but block UDP outright:

```
Corporate firewall:  TCP 443 → ALLOWED
                     UDP 443 → BLOCKED
```

If UDP is blocked, HTTP/3 can't connect. The client falls back to HTTP/2 over TCP, but the fallback adds latency.

**What it does:** HTTP/3 is unreachable on networks that block UDP — a real deployment barrier in enterprise environments.

### Chunk 3 — IP fragmentation is the enemy

UDP datagrams can be large. If a QUIC datagram exceeds the network's MTU (Maximum Transmission Unit), IP fragments it into smaller pieces. But fragmentation is problematic:

- Some fragments may arrive out of order
- Fragments can be spoofed
- Some middleboxes drop fragmented packets

QUIC avoids this by setting the **DF (Don't Fragment)** flag on IP packets. If a packet is too big, the network sends back an ICMP "packet too big" message, and QUIC repackages into smaller datagrams using **Path MTU Discovery**.

```
Large QUIC datagram → DF flag set → router can't fragment → ICMP "too big" sent back
                      → QUIC splits into smaller datagrams → retries
```

**What it does:** Avoids fragmentation overhead but adds round trips for PMTU discovery and complexity.

---

### Assembly Order

The cons of HTTP/3/QUIC assemble like this:

1. **Chunk 1** — QUIC does more CPU work per packet (connection IDs, stream IDs, flow control, congestion control, QPACK, encryption).
2. **Chunk 2** — Enterprise proxies/firewalls often block UDP, making HTTP/3 unreachable.
3. **Chunk 3** — IP fragmentation is dangerous; QUIC uses DF flag + PMTU discovery to avoid it, adding complexity.

---

## Gap Pass — Lecture 28 Complete

All units covered:

| # | Question | Status | Unit |
|---|----------|--------|------|
| 1 | TCP head-of-line blocking & QUIC's per-stream solution | ✅ Documented | Unit 1 |
| 2 | QUIC handshake (secure by default, merged TLS + TCP handshake) | ✅ Documented | Unit 2 |
| 3 | QUIC features (connection migration, per-stream flow control, congestion control) | ✅ Documented | Unit 3 |
| 4 | QPACK vs HPACK (why HPACK breaks without ordered delivery) | ✅ Documented | Unit 4 |
| 5 | Cons of HTTP/3/QUIC (CPU overhead, UDP blocking, IP fragmentation) | ✅ Documented | Unit 5 |

Key takeaways from the transcript:
- HTTP/2's TCP head-of-line blocking: one lost segment stalls all streams on a connection. QUIC orders per stream only.
- QUIC merges TCP 3-way handshake + TLS 1.3 into one 1-RTT handshake; secure by default (always encrypted).
- QUIC connection ID enables connection migration across IP changes (Wi-Fi → 4G).
- Per-stream flow control and per-stream congestion control prevent one slow/abusive stream from dragging down others.
- HPACK relies on TCP ordered delivery (dynamic dictionary); breaks if streams arrive out of order. QPACK uses a static dictionary agreed upfront, order-independent.
- CRIME attack is why dynamic header compression is dangerous; QPACK avoids it with static dictionary + encryption.
- Cons: CPU overhead, UDP blocked by enterprise proxies, IP fragmentation (DF flag + PMTU discovery).
- Connection ID must be plaintext (security tradeoff: hijacking risk).

*Status: Complete*

