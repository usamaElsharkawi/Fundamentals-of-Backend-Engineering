# Lecture 27: HTTP/2

## Status: In Progress

> ### 📋 Read this before continuing
>
> This lecture covers **HTTP/2** — how it improves on HTTP/1.1, its pros and cons, and when to use it. It walks through the HTTP/1.1 connection bottleneck, HTTP/2 multiplexing and stream IDs, header/data compression, secure by default (ALPN), server push (dead), TCP head-of-line blocking, CPU overhead, and workload-based recommendations.
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. HTTP/1.1 connection bottleneck — why browsers open 6 connections per domain | ✅ Documented | Unit 1 |
> | 2. HTTP/2 multiplexing & stream IDs (odd = client, even = server) | ✅ Documented | Unit 2 |
> | 3. Pros of HTTP/2 (multiplexing, header compression, secure by default) | ✅ Documented | Unit 3 |
> | 4. Cons of HTTP/2 (TCP HOL blocking, CPU overhead, server push failure, low-request workloads) | ✅ Documented | Unit 4 |
> | 5. When to use HTTP/1.1 vs HTTP/2 | ✅ Documented | Unit 5 |
>
> | Unit | Content |
> |---|---|
> | **1** | **HTTP/1.1 connection bottleneck** — one request per connection, pipelining failure, 6 connections per domain |
> | **2** | **HTTP/2 multiplexing & stream IDs** — concurrent streams on one connection, odd/even numbering, out-of-order responses |
> | **3** | **Pros of HTTP/2** — multiplexing, header/data compression (HPACK), secure by default (ALPN) |
> | **4** | **Cons of HTTP/2** — TCP HOL blocking, CPU overhead (binary framing + flow control), server push, low-request workloads |
> | **5** | **HTTP/1.1 vs HTTP/2** — workload determines speed, not protocol alone |

---

## Unit 1 — HTTP/1.1 Connection Bottleneck

### Direct Answer

HTTP/1.1 is a **request → response** protocol where a single TCP connection carries **one request at a time**. The connection is "busy" until the full response arrives. To load a page with multiple resources (HTML, JS, CSS, images), the browser opens **up to 6 concurrent TCP connections per domain** and fires requests in parallel across them.

### Chunk 1 — One request, one response per connection

```
GET /index.html HTTP/1.1
Host: example.com
```

The client sends the request and waits. The server processes it and sends back the full response:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 2048

