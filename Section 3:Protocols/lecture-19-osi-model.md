# Lecture 19: OSI Model

## Status: Complete

> ### 📋 Read this before continuing
>
> This lecture covers the **OSI (Open Systems Interconnection) model** — the seven-layer framework that describes how networked communication works. The lecturer's own story frames it: he memorized it 20 years ago in university, didn't see the value, and now considers it critical — because its real purpose is answering one question for engineers: **where does your application live?** (Are you bridging two other apps? Are you looking at MACs, IPs, ports, or byte streams?)
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. Why do we need a standard communication model? | ✅ Documented | Unit 1 |
> | 2. What is the OSI model? | ✅ Documented | Unit 2 |
> | 3. Layer 7 — Application | ✅ Documented | Unit 3 |
> | 4. Layer 6 — Presentation | ✅ Documented | Unit 4 |
> | 5. Layer 5 — Session | ✅ Documented | Unit 5 |
> | 6. Layer 4 — Transport | ✅ Documented | Unit 6 |
> | 7. Layer 3 — Network | ✅ Documented | Unit 7 |
> | 8. Layer 2 — Data Link | ✅ Documented | Unit 8 |
> | 9. Layer 1 — Physical | ✅ Documented | Unit 9 |
> | 10. Gap pass — middle boxes, TCP/IP comparison | ✅ Documented | Units 10–11 |
>
> | Unit | Content |
> |---|---|
> | **1** | **Why do we need a standard communication model?** — three reasons: medium-agnostic apps, interoperable/upgradable network equipment, decoupled per-layer innovation |
> | **2** | **What is the OSI model?** — the definition, each layer introduces a concept, the shared engineering language, criticism and the "grain of salt" |
> | **3** | **Layer 7 — Application** — your code and protocols (HTTP, FTP, gRPC); the dual perspective; libraries absorb L7 |
> | **4** | **Layer 6 — Presentation** — serialization and encoding; cross-language payoff; the criticized layer |
> | **5** | **Layer 5 — Session** — connection establishment, TLS, state on both ends; the Linkerd example; UDP's lack |
> | **6** | **Layer 4 — Transport** — ports, TCP/UDP/QUIC, segments vs datagrams; "we live in layer four and layer seven" |
> | **7** | **Layer 3 — Network** — routing, IP addresses, packets; routers are L3 devices; VPNs as IP-in-IP |
> | **8** | **Layer 2 — Data Link** — MAC addresses, frames, switches/ARP/Ethernet/WiFi; the hub contrast |
> | **9** | **Layer 1 — Physical** — the bare metal; signal conversion; medium independence |
> | **10** | **The middle boxes** — peek depth per device; transparent firewalls and the public-below-L5 boundary; the two-session reverse proxy |
> | **11** | **OSI vs TCP/IP** — the layer mapping, the numbering collision, the lecturer's preference for granularity |
>
> **Naming conventions to use throughout** (the lecturer repeats these): **frame** = L2, **packet** = L3, **segment** = L4 (TCP), **datagram** = L4 (UDP).

---

## Unit 1 — Why Do We Need a Standard Communication Model?

### Direct Answer

A standard communication model exists for three reasons, all from the lecture's "why" opening:

1. **To build agnostic applications** — the same app runs over any medium (WiFi, Ethernet, LTE, fiber, satellite) without knowing the physical layer's language.
2. **To make network equipment interoperable and upgradable** — routers, switches, and every vendor's box can talk to each other and be replaced/upgraded because the medium is fully decoupled from the equipment.
3. **To decouple innovation** — each layer can be improved or reinvented independently without breaking the layers above or below.

### Reason 1 — Agnostic applications

Without a standard, the lecturer's disaster scenario: your server has no idea how to talk to your client — "how do you transpose the bits into digital to analog and analog back to digital?" Worse, you'd need **a different version of your application per medium**:

> "Imagine if you have to order different version of your apps so that it works on wi fi. You need a different version to work on the Ethernet. You need a different logic to work on LTE and you need a different version to work on fibre. That will be a disaster."

