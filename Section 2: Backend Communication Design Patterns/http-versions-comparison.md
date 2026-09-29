# HTTP Versions Comparison — Deep Dive

## Status: Documented for Future Discussion

---

## Quick Overview

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| **Transport** | TCP | TCP | QUIC (UDP) |
| **Format** | Text | Binary frames | Binary frames |
| **Multiplexing** | ❌ (pipelining broken) | ✅ Streams | ✅ Streams (no HOL blocking) |
| **Header Compression** | ❌ | HPACK | QPACK |
| **Server Push** | ❌ | ✅ | ✅ |
| **Connection Reuse** | Keep-alive | Single connection | Connection ID (migration) |
| **TLS** | Optional | Required (ALPN) | Built-in (TLS 1.3) |
| **0-RTT** | ❌ | ❌ | ✅ |

---

## HTTP/1.1 — The Text Era

### Request Format
```
GET /api/users HTTP/1.1
Host: example.com
User-Agent: curl/7.68.0
Accept: application/json
Authorization: Bearer token123

```

### Key Characteristics

**Text-based parsing**: Every header is `Key: Value\r\n`. Parser scans for `\r\n\r\n` to find body start.

**Head-of-line blocking (HOL)**: On a single connection, Request B **must wait** for Request A to complete. Browsers opened 6 connections per domain to work around this.

**Pipelining (theoretical)**: Client could send multiple requests without waiting. **Broken in practice** — proxies and servers mishandled it. Disabled everywhere.

**No header compression**: `Cookie`, `User-Agent`, `Authorization` sent verbatim every request. Wasteful.

**Connection semantics**:
```
Connection: keep-alive   # Persist (default in 1.1)
Connection: close        # Close after this request
```

**Chunked encoding** (streaming body without Content-Length):
```
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
0\r\n
\r\n
```

### Where It Still Wins
- Simple debugging (`curl -v`, `telnet`)
- No TLS requirement
- Universal support (every client, every proxy)
- Low latency for single request/response

---

## HTTP/2 — Binary Framing + Multiplexing

### The Big Change: Binary Frames

Instead of text lines, everything is **frames**:

```
+-----------------------------------------------+
|                 Frame Header                  |
|  Length (24) | Type (8) | Flags (8) | Stream ID (31) |
+-----------------------------------------------+
|              Frame Payload (variable)         |
+-----------------------------------------------+
```

**Frame Types**:
| Type | Purpose |
|------|---------|
| `HEADERS` | Request/response headers (compressed) |
| `DATA` | Body payload |
| `SETTINGS` | Connection parameters |
| `PUSH_PROMISE` | Server push announcement |
| `PING` | Keepalive / RTT measurement |
| `GOAWAY` | Graceful shutdown |
| `WINDOW_UPDATE` | Flow control |

### Multiplexing — One Connection, Many Streams

```
Connection
├── Stream 1 (GET /users)     → HEADERS + DATA frames
├── Stream 3 (GET /posts)     → HEADERS + DATA frames  
├── Stream 5 (GET /comments)  → HEADERS + DATA frames
└── Stream 7 (Server Push)    → PUSH_PROMISE + DATA
```

**Each stream is independent**. Slow `/users` response doesn't block `/posts`. Frames interleave.

### HPACK Header Compression

- **Static table**: Common headers (`:method: GET`, `:path: /`, `:scheme: https`) → index lookup
- **Dynamic table**: Learned during connection (e.g., `Authorization: Bearer xyz` after first request)
- **Huffman encoding** for literal values

**Result**: Headers go from ~500-800 bytes → ~50-100 bytes.

### Server Push

Server can proactively send resources client will need:
```
Client: GET /index.html
Server: HEADERS (200 OK) + DATA (HTML)
Server: PUSH_PROMISE (Stream 2: /style.css)
Server: HEADERS (Stream 2) + DATA (CSS)
```

**Client can cancel** with `RST_STREAM` if already cached.

### Flow Control (Per-Stream + Connection)

`WINDOW_UPDATE` frames advertise receive window. Sender **must not exceed**. Prevents buffer bloat.

### The Catch: HOL Blocking at TCP Layer

```
TCP Segment 1: [Stream 1 HEADERS] [Stream 3 HEADERS] [Stream 5 DATA part 1]
TCP Segment 2: [Stream 5 DATA part 2]  ← LOST!
```

**TCP retransmits Segment 2** → **ALL streams blocked** until it arrives. HTTP/2 multiplexing is **logical**, not transport-level.

---

## HTTP/3 — QUIC: Fixing Transport-Layer HOL