<html>...</html>
```

**What it does:** This is a synchronous exchange. The TCP connection is occupied from the moment the request is sent until the last byte of the response is received.

### Chunk 2 — The connection is "busy"

```
[Connection 1]  GET /index.html  →  waiting...  →  HTML received  →  FREE
```

**What it does:** While the server is processing and transmitting, no other request can use this connection. If the HTML references 10 other resources (CSS, JS, images), they must wait — either sequentially on this connection or on other connections.

### Chunk 3 — Pipelining was tried, and failed

HTTP/1.1 allows **pipelining**: sending multiple requests on the same connection without waiting for responses:

```
GET /a.js   HTTP/1.1
GET /style.css HTTP/1.1
GET /img1.jpg HTTP/1.1
```

**The catch:** Responses must arrive in the **exact same order** as the requests. If the response for `/a.js` is slow, the responses for `/style.css` and `/img1.jpg` sit stuck behind it — even if the server already finished them. This is **head-of-line (HOL) blocking**. Browsers mostly disabled pipelining because of it.

> See `nested-hol-blocking.md` for a standalone explanation of head-of-line blocking.

### Chunk 4 — Browser solution: up to 6 connections per domain

To work around the one-at-a-time limit, browsers open **up to 6 TCP connections to the same domain** and dispatch requests in parallel:

```
[Conn 1]  GET /index.html
[Conn 2]  GET /app.js
[Conn 3]  GET /style.css
[Conn 4]  GET /img1.jpg
[Conn 5]  GET /img2.jpg
[Conn 6]  GET /img3.jpg
```

If more than 6 resources are needed, they queue and wait for a connection to free up.

**What it does:** Parallelizes resource loading across multiple connections, reducing total page load time compared to sequential fetching on one connection.

**Why 6?** Chrome picked it through trial and error. It's **per domain** — `example.com` gets 6, `api.example.com` gets another 6.

### Chunk 5 — The cost of 6 connections

Each TCP connection costs:

1. **TCP handshake** — SYN, SYN-ACK, ACK (3 round trips)
2. **TLS handshake** (if HTTPS) — additional round trips for certificate verification and key exchange
3. **Memory** — client and server must keep sockets open
4. **Server file descriptors** — each open connection consumes a file descriptor on the server

**What it does:** 6 parallel connections help, but they're still a bottleneck compared to sending everything over one pipe. Each connection consumes resources on both client and server.

---

## Assembly Order

The HTTP/1.1 bottleneck assembles like this:

1. **Chunk 1** — Understand that HTTP/1.1 sends one request per TCP connection.
2. **Chunk 2** — Recognize that the connection is busy until the response completes.
3. **Chunk 3** — See why pipelining failed (ordered responses → HOL blocking).
4. **Chunk 4** — Learn the browser workaround: up to 6 parallel connections per domain.
5. **Chunk 5** — Understand the tradeoff: parallelism helps, but each connection has a handshake, TLS, and memory cost.

This is why HTTP/2 was invented: to send all these requests over **one** connection instead of six.

---

## Unit 2 — HTTP/2 Multiplexing & Stream IDs

### Direct Answer

HTTP/2 solves the HTTP/1.1 bottleneck by breaking each request/response into **binary frames** tagged with a unique **stream ID**. Multiple streams interleave freely on a single TCP connection — no more one-at-a-time waiting, no more 6 parallel connections.

### Chunk 1 — The problem: HTTP/1.1 still sends one request at a time

Even with keep-alive, HTTP/1.1 processes requests sequentially on a connection:

```
Connection 1:
  Request 1 → /index.html    → Response 1 (HTML)
  Request 2 → /app.js        → Response 2 (JS)
  Request 3 → /style.css     → Response 3 (CSS)
```

If `/index.html` is slow, `/app.js` and `/style.css` wait behind it.

### Chunk 2 — HTTP/2 solution: binary framing + stream IDs

HTTP/2 tags every frame with a **stream ID**. Multiple streams interleave on one TCP connection:

```
Single TCP connection:

Stream 1:   GET /index.html
Stream 3:   GET /app.js
Stream 5:   GET /style.css
Stream 7:   GET /logo.png

Server responds (any order):
Stream 5:   [CSS frame part 1]
Stream 3:   [JS frame part 1]
Stream 5:   [CSS frame part 2]
Stream 1:   [HTML response complete]
Stream 7:   [PNG response]
Stream 3:   [JS response complete]
```

**What it does:** The browser uses each frame's stream ID to reassemble the correct response. No stream waits for another.

### Chunk 3 — Stream ID numbering: odd = client, even = server

- **Client-initiated streams**: odd numbers — 1, 3, 5, 7, 9...
- **Server-initiated streams**: even numbers — 2, 4, 6, 8, 10...

```
Client sends:    Stream 1, 3, 5, 7   (requests)
Server responds: Stream 2, 4, 6, 8   (responses, if server-initiated)
```

The server can also push on even-numbered streams (server push — now dead, replaced by Early Hints).

### Chunk 4 — Out-of-order responses

There's no required response order per stream. A multithreaded server can process Stream 7 first, then Stream 3, then Stream 1 — responses arrive as they finish.

```
Slow request (Stream 1):   5 seconds
Fast request (Stream 3):   100 ms