The lecturer stresses we take this for granted: a Node.js app compiled for any CPU sends requests "through the satellite or... radio wi fi or... Ethernet electric signals or... LTE radio waves or... fiber light" — it doesn't matter, "because we built a standard and that standard is used globally."

### Reason 2 — Network equipment management (interoperability + upgrades)

**What "network equipment" means** (the clarifying point from our discussion): the physical devices that sit between sender and receiver — **routers, switches, hubs, firewalls, load balancers, access points, network cards, proxies, CDNs**. These are the devices in the middle of the path.

Without a standard, "this router won't be able to talk to this router because there is no standard." Every vendor's box could speak a private dialect, so nothing interoperates and nothing can be upgraded safely.

The beauty of the standard: "regardless of the underlying medium, you can upgrade the actual network because it is completely decoupled from the actual medium itself." Whatever you bring in "will just support it... as long as we know how to talk to it."

### Reason 3 — Decoupled innovation per layer

**What "innovation in each layer" means** (the second clarifying point): the layer boundaries are standardized interfaces, so a new invention at one layer lands without redesigning the others. The contract is simple — honor the interface to the layer above/below, and everything else keeps working.

The lecturer's examples:

| Layer | Innovation scenario | Why the rest of the stack is untouched |
|---|---|---|
| **L1 Physical** | A more efficient medium than fiber is invented | "Layer two has nothing to do with it... layer two will pass in the bits and the conversion will happen in the physical layer" |
| **L3 Network (IP)** | New content/headers at the IP level | Can be added at that layer alone (caveat below) |
| **L4 Transport** | A brand-new transport protocol (e.g., QUIC) | Applications above adopt it without rewriting their logic |

This is "decouple the innovation" — the layered model is what makes isolated improvement possible.

**The caveat the lecturer flags — protocol ossification:** extending a layer mid-deployment isn't free, "because there is something called the protocol ossification, which is all these protocols, all these routers in the middle actually understand how to read things in a way. And if this thing changed, they freak out." He hints this deserves its own lecture.

### What the Source Got Right

- Starting with "why" before "what" — the three reasons make every subsequent layer decision motivated
- The disaster scenario (app-per-medium) as the most persuasive statement of what standardization buys
- Network equipment interoperability as a *practical* consequence, not just theory
- Innovation decoupling as the architectural payoff — each layer is built so it can "improve each layer alone"
- The ossification caveat as intellectual honesty: the layers are standard, but the deployed world constrains how freely they can evolve

### What the Source Left Open

- The specific list of layers and their boundaries (covered in later questions)
- The TCP/IP model comparison — the lecturer previews that layers 5–7 collapse into one "application" layer there, and calls the OSI model's finer granularity preferable
- Protocol ossification details — flagged as a possible future lecture

---

## Unit 2 — What Is the OSI Model?

### Direct Answer

The **OSI (Open Systems Interconnection) model** is a standard communication model consisting of **7 layers**, where **each layer is responsible for a certain purpose**. "Open" because communication must be open — a shared standard anyone can implement.

But the definition only becomes useful with three additions:

1. Each layer **introduces a concept** — not just a "purpose."
2. The model is a **shared engineering language** — its central question is "where does your application live?"
3. The granularity is **preferred but debatable** — the lecturer prefers 7 layers over the TCP/IP model's 4, while admitting the boundaries blur.

### Addition 1 — Each layer introduces a concept

The lecturer's phrasing: *"every layer has kind of a beautiful concept introduced with it."* The layers aren't arbitrary slices — each one **adds a new idea that didn't exist below it**:

| Layer | Concept it introduces |
|---|---|
| L2 Data Link | Frames, MAC addresses |
| L3 Network | Routing, IP addresses, packets |
| L4 Transport | Ports, segments/datagrams, flow & congestion control |
| L5 Session | Connection state (where stateful vs stateless lives) |

So the model answers "what's new at this layer?" — not just "what happens here?"

### Addition 2 — The model is a shared language, and its question is "where does your app live?"

Two points the lecturer hammers:

1. **It's a shared language for engineers** — *"there is no layer five"* in an actual computer. *"It's not something you read... it's something we talk about. So as engineers can talk through things."* The model exists so engineers can say "this is a layer 4 proxy" or "that CDN is layer 7" and mean something precise.

2. **The core question for a backend engineer: where does your application live?** If your app sits between two other apps (a reverse proxy, an API gateway), you need to know what you're even looking at — MAC addresses? IP packets? TCP segments and ports? Decrypted JSON? *"Every single layer has a meaning and every application out there — a CDN reverse proxy, a load balancer — has to live in one or more of these layers."*

### Addition 3 — The criticism and the "grain of salt"

- There **is** criticism of OSI: too many layers, hard to comprehend, and the boundaries are genuinely argueable — *"it's all people speculating at the end of the day... it becomes an opinion."*
- **Take it with a grain of salt** — boundaries blur; some layers do each other's jobs; nothing is "clear cut and dry."
- Despite that, he **prefers OSI's granularity** over the TCP/IP model, which collapses layers 5–7 into a single "application" layer.
- His practical goal for the course: don't memorize everything — *"just understand the seven layers effectively... and where does your application live?"*

### What the Source Got Right

- Framing the model by its *purpose* (a language for locating applications) rather than as a taxonomy to memorize
- The "beautiful concept per layer" framing — it turns seven names into seven ideas
- Admitting the model's boundaries are debatable while still preferring the granularity
- The naming discipline (frame/packet/segment/datagram) as the practical payoff of knowing layers

### What the Source Left Open

- The seven layers themselves — each gets its own unit (Units 3–9, walking 7 → 1 as planned)
- The TCP/IP model comparison — previewed; layers 5–7 collapse into one "application" layer there
- Protocol ossification — flagged as a possibly standalone topic

---

## Unit 3 — Layer 7: Application

### Direct Answer

Layer 7 is where **your application code and its protocols live** — the code running in a high-level language (JavaScript, Python, etc.) on the client device or the server, and the protocol governing the exchange between applications. HTTP is the flagship; FTP and gRPC also live here. The output of this layer descends to the presentation layer (L6), where the data gets serialized into bytes.

### The dual perspective on L7

The lecture stresses that Layer 7 is looked at **differently depending on who's looking**:

- **To a network engineer:** "layer seven to a network engineer, that is just data coming in" — opaque payload, contents irrelevant.
- **To a backend engineer:** it's where the actual work lives — "they are using different libraries, they're listening, they're sending packets... maybe a gRPC protocol or certain protocol that sits on top of HTTP/2." As the lecturer puts it, "the application is really huge here and everyone is looking just at that layer differently."

### What actually lives at L7

**Protocols:** HTTP is the main one ("the HTTP is the application, or FTP, or anything here at this layer"), plus gRPC — which itself sits on top of HTTP/2 and "creates more and more packets."

**On the receive side — where your framework takes over:** this is the layer that invokes your code. The lecture's example:

> The request arrives, "your Express [or] Apache request... will trigger that event which says, hey, someone just sent you a post request, what do you want to do with it?" — the handler function runs, hits the database, does work, and returns the response synchronously.

### The counterintuitive note: engineers rarely touch L7 directly

