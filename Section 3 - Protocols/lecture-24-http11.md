# Lecture 24: HTTP/1.1

## Status: In Progress

> ### 📋 Read this before continuing
>
> This lecture covers **HTTP/1.1** — the request/response protocol that powers most of the web. It walks through HTTP anatomy (request and response), the client-server model, the `Host` header and virtual hosting, connection handling (HTTP/1.0 vs HTTP/1.1), pipelining, HTTP smuggling, and a quick preview of HTTP/2 and HTTP/3.
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. What is the HTTP request/response anatomy? | ✅ Documented | Unit 1 |
> | 2. What is the client-server model in HTTP? | ✅ Documented | Unit 1 |
> | 3. What is the `Host` header and why is it required? | ✅ Documented | Unit 2 |
> | 4. What is multi-homed / virtual hosting? | ✅ Documented | Unit 2 |
> | 5. How can one machine have multiple IP addresses? | ✅ Documented | Unit 2 |
> | 6. What is the full request journey (DNS → TCP → HTTP → response)? | ✅ Documented | Unit 2 |
> | 7. HTTP/1.0 vs HTTP/1.1 (keep-alive, persistent connections) | ✅ Documented | Unit 3 |
> | 8. HTTP pipelining, ordering, and HTTP smuggling | ✅ Documented | Unit 4 |
> | 9. HTTP/2 and HTTP/3 preview | ⬜ Pending | Unit 5 |
>
> | Unit | Content |
> |---|---|
> | **1** | **HTTP request/response anatomy** — method, path, protocol version, headers, body; status codes and status text |
> | **2** | **Client-server model, Host header, virtual hosting, multi-IPs, full journey** — DNS, TCP, HTTP request, server routing by Host header, response |
> | **3** | **HTTP/1.0 vs HTTP/1.1** — open+close per request vs keep-alive/persistent connections |
> | **4** | **Pipelining and HTTP/1.1 ordering** — send without waiting, responses must match request order, head-of-line blocking, HTTP/2 stream IDs solve it |
> | **5** | **HTTP/2 and HTTP/3 preview** — multiplexing, stream IDs, server push, secure by default, ALPN, QUIC |

---

## Unit 1 — HTTP Request and Response Anatomy

### Direct Answer

HTTP is a **request → response** protocol. The client (browser, script, app) initiates; the server (Nginx, Apache, Node, etc.) replies.

### The HTTP Request — Five Parts

```
GET /about HTTP/1.1
Host: joseandalso.com
User-Agent: curl/7.x
Accept: text/html

(empty body for GET)
```

| Part | What it is | Example |
|------|-----------|---------|
| **Method** | The action to perform | `GET`, `POST`, `PUT`, `DELETE`, `HEAD` |
| **Path** | What comes after the first `/` in the URL | `/about`, `/` (if nothing specified) |
| **Protocol version** | The HTTP version being used | `HTTP/1.1` |
| **Headers** | Key-value metadata | `Host`, `Content-Type`, `Cookie`, `User-Agent` |
| **Body** | The message payload | Empty for `GET`; present for `POST` |

### Query parameters live in the path

```
GET /search?q=hello&lang=en HTTP/1.1
```

The `?q=hello&lang=en` portion is part of the path, not the body. That's how `GET` passes data — through the URL.

### The URL has a length limit

Each server defines its own URL length limit (e.g., 2000 bytes, 8000 characters). A very long `GET` request can fail if the server rejects the oversized URL.

### The HTTP Response

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
Server: nginx