Stream 3 response arrives first → browser renders it
Stream 1 response arrives later  → no one was blocked
```

**What it does:** A slow response on stream 1 doesn't prevent completed responses on streams 3 and 5 from being delivered.

### Chunk 5 — The TCP limitation

HTTP/2 still runs over TCP. TCP guarantees **ordered byte delivery**, so if one TCP packet is lost, all streams stall until it's retransmitted — even streams whose data already arrived. This is **TCP head-of-line blocking**. HTTP/3 fixes this by moving to QUIC over UDP.

---

### Assembly Order

The HTTP/2 multiplexing model assembles like this:

1. **Chunk 1** — Recognize that HTTP/1.1 keep-alive still sends one request at a time per connection.
2. **Chunk 2** — HTTP/2 uses binary frames with stream IDs to multiplex many streams on one TCP connection.
3. **Chunk 3** — Client streams are odd (1,3,5...), server streams are even (2,4,6...).
4. **Chunk 4** — Responses can arrive out of order; slow streams don't block fast ones.
5. **Chunk 5** — TCP still causes transport-level head-of-line blocking (solved by HTTP/3/QUIC).

---

> ### 🔗 Linked Nested Question
>
> **What is head-of-line blocking?** → See `nested-hol-blocking.md`

---

## Unit 3 — Pros of HTTP/2

### Direct Answer

HTTP/2 improves on HTTP/1.1 through four advantages: **multiplexing over a single connection**, **header/data compression (HPACK)**, **secure by default** (via ALPN + TLS), and **efficient binary framing**.

### Chunk 1 — Multiplexing over a single connection

HTTP/1.1 needs up to 6 TCP connections per domain to load resources in parallel. HTTP/2 sends all requests and responses over **one** TCP connection, interleaved by stream ID.

```
HTTP/1.1:  6 connections, 6 TLS handshakes, 6 sockets
HTTP/2:    1 connection,  1 TLS handshake,  1 socket

HTTP/2 frames on one connection:
Stream 1:  GET /index.html
Stream 3:  GET /app.js
Stream 5:  GET /style.css
Stream 7:  GET /logo.png
```

**What it does:** Eliminates the connection bottleneck. Fewer connections mean fewer handshakes, less memory, and fewer file descriptors on the server.

### Chunk 2 — Header and data compression (HPACK)

HTTP/1.1 sends headers as plain text with every request. Cookies, User-Agent, and other headers repeat on every request, wasting bytes.

HTTP/2 uses **HPACK** — a header compression format that:
- Indexes previously seen headers (references them by number instead of resending)
- Uses Huffman encoding for new header values
- Compresses both headers and data

```
HTTP/1.1 request headers (repeated on every request):
GET /api/user HTTP/1.1
Host: example.com
Cookie: session=abc123; theme=dark; lang=en
User-Agent: Mozilla/5.0 ...
Accept: application/json
Accept-Language: en-US,en;q=0.9

