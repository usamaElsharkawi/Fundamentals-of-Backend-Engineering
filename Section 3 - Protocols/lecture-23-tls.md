# Lecture 23: Transport Layer Security (TLS)

## Status: Complete

> ### 📋 Read this before continuing
>
> This lecture covers **TLS** — the cryptographic protocol that encrypts communication between two parties, most commonly HTTP over TLS (HTTPS). The lecturer frames TLS as a standard we need because we cannot send plaintext IP packets that anyone in the network path can read.
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. Why do we need to encrypt communication? | ✅ Documented | Unit 1 |
> | 2. Why do we need a standard for that? | ✅ Documented | Unit 2 |
> | 3. How does vanilla HTTP work (without encryption)? | ✅ Documented | Unit 3 |
> | 4. What does TLS add to HTTP (HTTPS and the handshake)? | ✅ Documented | Unit 4 |
> | 5. TLS 1.2 handshake: RSA key exchange | ✅ Documented | Unit 5 |
> | 6. TLS 1.2 with Diffie-Hellman and forward secrecy | ✅ Documented | Unit 6 |
> | 7. TLS 1.3: 1-RTT vs 2-RTT, 0-RTT | ✅ Documented | Unit 7 |
> | 8. Server authentication via certificates and PKI | ✅ Documented | Unit 8 |
>
> | Unit | Content |
> |---|---|
> | **1** | **Why do we need to encrypt communication?** — plaintext IP packets are readable by every router, ISP, and middlebox on the path |
> | **2** | **Why do we need a standard?** — interoperability, negotiation, public scrutiny of algorithms, shared PKI trust infrastructure |
> | **3** | **How does vanilla HTTP work?** — TCP handshake on port 80, HTTP request/response, segments/packets/frames, slow start congestion control |
> | **4** | **HTTPS — HTTP over TLS** — port 80 vs 443, two handshakes (TCP then TLS), what changes at each layer |
> | **5** | **TLS 1.2 with RSA** — 2 roundtrips, client encrypts pre-master with server's public RSA key, no forward secrecy |
> | **6** | **TLS 1.2 with Diffie-Hellman** — key exchange without transmitting the key, forward secrecy |
> | **7** | **TLS 1.3** — single roundtrip (or 0-RTT with PSK), DH mandatory, RSA key exchange removed |
> | **8** | **Certificates and PKI** — server authentication, public/private key pairs, certificate authorities, Heartbleed |

---

## Unit 1 — Why Do We Need to Encrypt Communication?

### Direct Answer

We encrypt because **raw IP packets are plaintext**. Every router, ISP, switch, and any device sitting on the network path between client and server can read the full contents of every packet — URLs, headers, cookies, passwords, HTML bodies, everything. Without encryption, communication is effectively a postcard that any mail handler can open and read.

### The threat model

From the lecture:

> "We can't send just IP packets with data that is plain, you know, text that everybody in the crowd can read."

When you send an HTTP request over an unencrypted connection:

1. Your browser breaks the request into TCP segments
2. Each segment wraps into an IP packet
3. That packet traverses routers, ISPs, and middleboxes
4. **Every hop can read the application data** — Layers 1–4 (physical, data link, network, transport) are never encrypted. Only the application payload needs protection.

### What gets exposed without encryption

| What | Visibility without TLS |
|------|----------------------|
| Source/destination IP | Visible to every router |
| Source/destination port | Visible to every router/firewall |
| HTTP method + path (`GET /login`) | Visible to every middlebox |
| HTTP headers (cookies, auth tokens) | Visible to every middlebox |
| Response body (HTML, JSON, user data) | Visible to every middlebox |

### What encryption buys us

Only the two endpoints (client and server) can read the application data. Anyone in the middle sees encrypted ciphertext — mathematically unreadable without the shared key. This is the core problem TLS solves.

---

## Unit 2 — Why Do We Need a Standard for That?

### Direct Answer