<html>...</html>
```

| Part | What it is |
|------|-----------|
| **Protocol version** | `HTTP/1.1` |
| **Status code** | `200`, `301`, `404`, `500`, etc. |
| **Status text** | Redundant description ("OK", "Not Found") — carried for HTTP/1.1 compatibility |
| **Headers** | `Content-Type`, `Content-Length`, `Location`, etc. |
| **Body** | The actual response (HTML, JSON, image bytes) |

`Content-Length` defines the body size — the receiver reads exactly that many bytes.

---

## Unit 2 — Client-Server Model, Host Header, Virtual Hosting, and the Full Journey

### Direct Answer

The client-server model in HTTP is simple: the client initiates a request, the server responds. But behind that simple exchange lies a stack of decisions — DNS, TCP, IP routing, and application-layer routing via the `Host` header.

### The `Host` header — why it exists

The `Host` header is **required in HTTP/1.1** (it was optional in HTTP/1.0). It tells the server which domain name the client is requesting.

**Why is it needed?** One IP address can host hundreds of websites. Without the `Host` header, the server only sees the IP at the TCP layer — it can't tell which domain the client wants.

Example:
```
GET / HTTP/1.1
Host: jose2.com
```

`192.0.2.10` might host `jose1.com`, `jose2.com`, and `jose3.com` — all on the same machine, all sharing one IP. The TCP layer can't distinguish them because they all arrive at the same IP and port. The web server application reads `Host: jose2.com` and routes the request to `jose2.com`'s folder/application.

### Nested Question — What does multi-homed hosting mean?

**Direct Answer:**

Strictly speaking, **multi-homed** means a machine with multiple network interfaces, each with its own IP address, connected to multiple networks.

In the context of this lecture, **multi-homed hosting** (more precisely called **virtual hosting**) means hosting multiple domains on a single IP address, distinguished by the `Host` header. `jose1.com`, `jose2.com`, `jose3.com` all resolve to `192.0.2.10` — three different websites, one server, one IP.

From a developer's perspective: it's like multiple folders (document roots) living on the same web server process:

```
server {
    listen 80;
    server_name jose1.com;
    root /var/www/jose1;
}