HTTP/2: same headers sent once, then referenced by index on subsequent requests
```

**What it does:** Drastically reduces bandwidth, especially for APIs with many small requests carrying large cookie/header overhead.

**Why HTTP/1.1 didn't compress headers:** The CRIME attack — an attacker could extract cookie values by observing how compression changed response sizes. HTTP/2 brought compression back safely with HPACK, which is resistant to that attack.

### Chunk 3 — Secure by default (via ALPN + TLS)

HTTP/2 effectively requires TLS. The browser won't negotiate HTTP/2 over plaintext.

**Why?** **Protocol ossification.** Middleboxes (routers, firewalls, proxies) built in the 1990s understand HTTP/1.1's text format on port 80. If you send HTTP/2 binary frames over plain port 80, those middleboxes reject it — they only know the old rules.

Wrapping HTTP/2 in TLS makes the traffic opaque, so middleboxes can't inspect or block it based on content.

The client and server negotiate HTTP/2 during the TLS handshake using **ALPN** (Application Layer Protocol Negotiation):

```
TLS Client Hello:  ALPN: ["h2", "http/1.1"]
TLS Server Hello:  ALPN: ["h2"]   ← server picks HTTP/2
```

**What it does:** HTTP/2 is secure by default because TLS is the vehicle that lets it evolve without middleboxes breaking it.

### Chunk 4 — Binary framing (efficiency over text parsing)

HTTP/1.1 is text-based. The server reads byte-by-byte until it finds request boundaries (`GET /path HTTP/1.1`, headers until blank line, `Content-Length`).

HTTP/2 uses a binary frame format:
- Fixed frame header (length, type, flags, stream ID)
- Variable frame payload

```
HTTP/2 frame:
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+---------------+---------------+-----------------+
|                 Stream ID (31)                |
+-----------------------------------------------+
|                   Payload (...)               |
+-----------------------------------------------+
```

**What it does:** The server parses frames efficiently — no hunting for blank lines or content lengths. Each frame is self-describing with its stream ID and length.

---

### Assembly Order

The pros of HTTP/2 assemble like this:

1. **Chunk 1** — Multiplexing replaces 6 connections with 1, reducing handshakes and resource usage.
2. **Chunk 2** — HPACK compresses headers, saving bandwidth on repeated requests.
3. **Chunk 3** — TLS + ALPN make HTTP/2 secure by default and let it bypass ossified middleboxes.
4. **Chunk 4** — Binary framing lets the server parse requests efficiently without text scanning.

Together these make HTTP/2 faster and more efficient than HTTP/1.1 for pages with many resources.

---

## Unit 4 — Cons of HTTP/2

### Direct Answer

HTTP/2 is not perfect. Its drawbacks are: **TCP head-of-line blocking**, **CPU overhead** from binary framing and per-stream flow control, **server push failure**, and **no benefit for low-request workloads**.

### Chunk 1 — TCP head-of-line blocking (transport-level)

HTTP/2 still runs over TCP. TCP guarantees ordered byte delivery, so if one TCP packet is lost, all streams stall until it's retransmitted — even streams whose data already arrived fine.

```
Stream 1:   [packet lost]  →  TCP stalls
Stream 3:   [data arrived] →  BLOCKED waiting for Stream 1's packet
Stream 5:   [data arrived] →  BLOCKED waiting for Stream 1's packet
```

**What it does:** Application-level multiplexing helps, but TCP-level head-of-line blocking still limits HTTP/2. HTTP/3 solves this with QUIC over UDP (independent streams).

### Chunk 2 — CPU overhead (binary framing + flow control)

HTTP/1.1 is simple: read until blank line, check Content-Length, serve. HTTP/2 requires more work per request:

- Parse binary frames (length, type, flags, stream ID)
- Reassemble frames into requests by stream ID
- Per-stream flow control (WINDOW_UPDATE frames)
- HPACK header compression/decompression

```
HTTP/1.1 backend:
  Read stream → find request end → serve → done

HTTP/2 backend:
  Read frame → check stream ID → buffer frame → 
  wait for all frames of this stream → reassemble → 
  decompress headers (HPACK) → check flow control → serve → 
  send WINDOW_UPDATE → repeat for next frame
```

**What it does:** The backend does more CPU work per request. The lecture puts it bluntly: "the backend starts doing a job that is not really its job" — parsing framing and flow control instead of just serving.

### Chunk 3 — Server push (dead)

HTTP/2 originally included **server push**: the server proactively sends resources it thinks the client will need (CSS, JS, images) without being asked.

```
Client:    GET /index.html
Server:    HTML response + pushes /style.css, /app.js, /logo.png
```

**Why it failed:**
1. The server doesn't know what the client has cached — it pushes resources the client already has.
2. HTTP's mental model is one request → one response. Pushing multiple streams confuses clients.
3. Building a client that understands push complexity is very hard.
4. It breaks normal APIs — not compatible with the single request/single response model.

**Replacement:** Early Hints — the server hints at future resources with a header, and the client decides whether to fetch them.

```
103 Early Hints: Link: </style.css>; rel=preload
```

### Chunk 4 — No benefit for low-request workloads

HTTP/2 was designed because browsers send many requests concurrently on one connection. If you don't send many requests, HTTP/2 adds overhead for no gain.

```
Low-request workload: 1 request per minute, backend processes instantly