### QUIC = UDP + TLS 1.3 + Reliability + Multiplexing

```
┌─────────────────────────────────────────────────────┐
│                    Application                       │
│  HTTP/3 (same semantics as HTTP/2)                  │
├─────────────────────────────────────────────────────┤
│                    QUIC Transport                    │
│  Streams | Crypto | Loss Recovery | Flow Control    │
├─────────────────────────────────────────────────────┤
│                      UDP                             │
└─────────────────────────────────────────────────────┤
```

**Each QUIC stream is independently reliable**. Lost packet in Stream 5 only blocks Stream 5. Stream 1, 3, 7 proceed.

### Connection ID (Not IP:Port)

```
Client IP:Port → Server IP:Port
     │
     ▼ Changes (WiFi → Cellular)
Connection ID: 0x1a2b3c4d  ← Stays SAME
```

**Connection migration**: Client changes network, Connection ID stays valid. No TCP re-handshake.

### 0-RTT (Zero Round-Trip Time)

```
First visit:  Full TLS 1.3 handshake (1-RTT)
Resume:       Client sends 0-RTT data immediately with ClientHello
              (encrypted with PSK from previous session)
```

**Risk**: 0-RTT data vulnerable to replay attacks. Only safe for idempotent requests (GET, HEAD).

### QPACK (HPACK Adapted for QUIC)

HPACK assumes in-order header delivery. QUIC streams can arrive out of order.

**QPACK solution**:
- **Encoder stream**: Sends dynamic table updates (in order)
- **Decoder stream**: Acknowledges receipt
- **Header blocks** reference only acknowledged entries

### TLS 1.3 Built-In

QUIC **requires** TLS 1.3. Handshake and transport merged:
```
QUIC Initial Packet = ClientHello + HTTP request (0-RTT)
QUIC Handshake Packet = ServerHello + Certificate + HTTP response
```

No separate TLS record layer overhead.

---

## Decision Framework

| Scenario | Choose |
|----------|--------|
| Simple API, broad compatibility needed | HTTP/1.1 |
| High concurrency, many resources/page | HTTP/2 |
| Mobile clients, network switching, low latency critical | HTTP/3 |
| Internal microservices (controlled env) | gRPC (HTTP/2) |
| Edge/CDN with mixed clients | HTTP/2 + HTTP/3 (ALPN negotiation) |

### ALPN Negotiation (How Client/Server Agree)

```
ClientHello: ALPN = [h3, h2, http/1.1]
ServerHello: ALPN = h3  ← Picks best mutual
```

---

## What This Means for Request/Response Timeline

| Phase | HTTP/1.1 | HTTP/2 | HTTP/3 |
|-------|----------|--------|--------|
| **T-2→T0** (Serialize) | Same | Same | Same |
| **T0→T2** (Connect) | TCP + TLS (2-RTT) | TCP + TLS (1-RTT) | QUIC (0-RTT or 1-RTT) |
| **T0→T2** (Request transfer) | Sequential per connection | Multiplexed frames | Multiplexed QUIC streams |
| **T2→T30** (Server parse) | Text scan + boundary | Binary frame parse | Binary frame parse |
| **HOL blocking** | Request level | TCP level | **None** (stream level) |
| **Header overhead** | Full every request | HPACK compressed | QPACK compressed |

---

## Vocabulary

| Term | Definition |
|------|------------|
| **HOL blocking** | Head-of-line blocking: later requests wait for earlier ones |
| **ALPN** | Application-Layer Protocol Negotiation (TLS extension) |
| **HPACK** | Header compression for HTTP/2 (static + dynamic table) |
| **QPACK** | HPACK adapted for QUIC's out-of-order delivery |
| **Connection ID** | QUIC identifier surviving IP/port changes |
| **0-RTT** | Zero round-trip resumption using PSK |
| **Stream** | Independent bidirectional flow within a connection |
| **Frame** | Atomic unit of communication in HTTP/2/3 |
| **PUSH_PROMISE** | Server announces intent to push resource |
| **WINDOW_UPDATE** | Flow control credit advertisement |

---

## Questions for Discussion

- When does HTTP/2's TCP-level HOL blocking actually hurt in production?
- Is HTTP/3's 0-RTT worth the replay risk for our APIs?
- How do load balancers/proxies affect HTTP/2 and HTTP/3 (especially connection migration)?
- What's the real-world latency difference between HTTP/2 and HTTP/3 for our workloads?
- When should we use gRPC (HTTP/2) vs REST/HTTP vs HTTP/3 for inter-service?

---

*Documented for future discussion. Covers protocol mechanics, trade-offs, and decision criteria.*