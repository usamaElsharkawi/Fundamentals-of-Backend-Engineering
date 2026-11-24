# Lecture 26: WebSockets

## Status: Complete

> ### 📋 Read this before continuing
>
> This lecture covers **WebSockets** — a bidirectional communication protocol designed for the web. It explains why raw TCP can't be exposed to browsers, how the WebSocket handshake upgrades an HTTP connection, what full-duplex means compared to HTTP request/response, the pros and cons (firewall-friendly, proxy issues, scaling challenges), and the rule of thumb for when to use (and not use) WebSockets.
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. Why do we need WebSockets? Why can't the browser just use raw TCP? | ✅ Documented | Unit 1 |
> | 2. How does the WebSocket handshake work (HTTP → WebSocket upgrade)? | ✅ Documented | Unit 2 |
> | 3. What is bidirectional/full-duplex communication vs HTTP request-response? | ✅ Documented | Unit 3 |
> | 4. What are the pros and cons of WebSockets? | ✅ Documented | Unit 4 |
> | 5. When should you NOT use WebSockets? The rule of thumb | ✅ Documented | Unit 5 |
>
> | Unit | Content |
> |---|---|
> | **1** | **Why WebSockets?** — TCP is bidirectional but too dangerous to expose to browsers; HTTP as the protective umbrella |
> | **2** | **The WebSocket handshake** — HTTP request with `Upgrade` header, `101 Switching Protocols`, Sec-WebSocket-Key/Accept |
> | **3** | **Bidirectional/full-duplex** — both sides send anytime vs HTTP's strict request/response |
> | **4** | **Pros and cons** — firewall friendly, no polling, vs proxy complexity, idle connection drops, stateful, hard to scale |
> | **5** | **Rule of thumb** — only use if you absolutely need bidirectional; alternatives: long polling, SSE |

---

## Unit 1 — Why Do We Need WebSockets? Why Not Raw TCP?

### Direct Answer

TCP **is** bidirectional. A TCP connection lets both sides send and receive data freely, anytime. So why doesn't the browser just expose raw TCP sockets to JavaScript?

**Because it would be a security disaster.**

### What raw TCP in the browser would enable

From the lecture:

> "Technically speaking, if you think about it, you guys, we have TCP as a bi directional protocol. Why don't we just expose that to the web? Bad idea. It was TCP. TCP does so much... you can connect to your server, you can connect to a SMTP server, you can connect to so many other services. So it's very dangerous to expose it into a browser. Effectively, anyone with a link can can get access to that JavaScript code."

If any website's JavaScript could open a raw TCP connection to any IP address and any port:

1. **Internal network scanning** — A malicious site you visit could have your browser scan your home or corporate network (`192.168.x.x`, `10.x.x.x`) for open ports — printers, databases, admin panels, IoT devices.

2. **Service abuse** — Your browser could connect directly to an SMTP server (port 25) and send spam through your network. Or to a Redis server, a database, an internal API — anything reachable from your machine.

3. **Cross-service attacks** — JavaScript could speak SMTP, FTP, or custom protocols to internal services that were never meant to be exposed to the web.

### Why HTTP is the protective umbrella

Browsers restrict JavaScript to HTTP (and HTTPS). HTTP is a request-response protocol with clear boundaries:
- The client initiates
- The server responds
- The connection is transactional

This limits what a random website can do to "make HTTP requests to servers." It's still not perfectly safe (CSRF, SSRF exist), but it's a known, bounded attack surface.

### What WebSockets do instead

WebSockets don't expose raw TCP. They expose a **controlled, HTTP-mediated upgrade** to a bidirectional socket:

1. It starts as a normal HTTP request (so normal CORS, cookies, TLS, and firewall rules apply)
2. It uses a special `Upgrade` header to ask the server: "Can we switch this connection to WebSocket?"
3. Only if the server agrees (`101 Switching Protocols`) does the connection become bidirectional

This means the server **opts in**. A server that doesn't understand WebSockets will just reject the upgrade and serve normal HTTP. You can't trick a random SMTP server into becoming a WebSocket endpoint.

### The mental model

| | Raw TCP in browser | WebSocket |
|---|---|---|
| Who initiates | Any JavaScript | Client via HTTP request |
| Server cooperation needed | No — any TCP port | Yes — server must accept the upgrade |
| Protocols reachable | SMTP, FTP, DB, anything | Only WebSocket-aware servers |
| Firewall friendly | No — arbitrary ports | Yes — port 80/443 |
| HTTP security layer (CORS, cookies, TLS) | Bypassed | Preserved |

---

## Unit 2 — The WebSocket Handshake: HTTP → WebSocket Upgrade

### The key insight

A WebSocket connection **starts as HTTP**. It's not a separate protocol that replaces HTTP from the beginning. It's an HTTP request that asks the server: *"Can we upgrade this connection to WebSocket?"* If the server agrees, the same TCP connection stops speaking HTTP and starts speaking WebSocket frames.

