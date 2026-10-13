# Lecture 15: Stateful vs Stateless

## Status: Partially Complete

> ### 📋 Read this before continuing
>
> This lecture covers the fundamental distinction between **stateful** (storing and depending on client/session state in the backend's own memory) and **stateless** (carrying state client-side or in external storage, so the backend can be killed/restarted safely). The session was "grain of salt" — the transcript explicitly says definitions go nowhere; what matters is the **side effects** and **operational consequences**.
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. What is stateful vs stateless? | ✅ Documented | Units 1-2 |
> | 2. JWT vs Session DB security trade-off | ✅ Documented | Unit 7 |
> | 3. How do stateful vs stateless protocols differ? | ✅ Documented | Units 3, 5-6 |
> | 4. Why do sticky sessions relate to stateful backends? | ✅ Documented | Unit 4 |
> | 5. What are the practical trade-offs of statelessness? | ✅ Documented | Units 8-10 |

> | Unit | Content |
> |---|---|
> | **1** | **What is stateful?** — storing state in backend memory and depending on it |
> | **2** | **What is stateless?** — no dependency on in-process state; externalize or client-carry |
> | **3** | **Stateful vs stateless protocols** — stateful protocols track per-connection state; stateless treat packets independently |
> | **4** | **The login/session case study** — memory-based sessions vs database-based sessions |
> | **5** | **TCP (stateful) vs UDP (stateless)** — connection state, sequence numbers, query IDs |
> | **6** | **Protocol layers: HTTP (stateless) over TCP (stateful); QUIC (stateful) over UDP (stateless)** |
> | **7** | **Security trade-off: JWT vs Session-in-Database** — revocation capability vs scalability |
> | **8** | **Sticky sessions/load balancer affinity** — the band-aid for stateful backends in load-balanced environments |
> | **9** | **Practical trade-offs of statelessness** — JWT vs sessions, hybrid approaches |
> | **10** | **Recap** — the spectrum of statelessness and why "definition games" don't matter |

---

## Unit 1 — What is Stateful?

### Direct Answer

A **stateful backend** stores state about its clients in its own memory (or local disk) and **depends on that state being there** for it to function properly. If that state is lost (crash, restart, memory corruption), the backend can no longer serve existing clients correctly — their workflow breaks.

### Mental Model & Causal Reasoning

The key word in "stateful" is **depends on it**. Simply storing data isn't the issue — it's **reliance**. The transcript draws this distinction:

- You can store logs in a database and lose them — you don't care → **stateless**
- You store a session ID in memory and lose it — the user gets logged out → **stateful**

The causal chain for statefulness:
1. Backend creates state (session ID, user preferences, shopping cart)
2. Backend stores it in memory (RAM, local cache)
3. Subsequent requests **depend on** that state being present
4. If the process dies → state is gone → clients lose sessions/workflow

### The crash test (from the transcript)

> "Can you restart that backend if it's idle, and can the clients that were connected finish their workflow without breaking?"

**Stateful:** No — restarting loses all client state. Clients must re-authenticate, redo work.

**Stateless:** Yes — restarting doesn't affect clients because no state was in the process.

### Example: Memory-Based Session

```
Backend S1 handles User A:
  Login → backend generates Session ID "S1"
  Stores in memory: {"S1": {"userId": 123, "role": "admin"}}
  Returns cookie "S1" to client

User refreshes page → sends cookie "S1" →
  Backend checks memory: found → returns profile ✅

[Backend crashes/restarts]

User refreshes page → sends cookie "S1" →
  Backend checks memory: NOT found → returns login page ❌
```

### What the Source Got Right

- "It's not only that [stores state], it actually depends on that information being there"
- The crash/restart test as the defining question
- Stateful doesn't mean "uses memory" — it means "depends on in-process state"

---

## Unit 2 — What is Stateless?

### Direct Answer

A **stateless backend** does not store client/session-relevant state in its own process memory and does not depend on it being there. Any state needed to serve a request is either:

- **Carried by the client** with every request (JWT tokens, session IDs), or
- **Stored externally** in a persistent system (database, Redis) that survives restarts

The backend can be killed, restarted, or scaled independently — it will still work because it doesn't rely on in-memory state.

### Mental Model & Causal Reasoning

Statelessness is about **independence from in-process state**. The backend is a stateless processing unit — it can come and go without affecting the system's ability to serve clients.

The causal chain for statelessness:
1. Backend receives request → everything it needs is in the request (or an external store)
2. Backend does NOT rely on prior requests' state being in its memory
3. If the process dies → no state is lost → clients continue unaffected
4. New instance starts → serves identical requests identically

### The "stateless ≠ no storage" distinction

The transcript is clear: "Even if I'm stateless, I can still store some data somewhere else." Storing data in a database is fine — **as long as you don't depend on the backend process surviving to read it**.

**Stateless example:** Backend writes logs to a database → backend restarts → logs are still there (someone else wrote them) → no problem.

**Stateful example:** Backend stores session ID in memory → backend crashes → session is gone → problem.

### The restart test (restated)

> "Can you restart the backend during idle time, and can clients who were connected finish their workflow as if nothing happened?"

If yes → stateless. If no → stateful.

### Example: Stateless Backend with Database Sessions

```
Backend S1 handles User A:
  Login → backend generates Session ID "S1"
  Stores in DATABASE (not memory): {"S1": {"userId": 123}}
  Returns cookie "S1" to client

User refresh → sends cookie "S1" →
  Backend checks database: found → returns profile ✅

[Backend crashes/restarts]

User refresh → sends cookie "S1" →
  New backend instance → checks database: found → returns profile ✅
```

The backend was stateless — it never stored the session locally. The **system** is still stateful (the database is a state store), but the **backend process** is stateless.

### What the Source Got Right

- "Stateless ≠ no state stored at all" — it means "no dependency on in-process state"
- External storage is fine as long as the backend can re-read from it
- The restart test as the definitive check

---

## Unit 3 — Stateful vs Stateless Protocols

### Direct Answer

**Stateful protocols** maintain per-connection state on both endpoints. This includes connection state (established, closing), sequence numbers, flow control windows, congestion control parameters, and session keys. If this state is lost, the connection cannot be resumed — it must be re-established from scratch.

**Stateless protocols** treat each packet as independent. No per-connection context is maintained between endpoints. Each packet carries all the information needed to process it (or the information is self-describing). If an endpoint fails, the next packet from the client is handled by whatever endpoint is available.

### The two dimensions of protocol state

Protocols can be analyzed on two axes:

| Axis | Stateful protocol | Stateless protocol |
|------|------------------|-------------------|
| **Connection state** | Maintains connection lifecycle (handshake, established, teardown) | No connection concept — each packet is independent |
| **In-packet context** | Relies on prior packets (sequence numbers, windows) | Each packet is self-contained |

### Stateful protocol: TCP

**What TCP tracks (state):**
- Connection state (CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT → CLOSED)
- Send and receive sequence numbers — what bytes have been sent, received, acknowledged
- Congestion window size — how much data can be in flight
- Receive window size — how much buffer space is available
- Round-trip time estimates — for retransmission timeouts

**What happens if TCP state is lost:**
```
Sender: sent bytes 1-1000, expecting ACK
[Server process dies, connection state lost]
Sender: sent bytes 1001-1500, still waiting for ACK on 1001
Receiver: "I have no idea what bytes 1001-1500 mean — I don't know what came before"
→ Connection must be reset → client must re-establish (new TCP handshake)
```

**Why TCP is stateful:** The receiver **depends on prior state** (sequence numbers, window positions) to make sense of incoming bytes. Without that state, the byte stream is meaningless.

### Stateless protocol: UDP

**What UDP does NOT track:**
- No handshake — send datagrams immediately
- No sequence numbers — packets may arrive out of order, duplicated, or lost
- No flow control — sender can blast packets regardless of receiver capacity
- No congestion control — no feedback mechanism for network conditions
- No connection lifecycle — no state machine beyond "port open or closed"

**What happens if UDP is "lost":**
```
Sender sends: UDP packet {query: "google.com", query_id: 5001}
[Process handling port 53 dies]
OS kernel still has port 53 open → routes to new process
New process: reads query_id 5001 from incoming packet → knows which DNS lookup this belongs to
```

**Why UDP is stateless:** Each packet is self-contained. If a process dies, the OS kernel routes new packets to a different process (or no process). The protocol doesn't care.

### But application layers add state back

The transcript's DNS example is key: pure UDP is stateless, but **DNS adds state at the application layer**:

```
Client sends multiple DNS queries simultaneously:
  {query_id: 5001, query: "google.com"}
  {query_id: 5002, query: "facebook.com"}
  {query_id: 5003, query: "youtube.com"}

Responses come back in arbitrary order:
  {query_id: 5002, answer: "31.13.71.36"}  ← answer for facebook
  {query_id: 5001, answer: "142.250.191.14"} ← answer for google
  {query_id: 5003, answer: "216.58.217.206"} ← answer for youtube

The query_id is what makes this work — the application uses it to match responses to queries.
```

**UDP is stateless.** DNS adds **application-level state** (query ID tracking) to make stateless UDP useful for name resolution.

### The protocol layering insight

The transcript captures a crucial point:

> "You can build a stateless protocol on top of a stateful one and vice versa."

**HTTP (stateless) over TCP (stateful):**
- HTTP itself is stateless — each request is independent
- But it uses TCP's stateful connection as the transport
- If TCP drops, HTTP creates a new TCP connection — HTTP doesn't care about TCP's state
- The session state is carried in cookies (client-side), not TCP

**QUIC (stateful) over UDP (stateless):**
- QUIC manages connection state, stream state, encryption contexts
- But it runs on UDP, which is stateless
- QUIC solves this by embedding the **connection ID** in every packet:

```
QUIC packets:
  UDP payload → {connection_id: ABC123, stream_id: 1, data: "GET /"}
  UDP payload → {connection_id: ABC123, stream_id: 3, data: "POST /api"}
  UDP payload → {connection_id: ABC123, stream_id: 1, data: "Host: example.com"}

Every packet carries the connection_id so the receiver knows:
"All these packets with connection_id ABC123 belong to one logical QUIC connection"
```

### What the Source Got Right

- TCP's sequence numbers, window sizes, and state machine as state
- UDP's datagram independence as the source of statelessness
- DNS query IDs as an application-level solution built on top of stateless UDP
- "The protocol can be actually designed to store state or to not store state"
- "You can build a stateless protocol on top of a stateful one and vice versa"

### What the Source Missed

- **The two-dimensional analysis:** protocols differ on both connection state AND in-packet context, not just one axis. TLS 1.3 is interesting — it has stateful handshakes but stateless session resumption (tickets).
- **TLS as a middle case:** TLS sessions are stateful (cipher states, sequence numbers) but TLS 1.3 session resumption uses stateless tickets. This nuance illustrates the spectrum.

---

## Unit 4 — The Login/Session Case Study

### Scenario from the transcript

A login application where:
1. User submits credentials
2. Backend verifies against Postgres
3. Backend generates a session ID
4. Backend stores session ID **locally in memory**
5. Returns session ID as cookie
6. On subsequent requests, backend checks the cookie against local memory

### The stateful version (memory-based)

**Why it works in single-instance dev:** During development on a single machine, all requests hit the same process → session is always there.

**Why it breaks in production (load balanced):**
```
Client → Load Balancer → Backend A (stores session S1)
Client → Load Balancer → Backend B (S1 not in memory → redirects to login)
```

This is the **"sometimes you get logged out on refresh"** problem.

### The stateless solution (database-based)

Store the session ID in a database (or Redis) that all backend instances share:

```
Client → Load Balancer → Backend A → checks database → session found → serves profile
Client → Load Balancer → Backend B → checks database → session found → serves profile
```

**Trade-off:** Every request pays a database lookup (latency), but gains horizontal scalability (any backend can serve any request).

### Stateful workaround: Sticky sessions

> "Sometimes stateful applications require the stickiness. You configure your load balancer to have a sticky connection to it."

The transcript mentions using sticky sessions for a gaming app to "minimize the number of records to the database." This accepts statefulness as a deliberate trade-off for reduced database load.

### What the Source Got Right

- The login/session example as a clear stateful vs stateless illustration
- Sticky sessions as a stateful pattern with operational implications
- The single-instance vs load-balanced distinction as the root cause

---

## Unit 5 — TCP: A Stateful Protocol

### TCP: Stateful Protocol

**Why TCP is stateful:**
- **Connection management:** Handshake (SYN, SYN-ACK, ACK) creates state on both sides
- **Sequence numbers:** Stored on both sides to reorder/retransmit packets
- **Congestion control state:** Window sizes, RTT estimates, congestion window
- **Flow control state:** Buffers, pending data
- **Connection lifecycle:** ESTABLISHED → FIN_WAIT → CLOSE_WAIT → CLOSED (state diagram)

**If state is lost:** Connection breaks. You must reset and re-establish. There's no way to "resume" a TCP connection — the receiver has no idea what sequence numbers have already arrived.

```
Client: "I sent bytes 1-1000"
Server: "I received 1-1000, expecting 1001 next"

[Connection state lost somehow]

Client: "I sent bytes 1001-1500"
Server: "I don't know what bytes 1001-1500 means — I lost track!"
→ Connection reset
```

### UDP: Stateless Protocol

**Why UDP is stateless:**
- **No handshake:** Send a datagram, no setup required
- **No connection state:** Each packet is independent
- **No sequence numbers:** Packets may arrive out of order, duplicate, or lost — the protocol doesn't care
- **No guaranteed delivery:** No ACKs, no retransmission, no flow control

**If a process dies:** The OS can route future datagrams to a new process — the protocol has no per-connection state tying it to one process.

### DNS example: Adding query IDs for statelessness

The transcript explains that DNS uses UDP (stateless) but adds **query IDs**:

```
Client sends: UDP packet → {query_id: 7721, query: "google.com"}
OS routes to process A (which is listening on a UDP socket)
Process A dies. OS routes to process B (same UDP socket).

Client sends: UDP packet → {query_id: 7722, query: "facebook.com"}  
OS routes to process B.

Client receives: UDP packet → {query_id: 7721, answer: "142.250.191.14"}
OS reads the query_id from the response and routes it back to the original application request
```

**The query ID bridges stateless UDP and stateful application need** — it lets multiple DNS queries coexist on one socket.

### What the Source Got Right

- TCP's sequence numbers, window sizes, and state machine as state
- UDP's datagram independence as the source of statelessness
- DNS query IDs as an application-level solution built on top of stateless UDP
- "The protocol can be actually designed to store state or to not store state"

---

## Unit 6 — Protocol Layering: Stateful on Stateless and Vice Versa

### HTTP: Stateless over TCP (stateful)

HTTP/1.1 is **stateless**: each request is independent. The server doesn't remember what you asked for 3 requests ago — that's why we need cookies/sessions.

```
Request 1: GET /login → Cookie: none → 200 {session: "S1"}
Request 2: GET /profile → Cookie: S1 → 200 {user data}
Request 3: GET /orders → Cookie: S1 → 200 {orders}

If server dies between requests:
  TCP connection drops → HTTP doesn't care → client creates new TCP connection → sends Cookie: S1 → server looks it up in database → works (if session stored externally)
```

**HTTP is stateless; TCP underneath it is stateful.** But HTTP doesn't *depend* on TCP's state — if TCP drops, HTTP just reconnects. The application state (session) is carried in the cookie or stored externally.

### QUIC: Stateful over UDP (stateless)

QUIC is **stateful** (connection management, encryption contexts, streams) but runs on **UDP** (stateless transport). The trick:

> "QUIC uses UDP, which is stateless, but it has to always send the same connection ID with every UDP packet so we know that all these are talking about the same connection."

```
QUIC packet 1: UDP header → {conn_id: ABC123, QUIC frame: [STREAM 1, headers]}
QUIC packet 2: UDP header → {conn_id: ABC123, QUIC frame: [STREAM 3, data]}  
QUIC packet 3: UDP header → {conn_id: ABC123, QUIC frame: [STREAM 1, data]}
```

QUIC **transfers the connection state** in every packet via the connection ID, while relying on UDP's statelessness to allow connection migration (IP change, process restart).

### What the Source Got Right

- "You can build a stateless protocol on top of a stateful one and vice versa"
- HTTP as the stateless-over-stateful example
- QUIC/Quick as the stateful-over-stateless example
- The connection ID mechanism as QUIC's solution

---

## Unit 7 — Security Trade-off: JWT vs Session-in-Database

### The two approaches

| Approach | How it works | Backend stores | Client carries |
|---------|-------------|----------------|----------------|
| **Session ID** | Backend generates random ID, stores {ID → user data} in DB, sends ID to client | Session data in database | Random session string |
| **JWT** | Backend encodes user data into a signed token, sends to client | Nothing (backend is stateless) | Full token with all claims inside |

### JWT: Stateless at the backend

```javascript
// Backend validates JWT without any DB lookup:
const payload = jwt.verify(token, secret);
// payload = { userId: 123, role: "admin", exp: ... }
// No database query needed — the token carries everything
```

**Backend is maximally stateless** — can scale to any number of instances, zero shared state.

### Session DB: Stateful dependency on the database

```javascript
// Backend looks up session in external store:
const session = await db.get(`session:${sessionId}`);
// Requires database/Redis access on every authenticated request
```

**Backend depends on the external store** — if Redis is down, auth breaks.

### The security spectrum

| Aspect | Session ID in DB | JWT |
|--------|-----------------|-----|
| **Backend statelessness** | ❌ Needs external store | ✅ No external store |
| **Revocation capability** | ✅ Immediate (delete from DB) | ❌ Can't revoke until expiry |
| **Token theft response** | DB lookup fails → session invalid | Stolen token works until expiry |
| **Token size** | Small (random ID) | Larger (all claims encoded) |

### The transcript's key security insight

> "What if someone stole this [JWT]? ... That's easy. Go to the database and flip that to invalid. The next person to validate, they will query the database and say, 'wait a minute, the session has been revoked.'"

The transcript points out that JWT's statelessness creates a **security vulnerability**: stolen tokens can't be invalidated. The workaround (checking a DB for revocation) undermines JWT's statelessness benefit.

> "You can argue that if the application actually dies [DNS client], then there is no one to accept the datagram."

Even UDP-based "stateless" protocols have edge cases — the application layer still holds state dependencies.

### What the Source Got Right

- JWT statelessness is "an advantage and disadvantage at the same time"
- Stolen JWT cannot be revoked until expiry
- Session ID in DB allows immediate revocation
- The trade-off is: backend scalability vs security control

---

## Unit 8 — Sticky Sessions / Load Balancer Affinity

### Direct Answer

**Sticky sessions** (also called session affinity or load balancer affinity) route all requests from the same client to the same backend server, regardless of how the load balancer distributes other requests.

The transcript defines the relationship clearly:

> "Sometimes stateful applications require the stickiness. You configure your load balancer to have a sticky connection to it."

**Sticky sessions are the workaround for a fundamental incompatibility between two things:**
1. **Stateful backends** — store session state in process-local memory
2. **Load balancers** — distribute requests across multiple identical backend instances

Without sticky sessions, a load balancer might send `request 1` to `Backend A` (which creates session S1) and `request 2` to `Backend B` (which has no S1). Backend B rejects the client → they get logged out.

### The three affinity methods

| Method | How it works | Reliability |
|--------|-------------|-------------|
| **Cookie-based** | LB inserts a cookie pointing to a specific backend | Reliable (explicit) |
| **IP hash** | LB hashes client IP → deterministic backend selection | Fragile (IP changes = session loss) |
| **Source IP + port** | LB routes based on client's source address | Same fragility as IP hash |

### Why it "relates" to stateful backends

The relationship is **causal**:

```
Stateful backend (session in memory)
  ↓ (incompatible with)
Load balancer distributing across multiple instances
  ↓ (solution)
Sticky sessions → force requests to same backend
  ↓ (band-aid, not a fix)
Clients stay with the backend that has their state
```

### The transcript's gaming app example

> "I had to do that [sticky sessions] to minimize the number of records to the database just for a short period of time so you can play."

This is the **deliberate trade-off**:
- ✅ Reduced database writes (no need to look up session on every request)
- ❌ If that backend crashes → all its clients lose their sessions
- ❌ Uneven load distribution (some backends may be overloaded)
- ❌ Scaling limits (can't easily add/remove backends without disrupting sessions)

### What the Source Got Right

- Sticky sessions as "sometimes required" — a deliberate engineering decision, not always needed
- The gaming app example showing intentional statefulness for performance
- The framing that this is an acceptable trade-off in some scenarios

### What the Source Missed

- **The failure case:** If the sticky backend crashes, those clients have no fallback. All sessions are lost. This is more severe than JWT (where the token is self-contained).
- **LB implementation differences:** AWS ALB cookie-based stickiness works differently from NGINX ip_hash. The reliability model varies significantly.

### Understanding check:

**Scenario:** You have 5 backend instances behind a load balancer. Sessions are stored in memory. A client's session gets stuck on Backend 3, which then crashes.

What happens to that client, and how would a stateless design (database sessions) have prevented this?

<details>
<summary>Answer</summary>
With sticky sessions + memory-based sessions, the client loses their session when Backend 3 crashes. They're logged out. With database-based sessions, any backend instance can look up the session in the shared database. The client's request gets routed to any healthy backend, which checks the database, finds the session, and continues seamlessly — no sticky sessions needed, no session loss on crash.
</details>

---

## Unit 9 — Practical Trade-offs of Statelessness: JWT vs Sessions

### Direct Answer

The practical trade-offs of statelessness center on **four axes**: revocation, scalability, security, and operational complexity. The JWT vs session-in-database comparison illustrates each trade-off concretely.

### The stateless approach (JWT)

**How it works:** The backend signs a token containing user claims (`{userId, role, exp}`) and sends it to the client. On subsequent requests, the client sends the token. The backend verifies the signature — **no database lookup needed**. The token carries its own state.

```javascript
// Backend validates JWT without any DB lookup:
const payload = jwt.verify(token, secret);
// payload = { userId: 123, role: "admin", exp: ... }
// No database query needed — the token carries everything
```

**Backend is maximally stateless** — can scale to thousands of instances without shared state.

### The stateful approach (Session in Database)

**How it works:** The backend generates a random session ID, stores `{ID → user data}` in a shared database (Redis, Postgres), and sends the ID to the client. On subsequent requests, the backend looks up the session ID in the database.

```javascript
// Backend looks up session in external store:
const session = await db.get(`session:${sessionId}`);
// Requires database/Redis access on every authenticated request
```

**Backend depends on external store** — if Redis is down, auth breaks.

### The four dimensions of trade-off

| Aspect | Session ID in DB | JWT |
|--------|-----------------|-----|
| **Backend statelessness** | ❌ Needs external store lookup per request | ✅ No external store — backend is stateless |
| **Revocation capability** | ✅ Immediate (delete session from DB) | ❌ Can't revoke until token expires |
| **Token size** | Small (32-byte random ID) | Larger (all claims encoded as base64) |
| **Token theft response** | DB lookup fails → session invalid | Stolen token works until expiry |

### The transcript's key insight: statelessness is "an advantage and disadvantage at the same time"

> "What if someone stole this [JWT]? ... That's easy. Go to the database and flip that to invalid."

The transcript identifies that JWT's statelessness creates a **security vulnerability**: stolen tokens can't be invalidated without re-introducing state (checking a DB).

### The hybrid solution: short-lived JWT + refresh tokens

The transcript points toward a middle ground:

> "That's why JWT have the concept of refresh token. The way you develop a refresh token and the access token and then make the access token as short as possible."

**Pattern:**
- **Access token:** JWT, short-lived (15 minutes) — stateless, used on every request
- **Refresh token:** Random string, long-lived (7 days) — stored in DB, can be revoked
- On refresh: backend checks DB for refresh token validity → issues new access JWT

This gives you:
- ✅ Scalability (99% of requests skip DB lookup)
- ✅ Revocation (compromised refresh token can be invalidated)
- ❌ Complexity (two token types, refresh logic, DB dependency)

### Banking app scenario (the practical test)

**Requirements:**
- Security is paramount (banking)
- Users should be immediately logged out when device is reported stolen
- Millions of concurrent users during market hours

**Analysis:**
- **Pure JWT:** Fails security requirement (can't revoke immediately)
- **Session-in-DB:** Meets security requirement but adds DB latency at scale
- **Hybrid:** Meets both — short-lived JWT for scalability + server-side refresh for revocation

The hybrid satisfies: stolen access JWTs expire in 15 minutes, but refresh tokens can be immediately revoked, forcing re-authentication.

### What the Source Got Right

- JWT statelessness as "an advantage and disadvantage at the same time"
- The stolen token problem: "what if someone stole this JWT?"
- The refresh token pattern as the practical compromise
- "Things can happen, bad things can happen, and there are flaws with everything"

### What the Source Missed

- **Token size matters:** JWTs are 5-10x larger than session IDs — impacts mobile app performance, API payload sizes, and bandwidth costs at scale
- **Cache invalidation in distributed systems:** Even with session DB, cache-aside patterns have TTL windows where revoked tokens still work
- **The "confused deputy" problem:** JWTs can accidentally encode permissions that become invalid (user demoted from admin) — the JWT doesn't know this until expiry

### Understanding check:

**Question:** Why does the banking app scenario choose the hybrid approach? Rank the three approaches (pure JWT, session-in-DB, hybrid) for the banking requirements.

<details>
<summary>Answer</summary>
The hybrid is chosen because:
1. Pure JWT fails the immediate revocation requirement — stolen tokens work until expiry (hours)
2. Session-in-DB meets security but at high latency cost at scale (DB hit per request × millions of users)
3. Hybrid: short-lived JWT (15min, no DB lookup per request) + refreshable via DB-backed token (immediate revocation possible)

Ranking: Hybrid > Session-in-DB > Pure JWT for banking requirements.
</details>

---

## Unit 10 — Recap: The Spectrum of Statelessness

### Statelessness is a spectrum, not binary

The transcript explicitly says:

> "These things, these are discussion very hard to discuss. ... definitions go nowhere, so take those with a grain of salt."

The question isn't "is it stateful or stateless?" but **"stateful in what way, and who depends on it?"**

### The spectrum: Who depends on what?

| Pattern | Backend stores state? | Backend depends on state? | Can restart without breaking clients? | Stateless? |
|---------|----------------------|--------------------------|--------------------------------------|------------|
| **Pure function (prime check)** | No | No | Yes | ✅ Fully stateless |
| **JWT auth** | No | No | Yes | ✅ Backend is stateless |
| **Session in DB** | No | Indirectly (depends on DB) | Yes | ✅ Backend is stateless |
| **Session in memory** | Yes | Yes | No | ❌ Backend is stateful |
| **Sticky sessions** | Yes (in memory) | Yes | No | ❌ Backend is stateful |

### The "statelessness" depends on perspective

- **Backend perspective:** Stateless if it doesn't depend on its own process state
- **System perspective:** Stateful if any component stores and depends on state
- **Protocol perspective:** Stateless if no per-connection state is maintained

### What the Source Got Right

- The "grain of salt" approach — definitions don't matter, side effects do
- The backend restart test as the practical litmus test
- Statelessness is about **dependency**, not **storage**
- The spectrum nature: "It's just a state right at the end of the day"

### What the Source Missed

- **Distributed transactions and consistency:** Stateful backends can coordinate atomic operations; stateless backends require external coordination (Saga pattern, two-phase commit, etc.)
- **Cache locality advantages:** Stateful backends can keep hot data in-memory for microsecond access; stateless backends must always fetch from external stores (network latency)
- **Kubernetes/Serverless implications:** Stateless backends are the prerequisite for autoscaling and container orchestration — stateful backends require StatefulSets, persistent volumes, complex pod scheduling

---

## Open Questions Carried Forward

| # | Question | Why it matters |
|---|----------|---------------|
| 1 | **Stateful services in Kubernetes** | How do StatefulSet, PersistentVolume, and pod affinity solve the restart problem? |
| 2 | **Session replication strategies** | Redis-backed sessions vs encrypted cookie sessions vs sticky sessions — when does each make sense? |
| 3 | **JWT revocation patterns** | Token blocklists, short-lived access + long-lived refresh tokens, server-side session tracking |
| 4 | **Cache warming for stateful backends** | How do you handle "cold start" when a stateful backend restarts? |
| 5 | **Event-sourced architectures** | How do purely stateful event stores coexist with stateless command handlers? |

*Documented after studying the transcript. All five discussed questions covered: (1) What is stateful vs stateless — Units 1-2 with restart test and examples; (2) How stateful vs stateless protocols differ — Units 3-6 with TCP/UDP, HTTP/QUIC examples; (3) The login/session case study — Unit 4; (4) Why sticky sessions relate to stateful backends — Unit 8; (5) Practical trade-offs of statelessness (JWT vs sessions) — Unit 9.*