We need a standard so that **any client can securely talk to any server** without pre-arranged, custom encryption schemes. Without a standard, every vendor would invent its own protocol, and nothing would interoperate.

### The four reasons

**1. Interoperability**

Imagine Amazon uses "Encryption Method A," your bank uses "Method B," and a random blog uses "Method C." Your browser wouldn't know which method to use for which site. Every HTTPS connection would require custom code on both ends. The internet would fragment into isolated silos.

A standard like TLS gives every browser and every server the same rulebook. Chrome can talk to Nginx, Safari to Apache, curl to IIS — they've never met, but they speak the same TLS language.

**2. Negotiation**

Because there's a standard, client and server can agree on:
- Which TLS version to use (1.2 vs 1.3)
- Which key exchange algorithm (RSA, Diffie-Hellman, Elliptic Curve)
- Which symmetric cipher (AES, ChaCha20)

When a new, faster algorithm appears, the standard gets updated and everyone adopts it together.

**3. Public scrutiny**

Standards are published and analyzed by thousands of security researchers. When you "roll your own crypto," you almost certainly introduce subtle flaws attackers exploit. A public standard means flaws get found and fixed openly.

**4. Trust infrastructure (PKI)**

The standard defines how certificates work, how certificate authorities are trusted, and how a server proves its identity. Without that shared framework, you'd have no way to know if `bank.com` is really your bank or an imposter.

### Without TLS as a standard

The web couldn't scale. You couldn't safely log into any site you hadn't pre-arranged encryption with. The standard is what makes secure communication universal.

---

## Unit 3 — How Does Vanilla HTTP Work (Without Encryption)?

### Direct Answer

Vanilla HTTP is **HTTP over plain TCP**, typically on **port 80**. No encryption, no TLS. The client opens a TCP connection, sends an HTTP request, the server responds, and the connection closes — all in plaintext readable by anyone on the path.

### Step 1: The server listens on port 80

The server sits waiting on port 80 — the default "loading dock" for unencrypted HTTP. Your browser wants to fetch `http://example.com/`.

### Step 2: The TCP three-way handshake (Layer 4)

Before any HTTP data flows, client and server establish a reliable TCP connection:

1. **Client → Server:** `SYN` — "I want to start a connection."
2. **Server → Client:** `SYN-ACK` — "I hear you, I'm ready too."
3. **Client → Server:** `ACK` — "Got it. Connection open."

Now we have an **open TCP connection**. It is stateful — both sides remember they're talking to each other. File descriptors, sequence numbers, window sizes — all that session state lives here at the transport layer.

> From the lecture: *"We opened a TCP connection. So we have established the sense and accent, right?"*

### Step 3: The client sends an HTTP request

Over the open connection, the client sends an HTTP request, for example:

```
GET / HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html

```

- `GET` — the method (fetching data)
- `/` — the path
- `HTTP/1.1` — the version
- Headers — metadata about the client and what it accepts
- Body — usually empty for a `GET`

The request is not sent as one giant blob. It gets broken into **TCP segments**. The number depends on the **MTU** (Maximum Transmission Unit) — the largest chunk a single network frame can carry. Small requests fit in one segment; larger ones span multiple.

### Step 4: Moving down the stack

Each TCP segment gets wrapped into an **IP packet**:

```
[ IP Header ][ TCP Header ][ HTTP Data ]
```

The IP packet carries the source IP (your machine) and destination IP (the server). It travels through routers, switches, ISPs — every hop — until it reaches the server.

### Step 5: The server receives and processes

At the server, the process unwinds in reverse:

1. Network card receives IP packets
2. IP layer strips the IP header, passes TCP segments up
3. TCP layer reassembles segments in order (using sequence numbers), checks for errors, acknowledges receipt
4. The complete HTTP request reaches the **application layer** — the web server software (Nginx, Apache, etc.)

The web server reads the request: *"Ah, they want `/` using HTTP/1.1."*

### Step 6: The server responds

The server sends back an HTTP response:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
Server: nginx