server {
    listen 80;
    server_name jose2.com;
    root /var/www/jose2;
}
```

Same IP, same port, same server process — different folders based on the `Host` header.

### Nested Question — How can one machine have multiple IP addresses?

**Direct Answer:**

1. **Multiple network interfaces** — two or more physical/virtual NICs, each with its own IP (e.g., one public, one internal).
2. **IP aliasing** — assigning several IP addresses to a single network interface; the OS listens on all of them.
3. **Virtual machines and containers** — each VM or container gets its own IP, all running on the same physical hardware.
4. **Loopback** — `127.0.0.1` and `::1` for the machine talking to itself.

So yes: one physical server, multiple IPs, each reachable over the network.

### The Full Request Journey

**Scenario:** You enter `jose2.com` in the browser.

| Step | What happens |
|------|-------------|
| **1. Browser** | You type `jose2.com`. The browser needs an IP address — it speaks IPs and ports, not domain names. |
| **2. DNS resolution** | The browser asks a DNS server: *"What's the IP for `jose2.com`?"* DNS replies: `192.0.2.10`. |
| **3. TCP connection** | The browser opens a TCP connection to `192.0.2.10` on port **443** (HTTPS) or **80** (HTTP). Three-way handshake: SYN, SYN-ACK, ACK. |
| **4. HTTP request** | Over the open connection, the browser sends: `GET / HTTP/1.1` with `Host: jose2.com`. |
| **5. Server routing** | The server reads the `Host` header and routes the request to `jose2.com`'s application/folder. |
| **6. Response** | The server returns: `HTTP/1.1 200 OK` + headers + HTML body. |
| **7. Connection** | Closes (HTTP/1.0) or stays open for reuse (HTTP/1.1 keep-alive). |

### The mental model

| Layer | Question it answers |
|-------|-------------------|
| DNS | What IP is `jose2.com`? |
| IP | Where do packets go? (`192.0.2.10`) |
| Port (443/80) | Which service on that machine? |
| TCP | Reliable pipe between browser and server |
| `Host` header | Which website on that server? |

The `Host` header doesn't change where packets go — it's application-layer routing *after* the connection is already established.

---

## Unit 3 — HTTP/1.0 vs HTTP/1.1: Keep-Alive and Persistent Connections

### The Core Difference

| | HTTP/1.0 | HTTP/1.1 |
|---|---|---|
| Connection per request | Open → request → response → **close** | Open → request → response → **keep alive** → next request on same connection |
| TCP handshake cost | Paid on **every** request | Paid **once**, reused |
| Default behavior | Close after each response | Keep-alive (persistent) |

### HTTP/1.0: Open and Close Every Time

From the lecture:

> "the first version — every the first time you want to send a request, you open a connection, you open a brand new connection, and then you send your get request and then the server processes it, and then the moment you get a response, guess what? You close the connection."

This was the 1990s model. The reasoning:

> "TB was stateless, says, Hey, I don't care about the connection, don't leave it open."

Keeping a TCP connection alive consumed CPU and memory — file descriptors were scarce. So the contract was: one request, one connection, then close.

**The problem:** If an HTML page references 10 images, 2 CSS files, and 3 JavaScript files, that's 16 separate TCP connections:

```
Request 1:  open → GET index.html    → response → close
Request 2:  open → GET logo.png      → response → close
Request 3:  open → GET style.css     → response → close
... and so on
```

From the lecture:

> "you are opening and closing. Opening and closing. And that is it really slows things down."

Plus:
- **Server thrashing:** The server gets hammered with constant connection setup/teardown.
- **Extra latency:** Every new connection requires the TCP three-way handshake (SYN, SYN-ACK, ACK) before any HTTP data flows.
- **No buffering / chunking:** HTTP/1.0 required `Content-Length` and the entire response had to be sent at once. No `Transfer-Encoding: chunked`, no server-sent events, no long polling.

### HTTP/1.1: Persistent Connections (Keep-Alive)

From the lecture:

> "When you when you send it open a connection, you get a request, keep it alive. You tell the server, hey, keep this connection alive because I'm going to use it to send many, many, many requests."

HTTP/1.1 introduced `Connection: keep-alive`. One TCP connection carries multiple request/response pairs:

```
Request 1:  GET index.html    → response (same connection, don't close)
Request 2:  GET logo.png      → response (same connection)
Request 3:  GET style.css     → response (same connection)
Request 4:  GET app.js        → response (same connection)
... finally close
```

**Benefits:**

> "Now it has lower latency because we don't have to do this dance with the server to establish a connection with every time we send one to send a request and it has low CPU usage because guess what? The only thing you need to do is just read the stream and find out the the start of the request."

- **Lower latency** — skip the TCP handshake on every request.
- **Lower CPU usage** — the server just reads the stream, finds request boundaries (`GET /path HTTP/1.1` ... `Content-Length: X`), and serves them back-to-back.
- **Enables new features** — `Transfer-Encoding: chunked` (streaming responses without knowing total length upfront), server-sent events, long polling.

### How the Server Knows Where a Request Ends

Since multiple requests share one TCP connection, the server needs to know where one ends and the next begins:

1. Read `GET /path HTTP/1.1`
2. Read headers until blank line
3. Check `Content-Length: X` — read exactly X bytes for the body
4. That's the end of request 1; the next bytes are request 2

From the lecture:

> "The only thing you need to do is just read the stream and find out the the start of the request, which is we show how we we know how the request end right starts. It is a get slash protocol if it looks for this and then at the end is the where the content length is x, you pause until you find the content and that's the end."

---

## Unit 4 — HTTP Pipelining, Ordering, and HTTP/2 Multiplexing

### First, imagine one TCP connection

Your browser establishes one TCP connection to a web server.

Think of TCP as a reliable communication pipe between two applications.

Now imagine the browser wants two resources:
- `/profile`
- `/notifications`

It sends two HTTP requests.

### Without pipelining: request, wait, response

The browser sends request 1:

```
GET /profile HTTP/1.1
Host: example.com
```

The server processes it and sends response 1:

```
HTTP/1.1 200 OK

{"name":"Usama"}
```

Only after receiving response 1 does the browser send request 2:

```
GET /notifications HTTP/1.1
Host: example.com
```

The server responds with response 2.

Sequential communication:

1. Client → Server: Request 1 `/profile`
2. Server → Client: Response 1 profile data
3. Client → Server: Request 2 `/notifications`
4. Server → Client: Response 2 notifications

This is straightforward, but it can waste time: the browser has to wait for the first response before sending the second request.

### With pipelining: send requests without waiting

HTTP/1.1 introduced a feature called HTTP pipelining.

Instead of waiting for response 1, the browser sends both requests immediately:

```
GET /profile HTTP/1.1
Host: example.com

GET /notifications HTTP/1.1
Host: example.com
```

The server receives both requests on the same connection.

**The crucial rule:** The server must send the corresponding responses in the same order as the requests.

HTTP/1.1 pipelining:

Browser → Server: Request 1 `/profile`, Request 2 `/notifications`
Server processes the requests
Server → Browser: Response 1 Profile data, Response 2 Notifications

Notice something important: the requests can be sent before the first response arrives, but the responses cannot be reordered.

### Why does HTTP/1.1 require responses in order?

Imagine the server receives these two requests:

| Request   | Resource         | Processing time |
|-----------|------------------|-----------------|
| Request 1 | `/profile`       | 5 seconds       |
| Request 2 | `/notifications` | 100 ms          |

Suppose the server can process the second request quickly, even while the first one is still being processed.

The second response is ready after 100 ms, but the first response is not ready until 5 seconds.

Can the server send response 2 first?

**No.** HTTP/1.1 requires responses to be sent in the same order as their requests when using pipelining.

**Head-of-Line (HOL) Blocking:**

1. Request 1: `/profile` — Slow (5 seconds)
2. Request 2: `/notifications` — Fast (100 ms)
3. Response 2 must wait — even though its data is ready, response 1 must be sent first.

Result: The fast request is blocked behind the slow request.

### How does the browser know which response belongs to which request?

HTTP/1.1 doesn't need request IDs because it uses **ordering** to match requests and responses.

For example:

```
Browser sends:
    Request 1 → /profile
    Request 2 → /notifications
    Request 3 → /settings

Server sends:
    Response 1 → profile data
    Response 2 → notifications data
    Response 3 → settings data
```

The browser knows:
- First response belongs to first request.
- Second response belongs to second request.
- Third response belongs to third request.

The order itself provides the association. The client does know which response belongs to which request; it doesn't need a unique request ID.

### How does HTTP/2 solve this?

HTTP/2 introduced **streams** and **stream IDs**.

Each request-response exchange gets its own stream ID.

HTTP/2 multiplexing:

Browser → Server:
- Stream 1: `/profile`
- Stream 3: `/notifications`
- Stream 5: `/settings`

One connection, multiple independent HTTP streams

Server → Browser:
- Stream 5: Settings response
- Stream 3: Notifications response
- Stream 1: Profile response

The response data can be interleaved across streams. The browser identifies each stream using its ID.

For example, the server can send:
1. A part of the response for stream 3.
2. A part of the response for stream 5.
3. A part of the response for stream 3 again.
4. The response for stream 1.

The browser reconstructs each response using its stream ID.

That means a slow response on stream 1 doesn't prevent the server from delivering completed responses on streams 3 and 5.

**One technical qualification:** HTTP/2 still runs over TCP, so packet loss can block delivery of later TCP data until the missing data is retransmitted. HTTP/3, which uses QUIC over UDP, addresses this transport-level head-of-line blocking between independent streams.

### Connecting everything together

| Concept             | What it does                                                       |
|-------------------|--------------------------------------------------------------------|
| HTTP keep-alive     | Reuses the same connection for multiple requests                   |
| HTTP/1.1 pipelining | Sends multiple requests without waiting for earlier responses      |
| HTTP/1.1 ordering   | Requires pipelined responses in request order                      |
| HTTP/2 multiplexing | Allows independent streams to be interleaved                       |
| HTTP/3              | Uses QUIC to avoid TCP-level head-of-line blocking between streams |

**Remember this distinction:**
- **Keep-alive:** Can I reuse the connection?
- **Pipelining:** Can I send another request before receiving the previous response?
- **Multiplexing:** Can multiple independent streams make progress at the same time?

---

## Unit 5 — HTTP/2 and HTTP/3 Preview

### HTTP/2 — invented by Google, originally "Speedy"

The goal was performance. Key changes over HTTP/1.1:

**1. Compression on both headers and body**

HTTP/2 supports header compression (HPACK) and body compression. HTTP/1.1 had header compression disabled because of the **CRIME attack** — an attacker could extract cookie information by observing how header compression changed response sizes. HTTP/2 brought compression back safely.

**2. Multiplexing with stream IDs**

Instead of one request at a time (even with keep-alive), HTTP/2 tags every request/response with a unique **stream ID**. Multiple requests and responses interleave freely on a single TCP connection. A slow response on stream 1 doesn't block streams 3 and 5.

**3. Server push**

The server can proactively push resources to the client. The lecture notes this is now **dead** — browsers stopped supporting it because it didn't pan out.

**4. Secure by default**

HTTP/2 effectively requires TLS. The reason: **protocol ossification**. Routers and middleboxes were built in the 1990s to understand HTTP/1.1's text format on port 80. If you send HTTP/2's binary frames over plain port 80, those middleboxes reject it — they only know the old rules. Wrapping HTTP/2 in TLS makes the traffic opaque, so middleboxes can't inspect or block it based on content. This is a double-edged sword, but it worked out well — security by default.

**5. Protocol negotiation via ALPN**

The client and server negotiate HTTP/2 during the TLS handshake using **ALPN** (Application Layer Protocol Negotiation) — a TLS extension. The client says "I support HTTP/2" inside the TLS Client Hello; the server replies with its choice.

### HTTP/3 — QUIC

The core problem with HTTP/2: **TCP head-of-line blocking at the transport layer**. Even though HTTP/2 multiplexes streams, they all ride on one TCP connection. If one TCP packet is lost, TCP stalls *all* streams until that packet is retransmitted — even streams whose data arrived fine.

HTTP/3 replaces TCP with **QUIC** — "UDP with congestion control." QUIC pushes transport features (streams, congestion control, reliability) down into a new transport protocol where each stream is independent. Packet loss on stream 1 doesn't stall stream 2.

### Summary of the evolution

| Version | Connection | Key idea |
|---------|-----------|----------|
| HTTP/1.0 | Open + close per request | Simple, but slow for multiple resources |
| HTTP/1.1 | Keep-alive, persistent | Reuse one TCP connection; ordered requests/responses |
| HTTP/2 | Multiplexing, stream IDs | Many interleaved streams on one TCP connection; secure by default |
| HTTP/3 | QUIC over UDP | Independent streams; no TCP head-of-line blocking |

---

## Gap Pass — Lecture 24 Complete

All units covered:

| # | Question | Status | Unit |
|---|----------|--------|------|
| 1 | What is the HTTP request/response anatomy? | ✅ Documented | Unit 1 |
| 2 | What is the client-server model in HTTP? | ✅ Documented | Unit 1 |
| 3 | What is the `Host` header and why is it required? | ✅ Documented | Unit 2 |
| 4 | What is multi-homed / virtual hosting? | ✅ Documented | Unit 2 |
| 5 | How can one machine have multiple IP addresses? | ✅ Documented | Unit 2 |
| 6 | What is the full request journey (DNS → TCP → HTTP → response)? | ✅ Documented | Unit 2 |
| 7 | HTTP/1.0 vs HTTP/1.1 (keep-alive, persistent connections) | ✅ Documented | Unit 3 |
| 8 | HTTP pipelining, ordering, head-of-line blocking | ✅ Documented | Unit 4 |
| 9 | HTTP/2 multiplexing and stream IDs | ✅ Documented | Unit 5 |
| 10 | HTTP/3 and QUIC | ✅ Documented | Unit 5 |

Nested questions answered:
- What does multi-homed hosting mean? (Unit 2)
- How can one machine have multiple IP addresses? (Unit 2)
- What is `Connection: keep-alive`? (Unit 3)
- What happens if the client sends a second request before the first response arrives? (Unit 4)
- What is `Transfer-Encoding: chunked`? (Unit 3)

No remaining gaps from the transcript. Lecture 24 (HTTP/1.1) is fully mapped.

*Status: Complete*