### Step 1: The client sends an HTTP request with upgrade headers

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat
```

| Header | Purpose |
|--------|---------|
| `Upgrade: websocket` | Tells the server the client wants to switch protocols |
| `Connection: Upgrade` | Confirms this is a hop-by-hop upgrade request (proxies must forward it) |
| `Sec-WebSocket-Key` | A base64-encoded random 16-byte nonce. Prevents caching/proxy issues and proves the client knows WebSocket |
| `Sec-WebSocket-Version: 13` | The WebSocket protocol version |
| `Sec-WebSocket-Protocol` | Optional subprotocol (e.g., "chat", "superchat"). Tells the server which application-level protocol to use |

From the lecture:

> "you add an upgrade header... And upgrade header is a special header that can be understood by the server effectively... we're going to upgrade the socket to a socket protocol... sub protocols is useful... for the back end itself to understand that, hey, I actually want the chat..."

### Step 2: The server responds with `101 Switching Protocols`

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

**`101 Switching Protocols`** is the HTTP status code that means: *"I agree to stop speaking HTTP on this connection."*

**`Sec-WebSocket-Accept`** is computed by the server:
1. Take the client's `Sec-WebSocket-Key`
2. Append the magic GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`
3. SHA-1 hash the result
4. Base64-encode it

The client verifies this value. If it matches, the client trusts that the server genuinely supports WebSocket.

From the lecture:

> "the server respond with this which says, Hey Steve 11101 switching protocol... And this is the code... Hey, we're good. We upgraded all of them..."

### Step 3: The connection becomes WebSocket — "the wild, wild west"

After the `101` response, **HTTP is gone** on this connection. No more request/response. No more `GET`, `POST`, headers, or status codes.

Both sides can now send **WebSocket frames** to each other at any time:
- Client → Server
- Server → Client
- Both simultaneously (full duplex)

From the lecture:

> "Then it's the wild, wild, wild West. The server can send us information. The client can send something to the server. The client can send something again. The server can send back. The client can send that. Both of them can send together."

### The security key purpose

From the lecture:

> "the key is basically some operation that's done on the server that can use that key to ensure that not anyone can just upgrade their connection. They have to know what they're doing."

The `Sec-WebSocket-Key` / `Sec-WebSocket-Accept` dance prevents:
- A cache or proxy from returning a stale WebSocket response
- A non-WebSocket server from being tricked into a WebSocket connection
- Accidental protocol upgrades

---

## Unit 3 — Bidirectional/Full-Duplex Communication vs HTTP Request-Response

### HTTP: Request-Response (Half-Duplex in spirit)

HTTP is strictly **transactional**:

1. Client sends a request
2. Server sends exactly one response
3. That's it for that exchange

The client always initiates. The server **cannot** send data to the client unless the client asked first. There is no way for the server to say *"hey, I have new data for you"* out of the blue.

If you want real-time updates in pure HTTP, you have to work around this:
- **Polling:** Client asks "anything new?" every N seconds. Most answers are "no." Wasteful.
- **Long polling:** Client asks, server holds the connection open until data arrives, then responds. Client immediately asks again. Better, but still one request at a time.
- **Server-Sent Events (SSE):** Server can push to client, but client can't send back on the same channel. One-way only.

### WebSocket: Full-Duplex (Bidirectional)

After the upgrade handshake, both sides are equal peers on the same TCP connection:

```
Client ◄── WebSocket frames ──► Server
```

- Client can send a message whenever it wants
- Server can send a message whenever it wants
- Both can send simultaneously
- No request/response pairing. A message from the client doesn't "expect" a reply from the server.

From the lecture:

> "Then it's the wild, wild, wild West. The server can send us information. The client can send something to the server. The client can send something again. The server can send back. The client can send that. Both of them can send together."

### The concrete mental model

| | HTTP | WebSocket |
|---|------|-----------|
| Who speaks first | Client always | Either side |
| Server initiates? | No | Yes |
| Multiple messages per connection | One request → one response | Many messages, any direction |
| Connection purpose | Fetch a resource | Maintain an ongoing conversation |
| Analogy | Sending letters | Phone call |

### Why this matters for real-time apps

**Chat:** User A sends a message → server → pushes to User B instantly. User B didn't ask for it.

**Live feed / stock ticker:** Server pushes price updates as they happen. Client doesn't need to keep asking.

**Multiplayer gaming:** Player A moves → server → broadcasts to Player B, C, D immediately. No turn-taking.

---

## Unit 4 — Pros and Cons of WebSockets

### Pros

**1. Full-duplex / bidirectional**
Both sides can send whenever they want. No request/response turn-taking.

**2. No polling**
You don't need to ask the server "anything new?" every few seconds. The server pushes data the moment it has it.