<html>...index.html content...</html>
```

- Status line: `200 OK` means success
- Headers: content type, length, server software
- Body: the actual content — `index.html`, an image, JSON, whatever

Like the request, the response gets chopped into **one or more TCP segments** depending on size and congestion.

### Step 7: TCP congestion control and slow start

The server does not necessarily blast out all response segments at once. TCP uses **congestion control** to avoid clogging the network.

At the beginning of a connection, TCP uses **slow start** — it sends a small number of segments, waits for acknowledgments, then gradually increases how much it sends. It's like testing the water before diving in. The response might dribble out segment by segment at first, then speed up.

### Step 8: Closing the connection

Once the full response is sent, the connection is closed. HTTP/1.1 can keep the connection alive for reuse (`Connection: keep-alive`), but eventually it gets torn down with a TCP four-way handshake.

---

## Nested Question — What does slow start actually look like in practice?

### Direct Answer

TCP **slow start** is how a new connection figures out how much data the network can handle without flooding it. It starts conservative and ramps up quickly.

**The variables:**
- **cwnd** (congestion window): how many bytes/segments the sender is allowed to have "in flight" (unacknowledged) at once
- **MSS** (Maximum Segment Size): the largest chunk TCP will send in one segment, typically ~1,460 bytes
- **RTT**: round-trip time — how long until an ACK comes back

**A concrete walkthrough:**

Assume modern TCP starts with `cwnd = 10 MSS` (~14,600 bytes) and an RTT of 100 ms.

- **RTT 1:** Sender transmits 10 segments. Waits for ACKs. When all 10 ACKs come back, `cwnd` increases by 1 MSS per ACK → now `cwnd = 20 MSS`.
- **RTT 2:** Sender transmits 20 segments. Gets 20 ACKs. `cwnd` doubles again → now `cwnd = 40 MSS`.
- **RTT 3:** Sender transmits 40 segments. `cwnd` → `80 MSS`.

So slow start grows **exponentially** — doubling every round trip — until one of these happens:

1. `cwnd` hits **ssthresh** (slow start threshold), then TCP switches to **congestion avoidance** (linear growth: +1 MSS per RTT instead of doubling).
2. A segment is **lost** (timeout or duplicate ACKs), then `cwnd` collapses — usually cut in half — and slow start restarts from a smaller number.
3. The **receiver's advertised window** is smaller, capping how much can be sent regardless.

**Why it matters for this lecture:** This is exactly why the server doesn't dump the whole `index.html` response in one burst. It dribbles out segments at first, watches for ACKs, and only speeds up once it knows the path can handle it. A tiny HTML file might finish in the first RTT; a 16 KB JavaScript file might take 3–4 round trips just to get moving.

---

### The problem vanilla HTTP sets up

Here's the critical point from the lecture: **every step above is unencrypted.**

That `GET / HTTP/1.1` request? Plain text. The `index.html` response? Plain text. Any router, ISP, or malicious actor sitting on the network path can read every byte.

| Layer | What's visible without TLS |
|-------|---------------------------|
| Application | Full HTTP request and response — URLs, cookies, passwords, HTML |
| Transport | TCP headers (ports, sequence numbers) |
| Network | IP headers (source/destination IPs) |

That's why we need TLS. The lecture's next step: *"We open a connection, we do a handshake first. And the goal of this handshake is to share a symmetric key..."*

---

## Unit 4 — HTTPS: HTTP Over TLS

### Direct Answer

**HTTPS = HTTP + TLS.** It's the exact same HTTP protocol — same methods (GET, POST), same headers, same status codes, same request/response structure. The only difference is that the entire HTTP conversation travels **inside an encrypted TLS tunnel** instead of riding naked over plain TCP.

Think of it like this:
- **Vanilla HTTP:** You send a postcard through the mail. Anyone handling it can read the message.
- **HTTPS:** You put that same postcard inside a locked safe, send the safe, and only the recipient has the key. The message inside is unchanged; the container is now armored.

### Port 80 vs. Port 443

| Protocol | Default Port | Encrypted? |
|----------|-------------|------------|
| HTTP | 80 | No |
| HTTPS | 443 | Yes |

When you type `https://example.com`, your browser connects to **port 443** on the server, not port 80. The server listens on port 443 specifically for TLS-wrapped connections.