HTTP/1.1:  open → request → response → done  (simple, fast)
HTTP/2:    negotiate ALPN → binary frame → HPACK → flow control → serve  (overhead)
```

**The key insight from the lecture:** "HTTP/2 is not faster. The workload defines the fastness." If you send one request per second and it processes fast, HTTP/1.1 is fine. HTTP/2 shines when you're sending 100+ requests concurrently (like a browser loading 100 images).

---

### Assembly Order

The cons of HTTP/2 assemble like this:

1. **Chunk 1** — TCP still causes head-of-line blocking (solved by HTTP/3/QUIC).
2. **Chunk 2** — Binary framing + per-stream flow control + HPACK cost more CPU than plain text parsing.
3. **Chunk 3** — Server push was a bad idea; dead, replaced by Early Hints.
4. **Chunk 4** — HTTP/2 adds overhead with no benefit for low-request workloads.

---

## Unit 5 — When to Use HTTP/1.1 vs HTTP/2

### Direct Answer

Choose based on your **workload**, not because HTTP/2 is "newer" or "faster."

### Chunk 1 — Use HTTP/2 when: many concurrent requests

HTTP/2 shines when a client sends many requests at the same time:

- Browser loading a page with 50–100 resources (HTML, JS, CSS, images)
- API clients making many parallel calls
- Mobile apps fetching multiple endpoints on load

**Why:** Multiplexing over one connection avoids the 6-connection limit and reduces handshake/TLS overhead.

```
Use HTTP/2 when: "you're sending 100 requests at the same time"
```

### Chunk 2 — Use HTTP/1.1 when: few, infrequent requests

HTTP/1.1 is simpler and cheaper when requests are rare and fast:

- One request per minute or per second
- Backend processes each request instantly
- Simple APIs with low traffic

**Why:** HTTP/2's framing, flow control, and HPACK overhead aren't worth it if you're not multiplexing.

```
Use HTTP/1.1 when: "you send one request a minute, it processes fast, you get the response immediately"
```

### Chunk 3 — The workload determines speed

The lecture's core message: **HTTP/2 is not inherently faster.** The workload defines the fastness.

| Workload | Better choice | Why |
|----------|-------------|-----|
| Many concurrent resources (browser page load) | HTTP/2 | Multiplexing avoids connection limit |
| Few, infrequent, fast requests | HTTP/1.1 | Simpler, less CPU overhead, no multiplexing needed |
| Server needs to push resources | Neither (use Early Hints) | Server push is dead |

### Chunk 4 — Practical rule of thumb

- If you're serving a website with lots of assets → HTTP/2 (usually automatic via your CDN/load balancer).
- If you're building a simple internal API with low traffic → HTTP/1.1 is perfectly fine.
- Don't spin up extra Docker containers/Kubernetes pods just to handle HTTP/2 CPU overhead unless you actually have the request volume to justify it.

---

## Gap Pass — Lecture 27 Complete

All units covered:

| # | Question | Status | Unit |
|---|----------|--------|------|
| 1 | HTTP/1.1 connection bottleneck — why browsers open 6 connections per domain | ✅ Documented | Unit 1 |
| 2 | HTTP/2 multiplexing & stream IDs (odd = client, even = server) | ✅ Documented | Unit 2 |
| 3 | Pros of HTTP/2 (multiplexing, header/data compression, secure by default) | ✅ Documented | Unit 3 |
| 4 | Cons of HTTP/2 (TCP HOL blocking, CPU overhead, server push failure, low-request workloads) | ✅ Documented | Unit 4 |
| 5 | When to use HTTP/1.1 vs HTTP/2 | ✅ Documented | Unit 5 |

Nested questions answered:
- What is head-of-line blocking? → `nested-hol-blocking.md` (linked from Units 1, 2, 4)

Key takeaways from the transcript:
- HTTP/2 was designed because clients send many requests concurrently on one connection.
- Multiplexing + stream IDs (odd=client, even=server) replace the 6-connection workaround.
- HPACK compresses headers; ALPN negotiates HTTP/2 over TLS; secure by default due to protocol ossification.
- Server push failed (cached resources, breaks one-request/one-response model); replaced by Early Hints.
- TCP head-of-line blocking still affects HTTP/2; HTTP/3/QUIC fixes it.
- HTTP/2 is not inherently faster — the workload defines the fastness.
- Performance test in lecture: H2 loads 100 images way faster than H1 on a 3G network because of multiplexing + connection limits.

*Status: Complete*