**3. Firewall friendly**
WebSockets run over port 80 (HTTP) or 443 (HTTPS). Firewalls already allow these ports. You don't need to open a custom TCP port that might be blocked.

From the lecture:

> "if you use TCP, then you are going to build your own port, right? And this port might be blocked in the server side. That's why a lot of people use of socket because it's firewall friendly."

**4. Compatible with existing web infrastructure**
It starts as HTTP, so cookies, TLS, and CORS all work normally.

---

### Cons

**1. Proxies are tricky**
A Layer 7 (HTTP) proxy doesn't naturally know how to handle a WebSocket connection. It has to:
- Understand the upgrade handshake
- Terminate the WebSocket connection
- Read the message
- Establish a *new* WebSocket connection to the destination server
- Forward the message

From the lecture:

> "proxy at layer seven or socket is going to be really, really tricky because now you get, you have to really understand how to terminate the socket connection and then read the message and then turn around and establish another web socket connection to the destination server"

**2. Middle routers drop idle connections**
TCP connections sitting idle for a long time may be dropped by middleboxes (routers, firewalls, load balancers). WebSockets are long-lived, so you need **ping/pong heartbeats** to keep them alive.

From the lecture:

> "Some proxies will sometimes terminate the connection if you don't send anything because they say, Hey, why is this connection being sitting there for a long time?... The problem is that socket connections can be closed by either party... or some middle routers. The middle routers can just drop the knowledge of a TCP connection."

**3. Stateful**
Both client and server maintain connection state. You can't treat each message as an independent transaction like HTTP.

**4. Hard to horizontally scale**
With Layer 4 load balancing, each connection is pinned to one backend server. If you have three chat servers behind a load balancer, User A on Server 1 can't easily message User B on Server 2 — they're on different persistent connections.

From the lecture:

> "it's just difficult to horizontally scale, but you can do layer four load balancing... that's really dumb load balancing because it just based on the connection, you just each connection can never be used to do anything else except that thing."

---

## Unit 5 — When Should You NOT Use WebSockets? The Rule of Thumb

### The rule

From the lecture:

> "The rule of thumb is just do you absolutely need bidirectional communication? I would go with long polling or service interval... If you don't need it, don't use it."

### Why WebSockets are "extra stuff"

WebSockets aren't free. Compared to plain HTTP:

- **Extra headers** — The upgrade handshake, Sec-WebSocket-Key, subprotocols, ping/pong frames
- **Connection state** — Both sides must track open connections, handle disconnects, clean up dead connections
- **Operational complexity** — Proxies, load balancers, firewalls, horizontal scaling all get harder

From the lecture:

> "popsockets is having it to make keep it plus socket, have extra headers and stuff like that that needs to be managed. And yeah, it's just extra stuff."

### The alternatives

**Long polling / service interval**
Client asks "anything new?" periodically. Server holds the connection open until data arrives (long polling) or responds immediately with "no" (short polling). It's just HTTP — no upgrade, no persistent socket state, works through any proxy.

> "long component is just HTTP at the end of the day"

**Server-Sent Events (SSE / "recent events")**
Server pushes to client over a single HTTP connection. One-way only (server → client), but simpler than WebSockets if you don't need the client to send back on the same channel.

> "Another good alternative. But again, it's not bi directional per se."

### Decision framework

| You need | Use |
|----------|-----|
| Request/response (fetch data, submit forms) | Plain HTTP/1.1 or HTTP/2 |
| Server → client push only (stock ticker, notifications) | SSE or long polling |
| True bidirectional (chat, multiplayer games, collaborative editing) | WebSocket |
| Not sure | Start with HTTP/SSE, upgrade to WebSocket only if you hit a wall |

### A concrete example from the lecture

The lecturer built a chat app where multiple browsers connect to a Node.js WebSocket server. When one user sends a message, the server loops through all connections and broadcasts to everyone. That's genuinely bidirectional — every client both sends and receives on the same persistent connection. That's the right use case.

But if you were building, say, a news feed where users just read articles and occasionally click "like" — WebSocket would be overkill. Plain HTTP suffices.

---

## Gap Pass — Lecture 26 Complete

All units covered:

| # | Question | Status | Unit |
|---|----------|--------|------|
| 1 | Why WebSockets? Why not raw TCP? | ✅ Documented | Unit 1 |
| 2 | How does the WebSocket handshake work? | ✅ Documented | Unit 2 |
| 3 | Bidirectional/full-duplex vs HTTP request-response? | ✅ Documented | Unit 3 |
| 4 | Pros and cons of WebSockets? | ✅ Documented | Unit 4 |
| 5 | When NOT to use WebSockets? Rule of thumb | ✅ Documented | Unit 5 |

No remaining gaps from the transcript. Lecture 26 (WebSockets) is fully mapped.

*Status: Complete*