### Nested Question — What's the difference between port 80 and port 443 beyond the number?

**Direct Answer:**

The number itself is arbitrary — IANA could have assigned any numbers. What matters is **what each port conventionally means** and **how the server behaves**:

| | Port 80 | Port 443 |
|---|---|---|
| Default protocol | HTTP | HTTPS |
| Encryption | None | TLS |
| First thing sent | HTTP request | TLS `Client Hello` |
| Browser default | `http://example.com` | `https://example.com` |
| What ISPs can read | Everything | Only metadata (IPs, port, timing) |

They're not locked to protocols — a server can run HTTPS on port 80 or plain HTTP on port 443 — but by convention, port 80 means plain HTTP and port 443 means TLS-first HTTPS. Browsers hardcode these defaults. Many servers listen on both, with port 80 issuing a redirect to port 443.

### Nested Question — What is a TLS record?

**Direct Answer:**

A **TLS record** is the fundamental unit of data that TLS sends over the wire. Think of it as an envelope that wraps application data (or handshake messages) before it gets handed down to TCP.

**What a record contains:**

```
+--------+--------+-----------+---------------------+
| Type   | Version| Length    | Fragment (payload)  |
| (1 byte)|(2 bytes)|(2 bytes)| (encrypted/compressed) |
+--------+--------+-----------+---------------------+
```

- **Type** — what's inside: handshake data, alert, or application data
- **Version** — the TLS version (e.g., 1.2, 1.3)
- **Length** — how many bytes follow
- **Fragment** — the actual payload, encrypted and authenticated

**What happens before data becomes a record:**

1. **Fragmentation** — large application data gets broken into records (typically up to 16 KB each)
2. **Compression** — optional (rarely used today)
3. **MAC / Authentication** — a message authentication code is added so the receiver can detect tampering
4. **Encryption** — the payload is encrypted with the negotiated symmetric key
5. **Header added** — the type, version, and length header is prepended

**Where it fits in the stack:**

```
HTTP request
     ↓
TLS Record (encrypted, authenticated)
     ↓
TCP segment
     ↓
IP packet
     ↓
Frame → physical wire
```

**Key point:** TLS records ride inside TCP segments. One TCP segment might carry part of a record, a whole record, or multiple records. Handshake messages (Client Hello, Server Hello, certificates, key exchange) also travel as TLS records — they're just a different `Type` byte.

### The Connection Flow: HTTP vs. HTTPS

**Vanilla HTTP:**

```
Client ──TCP handshake──> Server ──HTTP request──> Server ──HTTP response──> Client
                          (port 80)              (plaintext)              (plaintext)
```

**HTTPS:**

```
Client ──TCP handshake──> Server ──TLS handshake──> Server ──Encrypted HTTP request──> Server
                          (port 443)              (share symmetric key)    (inside TLS tunnel)
                                                                           ──Encrypted HTTP response──> Client
                                                                              (inside TLS tunnel)
```

There are now **two handshakes** before any application data flows:

1. **TCP handshake** (3-way: SYN, SYN-ACK, ACK) — establishes the reliable pipe
2. **TLS handshake** — negotiates encryption, authenticates the server, establishes the shared symmetric key

Only after *both* handshakes complete does the first actual HTTP request (`GET /`) get sent — and it gets sent **encrypted**.

### What Changes at Each Layer