The lecturer says backend engineers "don't really interact with layer seven directly" — they work through libraries (Axios, fetch, Python's requests) that handle the protocol mechanics. What the engineer controls is the *logic* the layer invokes: the handler that consumes the request and produces the response.

### The handoff to L6

A nuance worth pinning: the application doesn't transmit its objects directly. It hands the data (e.g., a JSON object) **down** to Layer 6, where the object is serialized into a flat byte string. The protocol and port decisions (HTTP, port 443) are settled as the message descends through L5/L4 — not by the application code itself.

### What the Source Got Right

- L7 as the home of both the application code and its protocol (HTTP/FTP/gRPC)
- The dual perspective — "just data" vs. "libraries and protocols" — as the reason L7 is so contested
- The receiving-side flow (Express/Apache triggering the handler, DB call, synchronous response)
- "Backend engineers don't interact with layer seven directly" — libraries absorb it

### What the Source Left Open

- The mechanics of what happens *inside* the presentation layer when the object becomes bytes (next unit)
- Which specific protocols beyond HTTP/FTP/gRPC live at L7 (SMTP, DNS — both appear in the section's protocol lineup)

---

## Unit 4 — Layer 6: Presentation

### Direct Answer

The Presentation layer converts application data structures into a **transmittable format** — and back again on the receiving side. Its two jobs are **serialization** (object → bytes) and **encoding** (character set, e.g. UTF-8). "Your application sends a whole object, but the conversion of the encoding happens there."

### The mechanics

The lecture's walkthrough: a POST with JSON via Axios/fetch. The presentation layer "takes your JSON object... and serializes it to flat byte strings... and shove this bytes as a bunch of data."

Why it exists: "you can send an object doesn't mean anything... the object is meaningful in your language only." The bytes are the universal form.

### It's "a step down from your application"

The lecturer's framing: "while you shouldn't care about this, this is already happening for you." Your code thinks in objects; L6 thinks in bytes. This layer is largely invisible to backend engineers — libraries handle it.

### The cross-language payoff

On the receive side, the layer reverses the process — "that fly-by string JSON back to the JSON object for the app to consume." The payoff of a standard format: serialization on each side can be **completely different languages** — "you can serialize from JavaScript, but this serialized into Python or you can serialize from C sharp and D serialize into go." The format is the shared contract; each side converts to and from its own native types.

### The criticism (the honest part)

This is the layer people mock: "Really? You really need to break this fine of a level of detail. I don't care... sometimes this is one of the criticisms that presentation layer and also the session layer." That's exactly why the **TCP/IP model collapses L5/6/7 into one "application" layer**. The lecturer admits L6 is "really the layer that I almost never hear anyone talk about" — but the granularity still maps something real.

### The layer that can do nothing

In the connection-establishment flow, when a segment reaches session and the question comes back: "Do I need to do anything in presentation layer? Nope." The layer only acts when there's actual data to encode/decode — during a SYN, there is none.

### What the Source Got Right

- Serialization/encoding as the layer's concrete job, with the JSON example
- The "invisible to you" framing — it's already happening for you
- The cross-language deserialization payoff (JS → Python, C# → Go) as the reason a standard format matters
- Honest acknowledgment that L6 is the most-criticized, least-talked-about layer

### What the Source Left Open

- Whether encryption belongs here — this lecturer puts TLS at the **session layer (L5)**, not L6, which differs from the classic OSI textbook assignment
- The specific encoding formats beyond JSON (protobuf as binary serialization — covered in the properties lecture)

---

## Unit 5 — Layer 5: Session

### Direct Answer

The Session layer is responsible for the **state of the communication** — connection establishment, session state on both endpoints, and TLS. It's the layer that decides whether a connection even exists before any application data flows.

### This is where TLS happens

"the session layer is where TLS happened, where connection establishment happened, where state effectively takes place." (Note for later: in this lecturer's model the TLS handshake is L5, not L6.)

### State lives on both sides

"you store a state in the client... you state, store a state in the server." This is the origin of the **stateful vs stateless** distinction from Section 2: **TCP is a stateful protocol; HTTP is stateless and "doesn't have a session layer."** If a session is destroyed, "either you can restart the connection or invalidate it."

Corollary from the middle-boxes discussion: **UDP has no session layer** — "there is no state in UDP. UDP is stateless. It just sends data and that's it."

### The defense of this layer (the Linkerd example)

People mock it — "really session layer? Really? You need a whole layer just to talk about session" — but the lecturer came to defend it with a concrete proof: **Linkerd** (and Envoy) "built logic only at the session layer, only at the connection establishment. They would do certain logic... they capture a connection and save it or do pooling about it. So their application is a layer five app."

This is the "where does your application live" payoff in action — a service mesh proxy is a *layer 5 app* because it operates purely on connection state.

### The gatekeeping behavior

From the "do I always go through all layers?" walkthrough, L5 is the layer that **pauses the descent**: the app sends its POST, then "session says, wait a minute, I don't have a session here... let me create one" (SYN/ACK exchange). Only after the connection exists does the session say "now the post JSON request is resumed" and let the data descend to L4.

On the receive side, a segment with no valid session can be rejected at L5 alone — "you wouldn't be sending me data without a connection... that's why... that's the TCP ID [RST] effectively" — the data never reaches the application.

### What the Source Got Right

- Connection establishment and TLS as L5's concrete jobs
- State on both client and server, and the stateful/stateless split as direct consequences
- The Linkerd/Envoy example as proof the layer maps to real products ("layer five app")
- UDP lacking a session layer as the clean contrast
- The gatekeeping flow — connection first, then data — as the layer's operational role

### What the Source Left Open

- The exact mechanics of the TCP handshake itself (SYN, SYN-ACK, ACK) — deferred to the TCP lecture (22)
- Whether UDP's "no session layer" is a model artifact or a real architectural absence — the lecturer hedges ("might be wrong there")

---

## Unit 6 — Layer 4: Transport

### Direct Answer

The Transport layer enables **process-to-process (application-to-application) communication** by adding **port numbers** to the data, so the receiving host knows which process to deliver to. It is also where the reliability machinery — flow control, congestion control, ordering, retransmission — lives when the transport protocol is TCP.

### The lecturer's declaration: backend engineers live here

"As a back engineer, that is the only two things that are you going to worry about most of the time... we live here, baby. We live in layer four and layer seven." Keep-alive, RPC sessions, connection behavior — all L4 territory.

### The protocols that live here

"TCP UDP. Very, very important concept. The protocols live here. These are the only two protocols, let's be honest" — plus QUIC as the newer third ("roughly new, but it's a transport protocol"). The dependency chain: "HTTP built on top of TCP, HTTP/2 [on] TCP, HTTP/3 [on] QUIC, QUIC is built on UDP" — **everything eventually runs on TCP or UDP**.

Naming discipline: **segments** in TCP, **datagrams** in UDP.

### What L4 introduces that L3 didn't have

"Here we have visibility to ports. Port 80, 443. Here [Layer 3] we don't know what ports are. We only know addresses, IP addresses."

And the practical mapping: "each port is technically one process... this can change, obviously, but this is just a general rule of thumb."

### The machinery that lives here

From the receiver walkthrough: "the IP packets from layer three are assembled into TCP segments... and then deals with congestion controls or flow controls and [re]transmission and hey, is this the SYN, is this the [right] packet? Is this the old packet? Is this a new one... going to order all that stuff."

So L4 delivers: ports, ordering, flow/congestion control, and retransmission (full mechanics in the TCP lecture, 22). The contrast — skipping transport entirely: "you can be fancy and build application directly on the IP, which is layer three... I don't care if the packet reaches or not. I'll try my best."

### What the Source Got Right

- Ports as the concept L4 introduces — and the port-to-process mapping
- "The only two protocols, let's be honest" — TCP/UDP as the foundation everything else sits on
- "We live in layer four and layer seven" — the backend engineer's home
- The naming discipline (segments vs datagrams) as a communication tool

### What the Source Left Open

- TCP internals: handshake, flow control algorithms, congestion control (lecture 22)
- QUIC's design and why it exists (lecture 28)
- How exactly multiple IP packets are reassembled into one application-level message (touched in the lecture's POST example — "this could be a single packet or a segment... or multiples")

---

## Unit 7 — Layer 3: Network

### Direct Answer

The Network layer is responsible for **routing the packet to its destination host**. This is the layer that introduces **IP addresses** — and where routers do their work.

### What this layer introduces

"The IP have the concept of the **routing and the IP addresses**." Naming discipline: **packet** at L3.

### What's absent here

Two important absences:

- **No ports** — "we only know addresses, IP addresses"
- **No transport concept** — "you can be fancy and build application directly on the IP, which is layer three, the IP protocol, the internet protocol. Here, we don't have a transport concept... I don't care if the packet reaches or not. I'll try my best... it's a packet that's destined to an IP"

### Routers are layer-3 devices

"The router will need... the IP addresses in order to route, so it needs to go up to layer three. That's why a **router is a layer three device, while the switch is a layer two device**. Very critical concept."

And every router on the path asks the same question: **"is this packet for me?"** — if not, it forwards along toward the destination. (This is why a proxy that inspects traffic "goes up to layer three and go down" as packets traverse.)

### Where source/destination IPs get attached

From the request walkthrough: the TCP segment descends into an IP packet, and "we shove that send request... down to an IP packet and add the source and destination IP. So if you need the IP address, you need to do a **DNS** to get the IP address." DNS feeds this layer; then the packet descends to L2, where ARP resolves the MAC.

### VPNs live here

"VPN is actually a layer three protocol... it takes the IP packets and put them in another IP packet." Tunneling is IP-in-IP — the tunnel doesn't care whether the inner payload is HTTP, gRPC, or anything else; it's all just IP. (The lecturer notes some VPNs also operate at L2.)

### What the Source Got Right

- Routing + IP addresses as the concepts this layer introduces
- The router-vs-switch layer distinction as "a very critical concept"
- The "is this packet for me?" question every hop asks
- DNS as the prerequisite that feeds L3, and ARP as what feeds L2 below
- IP-in-IP tunneling as the elegant explanation of what a VPN fundamentally is

### What the Source Left Open

- IP packet structure, routing tables, TTL, fragmentation mechanics (the Internet Protocol lecture, 20, is 1h7m — the full treatment is there)
- IPv4 vs IPv6 differences
- How routers make forwarding decisions beyond "is this for me?"

---

## Unit 8 — Layer 2: Data Link

### Direct Answer

The Data Link layer handles delivery using **physical network addresses — MAC addresses**. This is the layer that introduces **frames**, and it's where switches, Ethernet, and WiFi operate.

### The knowledge boundary

"we don't know about IP addresses at this layer. Nothing. We only know about MAC addresses." Each layer understands only its own concept: L3 talks IP, L2 talks MAC.

### The unit of data: frames

Naming discipline: **"we send frames in layer two"** (vs packets at L3, segments at L4). The frame wraps the IP packet — the IP packet "gets shoved into a frame" and the frame adds the **source and destination MAC addresses**.

### How the MAC gets filled in — ARP

From the request walkthrough: "each packet goes into a single frame and adds the source and destination Mac addresses. So we need to figure out the Mac as well. **That's ARP.**" ARP resolves the IP into the MAC so the frame can be built.

### Who lives here: switches, Ethernet, WiFi

- **Switches are layer-2 devices** — the opposite depth from routers: "the switch... only need to take the physical layer, convert that to a data link, look at the frames, look at the destination Mac address and then that's it. **It doesn't need the IP address.**" It knows "this computer is connected to this port, so the MAC address of that is this, so it goes to this port" — no broadcast to everyone.
- **The hub as the bad old days:** "unlike a hub... where just literally the hub broadcasts the data to every single port, which is a lot of bandwidth wastage."
- **The protocols here:** Ethernet, WiFi 802.11, and similar.

### The nesting (matryoshka)

Your data at this stage: application bytes → JSON string → TCP segment → IP packet → **frame** → then the whole frame becomes "this red blob of radio signals" at the physical layer. Each layer wraps the one above, adding its own addresses at the edges.

### What the Source Got Right

- MAC addresses as the concept this layer introduces, with the hard boundary ("we don't know about IP addresses at this layer")
- The switch's MAC-only lookup as proof of what "a layer-2 device" means
- ARP as the bridge from L3's world to L2's world
- The hub-vs-switch contrast as the motivation for MAC-aware switching

### What the Source Left Open

- Frame structure (headers, FCS/checksum, frame types) — not detailed
- Ethernet vs WiFi frame differences
- What happens when the destination MAC is unknown (broadcast/ARP request mechanics)

---

## Unit 9 — Layer 1: Physical

### Direct Answer

The Physical layer is **the bare metal** — it converts frames into the signals the medium requires and back again. It's where the medium-agnostic promise of the standard model is physically realized.

### The signal types

"Once you take the frame, how does it transport to the other medium? **Electric signal** when it comes to Ethernet... **fiber and light**, light waves. Is it **radio waves**, you know WiFi or LTE. All this stuff, the physical. And when it comes to this, it's just radio waves at the end of the day or light or electricity."

### The conversion chain

"Each frame becomes a string of bits in the physical layer. Just **10101010**. These ones and zeros are converted into this radio signal or electric signal or light, and then that's it."

On the receive side, the network card picks up the signal and the chain reverses: signal → digital bits → frame (the bits "from layer one are a symbol and into frames") → IP packet → segments → up the stack to the application.

This layer also detects **where the frame starts and ends** — "that's what the physical layer does" — it's the receiver's entry point back into L2.

### The beautiful payoff: medium independence

Endpoints don't need to share a medium: "your server might be on fiber, but your network you can have sent from WiFi. I can send your data from WiFi, right? And then you receive it through fiber."

This is the OSI promise from Unit 1 realized at the bottom layer — the same frame can traverse electric, light, and radio segments without the application knowing.

### The decoupling made concrete

L1 is also where the decoupled-innovation idea from Unit 1 lands: a new/faster medium is an L1 invention — "layer two has nothing to do with it" — the conversion happens here, and everything above keeps working unchanged.

### What the Source Got Right

- "The bare metal" as the honest description — radio/light/electricity
- The bits-101010 conversion as the layer's concrete job
- Frame boundary detection as the physical layer's responsibility
- The WiFi-sender → fiber-receiver example as the payoff of standardization
- The layer's role in enabling medium-independent innovation

### What the Source Left Open

- Modulation, encoding schemes, bit rates, cabling/wavelength specifics — deliberately out of scope ("bare metal")
- How frame start/end detection actually works (preamble, synchronization)
- Signal degradation, repeaters/amplifiers

---

## Unit 10 — The Middle Boxes: Where Network Devices Live

### The premise

"The client is not directly connected to the server. There are switches in the middle, there are routers, there are proxies, there are CDNs, there are reverse proxies... there are load balancers. What are those suckers doing in the middle? They look, they **peek the content** and when they peek at the content, they take **finite amount of time** because they make decisions based on this peeking of the data."

The OSI framework's payoff: you can say exactly *how deep* each device peeks, and that determines its power and its cost.

### The peek depth by device

| Device | How deep it peeks | What it can decide |
|---|---|---|
| **Switch** (L2) | Destination MAC only | Which port to forward out; no broadcasting |
| **Router** (L3) | IP addresses | "Is this packet for me?" — where to route next |
| **Firewall** (L4) | IP addresses + ports | Block unwanted traffic ("certain applications from sending data or blocks certain unwanted packets") |
| **L4 proxy / LB** (L4) | Ports; can rewrite | "If you are going to port 8080, I want you to rewrite the packet and send it to this address instead" |
| **L7 proxy / CDN / reverse proxy** | Decrypted application content | Path-based routing, caching, "they decrypt everything you send, look at it, cache it and then resend it" |

### Transparent firewalls — what the world can see about you

The critical insight: **everything below L5 is public.** "Everyone can read your ports, everyone can read your IP addresses. That is why... it's public. It's never encrypted. Never."

That's why your ISP can block you from any website — it just drops packets by IP. "A lot of governments use transparent proxies."

But they **cannot read your application data** without decrypting — "if you go to the application, you need to decrypt, you need to stop the session and decrypt it. And to do that, you need to serve the certificate of the server, which you don't have." The exception the lecturer gives: **Kazakhstan**, "which literally forces a certificate into every computer in their citizens" for MITM inspection — "most of the world, they are not like that."

### The two-session split of a reverse proxy

The deepest point in this section. When an L7 proxy terminates your connection and opens its own to the backend: "you effectively now need a completely different session between those two parties... this session is one session and this is one session. This is the final endpoint to you as a client... that's why this is called a **reverse proxy**. Your true final destination is actually this backend server which you don't know anything about. **That's how Google works.**"

**Proxy vs reverse proxy:** "a proxy goes through you... you know this is your final destination, but the proxy makes that request on your behalf. [A reverse proxy] is your final destination, but your true final destination is actually this backend server which you don't know anything about."

### The cost of depth

Deeper peeking = more latency: L7 proxies "are way slower than a router or a firewall" — a router goes "up to layer three and go down," while an L7 proxy must "go all the way up and then re-transmit everything back down" (plus terminate one TLS session and originate another).

### The summary the lecturer builds to

> "One of the most important things specifically when you write applications... it's really good to understand where your application sits. If data passes through your application, it's good to know where your application is so that you can talk through it, so you can know what you can optimize, what do you have access to and what can you improve."

Plus the standing caveat: "each device doesn't have to map to all the seven layers... the line is blurred sometimes."

### What the Source Got Right

- The "peek depth" framing — every middlebox is defined by *which layer it reads*, and L5 is the boundary between public and encrypted
- The transparent-firewall section as a real-world explanation of why TLS matters (ISPs/governments can route-block but not read)
- The Kazakhstan counterexample proving the rule
- The two-session split as the true definition of a reverse proxy (and "that's how Google works")
- The latency gradient: MAC peek < IP peek < decrypt-and-re-encrypt

---

## Unit 11 — OSI vs TCP/IP: The Comparison

### The criticism of OSI

"OSI model has too many layers which can be hard to comprehend" — specifically layers 5 and 6: "It's definitely simpler to deal with layer five, six and seven. That's just one layer, which is the application — and that's what the TCP IP model does."

### What the TCP/IP model is

"Layer five, six and seven... just one layer, which is the application." So the TCP/IP stack is effectively: **application (OSI 5+6+7) / transport (TCP) / internet (IP) / data link (which absorbs OSI 1+2)**. The lecturer: "it's literally the TCP IP, which is TCP and IP and the data link, right, which is the MAC addresses and all that jazz."

### The numbering confusion — the practical warning

The lecturer's concrete advice: **avoid numbering TCP/IP layers**, because the two models share vocabulary for different things:

- **L4 and L3 are identical** in both models — same language: "layer four, layer three are identical to the OSI model"
- **L5 differs dangerously:** "if you say layer five, you have to specify, oh, this is the TCP IP model. That means this is the application while layer five [in OSI] is actually the session layer"
- **L2 and below:** "data link and physical layer is really something never talked about as far as I know" in TCP/IP discussions

So "layer 4 proxy" means the same thing in both models, but "layer 5 app" depends on which model you're speaking.

### The lecturer's stance

Mixed feelings, stated honestly: "which I, I have mixed feeling about this. It's up to you really." He **prefers the OSI model's finer granularity** — "the more I understand, the more I build backend applications, the more I started to read and understand we really need that low level [granularity]... not all the time" — while conceding "TCP IP is definitely a simpler model" and that boundary arguments are "all people speculating at the end of the day."

### What the Source Got Right

- Naming the exact mapping between the models (5/6/7 → application; 1+2 → data link)
- The numbering-collision warning as the practical takeaway
- Presenting the granularity-vs-simplicity trade-off as a genuine preference, not dogma
- Closing on usability ("where your app sits... what you can optimize") rather than taxonomy

---

## Open Questions Carried Forward

| # | Question | Why it matters |
|---|----------|---------------|
| 1 | What is protocol ossification? | Flagged by the lecturer as a critical, possibly standalone topic — a lecture teaser, not covered in this lecture's scope |
| 2 | Wire-level details of each layer (TCP handshake mechanics, IP packet structure, frame structure, modulation) | Deferred to the dedicated lectures: Internet Protocol (20), TCP (22), and others in this section |

*Documented after discussing the transcript. All nine main questions covered (Units 1–9: the "why" of a standard model, the definition of OSI, then each layer 7 → 1). Gap pass completed: Unit 10 covers the middle-boxes section (peek depth per device, the public-below-L5 boundary and transparent firewalls/ISPs/the Kazakhstan exception, proxy vs reverse proxy, the two-session split, "that's how Google works"), and Unit 11 covers the TCP/IP model comparison (layer mapping, numbering collision warning, the lecturer's preference for OSI granularity). No nested sub-files were needed — all discussion happened inline.*