| Layer | Vanilla HTTP | HTTPS (HTTP over TLS) |
|-------|-------------|----------------------|
| L7 Application | Plain HTTP request/response | Same HTTP, but encrypted before it leaves L7 |
| L6 Presentation | Serialization/encoding | Serialization/encoding, then passed to TLS for encryption |
| L5 Session | TCP connection state | TCP state **plus** TLS session state (keys, cipher suite, certificates) |
| L4 Transport | TCP segments | TCP segments carrying TLS records (encrypted) |
| L3 Network | IP packets | IP packets (unchanged — IP headers still visible) |
| L2 Data Link | Frames | Frames (unchanged) |
| L1 Physical | Signals | Signals (unchanged) |

**Key insight:** Everything below Layer 5 is **still visible** to routers and ISPs. They can still see your IP, the destination IP, the port (443), and the TCP headers. What they **cannot** see is anything inside the TLS tunnel — the HTTP method, path, headers, cookies, or body.

---

## Unit 5 — TLS 1.2 with RSA Key Exchange

### Direct Answer

In TLS 1.2 with RSA key exchange, the client and server perform **2 roundtrips** after the TCP handshake to establish a shared symmetric key. The client generates a pre-master secret, encrypts it with the server's **public RSA key** (from the server's certificate), and sends it to the server. Only the server can decrypt it with its **private RSA key**.

### The handshake step by step

**Roundtrip 1:**

1. **Client → Server: Client Hello**
   - Says "I want to encrypt"
   - Lists supported cipher suites (e.g., RSA key exchange + AES symmetric)
   - Sends a random number (client random)

2. **Server → Client: Server Hello + Certificate + Server Hello Done**
   - Picks a cipher suite from the client's list
   - Sends its own random number (server random)
   - Sends its **certificate**, which contains the server's **public RSA key**
   - `Server Hello Done` — "that's all I have for now"

**Roundtrip 2:**

3. **Client → Server: Client Key Exchange + Change Cipher Spec + Finished**
   - Client generates a **pre-master secret** (a random number that will become the symmetric key)
   - Encrypts the pre-master secret with the server's **public RSA key**
   - Sends it over the wire — anyone sniffing sees only ciphertext
   - `Change Cipher Spec` — "switching to encrypted mode now"
   - `Finished` — handshake complete from client side

4. **Server → Server: Decrypt**
   - Server decrypts the pre-master secret with its **private RSA key**
   - Both sides combine the pre-master secret with the two random numbers to derive the **master secret**
   - The master secret becomes the symmetric key for all subsequent HTTP traffic

### The fatal flaw: no forward secrecy

Because the pre-master secret was encrypted with the server's long-term RSA public key, anyone who records this session and later obtains the server's **private key** can decrypt every recorded session and recover all symmetric keys.

From the lecture, the Heartbleed scenario: an attacker (e.g., an ISP) records all encrypted traffic for a year. Later, they obtain the server's private key (via a memory leak bug like Heartbleed in OpenSSL). Now they can go back, decrypt every recorded `Client Key Exchange` message, recover every session's symmetric key, and read every recorded conversation.

This is why RSA key exchange was eventually deprecated in favor of Diffie-Hellman.

---

## Unit 6 — TLS 1.2 with Diffie-Hellman (Forward Secrecy)

### Direct Answer

Diffie-Hellman (DH) is a key exchange algorithm that lets two parties derive a shared symmetric key **without ever transmitting the key over the wire**. Both parties contribute private values, exchange public values, and independently compute the same shared secret. This provides **forward secrecy** — even if the server's long-term private key is leaked later, past recorded sessions cannot be decrypted.

### The handshake step by step

**Roundtrip 1:**

1. **Client → Server: Client Hello**
   - Lists supported cipher suites (now including DH-based ones)
   - Sends its **Diffie-Hellman public value** (derived from a private DH exponent)
   - Sends a random number

2. **Server → Server: Server Hello + Certificate + Server DH params + Server Hello Done**
   - Picks a DH-based cipher suite
   - Sends its certificate (for authentication only — the public key in it is used to verify signatures, not to encrypt the pre-master secret)
   - Sends its own **DH public value** and DH parameters
   - Signs the handshake messages with its private key (so the client knows it's really the server)

**Roundtrip 2:**

3. **Client → Server: Client Key Exchange (optional in some DH modes) + Change Cipher Spec + Finished**
   - In ephemeral DH, the client may not need to send a separate key exchange message if it already sent its DH public value in the Client Hello
   - `Change Cipher Spec` + `Finished`

4. **Both sides derive the shared key**
   - Client computes: `server_DH_public ^ client_private mod p`
   - Server computes: `client_DH_public ^ server_private mod p`
   - Due to the math, both arrive at the **same shared secret**: `g^(xy) mod p`
   - This shared secret becomes the input to the symmetric key algorithm

### Why this provides forward secrecy

The DH private exponents (`x` and `y`) are **ephemeral** — generated for this session only and thrown away after. The shared secret `g^(xy) mod p` was never transmitted. Even if an attacker records the entire handshake and later obtains the server's long-term private key, they cannot reconstruct past session keys because the ephemeral DH exponents are gone.

### The math (briefly, from the lecture)

- Public numbers: `g` (a prime base) and `p` (a large prime modulus)
- Client picks private `x`, sends `g^x mod p`
- Server picks private `y`, sends `g^y mod p`
- Client computes `(g^y mod p)^x mod p = g^(xy) mod p`
- Server computes `(g^x mod p)^y mod p = g^(xy) mod p`
- Both arrive at `g^(xy) mod p` — the symmetric key input — without either sending `x` or `y` over the wire

An eavesdropper sees `g`, `p`, `g^x mod p`, and `g^y mod p`, but cannot efficiently derive `x` or `y` (the discrete logarithm problem), so they cannot compute `g^(xy) mod p`.

---

## Unit 7 — TLS 1.3

### Direct Answer

TLS 1.3 is a major redesign that removes the negotiation round trip, makes Diffie-Hellman mandatory, and removes RSA key exchange entirely. It completes the handshake in **1 roundtrip** (or **0-RTT** with pre-shared keys), making it significantly faster than TLS 1.2.

### What changed

**1. One roundtrip instead of two**

TLS 1.3 removes the separate "negotiation" roundtrip. The client immediately sends its Diffie-Hellman parameters in the first flight (`Client Hello`), guessing the best cipher suite. The server responds with its own DH parameters, certificate, and `Finished` — all encrypted under the newly derived keys.

```
TLS 1.3 handshake:
Client → Server: Client Hello + DH parameters + (optional early data)
Server → Client: Server Hello + DH parameters + Certificate + Finished (encrypted)
```

**2. RSA key exchange is removed**

TLS 1.3 only allows ephemeral Diffie-Hellman (or elliptic-curve variants). Forward secrecy is mandatory — there is no way to do non-forward-secret key exchange.

**3. 0-RTT (Zero Round Trip Time)**

If the client has communicated with this server before, it can use a **pre-shared key (PSK)** from that earlier session and encrypt application data in its very first message — no waiting for the server to reply.

**Nested Question — What is 0-RTT and when is it safe?**

**Direct Answer:**

**What it is:** 0-RTT lets the client send encrypted application data in its first flight, immediately alongside its TLS handshake parameters, without waiting for the server to respond. It only works if the client has talked to this server before and they agreed on a pre-shared key (PSK) from that earlier session.

**The trade-off: replay attacks.** Because the server hasn't yet confirmed it's talking to the right client in this new connection, an attacker can capture the client's 0-RTT data and **replay it** to the server. The server might process it twice.

**When it's safe:**

| Safe for 0-RTT | Not safe for 0-RTT |
|---|---|
| `GET /` — fetching a public page (idempotent) | `POST /checkout` — charging a credit card |
| `GET /api/search` — read-only lookup | `POST /api/transfer-money` — state-changing |
| Static assets (CSS, images) | `DELETE /account` |
| Anything **idempotent** — doing it twice causes no harm | Anything that **mutates state** |

**Rule:** Only send idempotent, read-only requests in 0-RTT. For anything that changes data on the server, wait for the full 1-RTT handshake.

### Side-by-side comparison

| | TLS 1.2 + RSA | TLS 1.2 + DH | TLS 1.3 |
|---|---|---|---|
| Round trips after TCP | 2 | 2 | 1 (or 0 with PSK) |
| How symmetric key is shared | Client encrypts pre-master with server's public RSA key | Both compute `g^(xy) mod p` from exchanged public values | Client sends DH params immediately; server responds |
| Key ever transmitted? | Yes (encrypted with RSA) | No | No |
| Forward secrecy | No | Yes | Yes (mandatory) |
| Server cert purpose | Contains public RSA key for key exchange | Authentication only (signs DH params) | Authentication only |

---

## Unit 8 — Certificates, PKI, and Server Authentication

### Direct Answer

The TLS handshake doesn't just exchange keys — it also **authenticates the server**. The client needs to know it's really talking to `example.com` and not an imposter. This is done through **digital certificates** and the **Public Key Infrastructure (PKI)**.

### How server authentication works

1. **The server sends its certificate** during the handshake. The certificate contains:
   - The server's domain name (e.g., `example.com`)
   - The server's **public key**
   - The identity of the **Certificate Authority (CA)** that signed it
   - A digital signature from the CA

2. **The client verifies the certificate:**
   - Is the domain name correct?
   - Is the certificate signed by a CA the client trusts?
   - Has the certificate expired or been revoked?
   - Does the signature chain lead back to a trusted **root CA**?

3. **The chain of trust:**
   - Client trusts root CAs (pre-installed in the OS/browser)
   - Root CA signs intermediate CAs
   - Intermediate CA signs the server's certificate
   - Client walks the chain from server cert → intermediate → root → trust store

### Why RSA public key was in the certificate

In TLS 1.2 with RSA key exchange, the server's certificate contained the **public RSA key** used to encrypt the pre-master secret. The client used this public key to encrypt, and only the server's matching private key could decrypt. This proved the server possessed the private key corresponding to the certificate.

### PKI and certificate lifetimes

The lecture notes that certificate lifetimes have been getting shorter to limit damage if a private key is leaked:

- Traditional certificates: valid for years
- Modern recommendations: as short as **3 months**
- Cloudflare sometimes issues certificates valid for just **2 weeks**

**Reasoning:** Even if someone hacks a server and steals its private key, they only have a narrow window (2 weeks or 3 months) to abuse it before the certificate expires.

### Heartbleed — the real-world failure

From the lecture:

> "There was something called Heartbleed — an OpenSSL bug. People were able to extract where the private key was loaded into the server memory and leak part of the memory. Part of that junk is gold — it's a private key."

Once an attacker has the server's private key:
- With **RSA key exchange**: They can decrypt all recorded sessions (no forward secrecy)
- With **Diffie-Hellman**: They still can't decrypt past sessions (forward secrecy protects them)

This is exactly why the industry moved away from RSA key exchange toward ephemeral Diffie-Hellman.

---

## Gap Pass

All main questions from the transcript have been covered:

| # | Question | Status |
|---|---|---|
| 1 | Why encrypt? | ✅ Unit 1 |
| 2 | Why a standard? | ✅ Unit 2 |
| 3 | Vanilla HTTP flow | ✅ Unit 3 |
| 4 | HTTPS / HTTP over TLS | ✅ Unit 4 |
| 5 | TLS 1.2 RSA handshake | ✅ Unit 5 |
| 6 | TLS 1.2 Diffie-Hellman / forward secrecy | ✅ Unit 6 |
| 7 | TLS 1.3 / 0-RTT | ✅ Unit 7 |
| 8 | Certificates / PKI / Heartbleed | ✅ Unit 8 |

Nested questions documented:
- What does slow start look like in practice? (Unit 3)
- What's the difference between port 80 and 443 beyond the number? (Unit 4)
- What is a TLS record? (Unit 4)
- What is 0-RTT and when is it safe? (Unit 7)

No remaining gaps from the transcript. Ready for quiz creation when you are.