# Lecture 18: Protocol Properties

## Status: Complete

> ### 📋 Read this before continuing
>
> This lecture is the theoretical foundation of the protocols section: it answers **what a protocol is** (a system of rules that lets two parties communicate, shaped by the problem it was designed to solve) and **what protocol properties are** (the design dimensions every protocol must decide on). The lecturer explicitly says "take this with a grain of salt" — don't memorize the properties as rules, because *performance and whether your use case is solved* are what matters.
>
> | Question | Status | Covered in |
> |---|---|---|
> | 1. What is a protocol? | ✅ Documented | Units 1-2 |
> | 2. What are protocol properties? | ✅ Documented | Unit 3 |
> | 3. What are the transfer mode and data format for HTTP? | ✅ Documented | Unit 4 |
>
> | Unit | Content |
> |---|---|
> | **1** | **What is a protocol?** — a system of rules that lets two parties communicate, where both must follow the same rules |
> | **2** | **Protocols encode assumptions about their environment** — the TCP/Homa case: rules are bets on a specific problem context |
> | **3** | **What are protocol properties?** — the eight design dimensions, with the unifying idea that each is a bet on the problem being solved |
> | **4** | **HTTP case study: transfer mode + data format** — text vs binary format, message-based HTTP over stream-based TCP, and the framing problem |
>
> **Nested questions documented:** [does the HTTP library handle framing?](nested-http-library-framing.md) · [how can HTTP be "text" over TCP's stream?](nested-http-text-over-tcp-stream.md)

---

## Unit 1 — What is a Protocol?

### Direct Answer

A **communication protocol** is a system of rules that allows two parties to communicate. The critical condition: **both parties must follow the same rules**. The rules are a shared contract — if only one side speaks the protocol, communication doesn't happen.

The lecturer's framing from the transcript:

> "A communication protocol? It's the definition is it's a system that allows two parties to communicate and a system of rules to be specific. It's a set of rules. ... And if both parties follow the same rules, they can communicate."

### Mental Model

Think of a protocol as an agreed-upon syntax for a conversation. One party says "one, two, three, four" and the other knows what that means — not because the data itself is meaningful, but because both sides pre-agreed on the format. Remove the shared agreement and the same bytes are just noise.

Two things make this definition complete:

1. **Mutual agreement** — the rules must be identical on both sides. A protocol is only useful as a *shared* contract.
2. **It has properties (structure)** — the rules aren't arbitrary; they're organized into categories (data format, transfer mode, addressing, etc., covered in Unit 3).

---

## Unit 2 — Protocols Encode Assumptions About Their Environment

### The full definition the lecture teaches

The "system of rules" definition is only half the answer. The lecture's real thesis is:

> A protocol is a system of rules that lets two parties communicate, **where the rules are shaped by a specific problem the protocol was designed to solve**.

Match the environment the protocol was designed for, and it works well. Put it in a different environment, and its design bets start working against you.

> "What are we building to begin with? Why are we building it? What problems are we trying to solve? ... Every single protocol out there was designed to solve a problem." — lecture transcript

### Case study: TCP designed for a world that no longer exists

**TCP's original problem (1960s):** communicate reliably across a slow, unreliable, congested network where machines didn't trust each other.

**The bets TCP made:**

| TCP's bet | Why it was correct then |
|---|---|
| Reliability by default (ACKs, retransmission) | The network would lose and reorder data |
| Congestion control — back off on packet loss | Loss was assumed to be caused by congestion |
| Stream-based transfer (continuous bytes) | Designed for low-bandwidth links, not discrete-message workloads |

**The new problem (modern data centers):** machines in close proximity exchanging massive volumes of data. The environment has changed:

- Links are short, controlled, high-bandwidth — congestion from the wider internet is no longer the assumption
- Data volumes dwarf anything the 1960s designers imagined

**Result:** TCP is "hitting its limit" in data centers. Its caution (throttling on loss, heavy connection state) becomes overhead in an environment that doesn't need it. As the transcript says: "TCP was designed for low bandwidth network back then. They didn't imagine that we're going to have data centers that with very close proximity, that sends tons of data right back and forth."

### The response: Homa (2022)

The lecturer mentions **Homa**, a 2022 **research paper — not even an RFC yet**. Its striking design choice: it attempts a **message-based TCP**.

Why message-based? TCP is *stream-based* — just a river of bytes. Any layer built on top (like HTTP/1.1) must parse that stream to find where one message ends and the next begins, which the lecture flags as a real cost. Homa rethinks the transfer mode itself for the data center environment.

### What the Source Got Right

- Protocols exist because someone had a different problem than the protocol before it
- The TCP → data-center failure as the clearest illustration of "environment shifts break protocol assumptions"
- New protocols (Homa) get invented precisely because old ones hit their limits
- This thread runs through the whole section: DNS, UDP, HTTP, gRPC, TLS each solve a different problem

### The Thread Running Through the Section

The lecturer frames this lecture as the intro to a lineup of "the classics," each designed to solve a different problem:

**DHCP · UDP · HTTP · gRPC · Apache Thrift · FTP · SMTP**

Each exists in the syllabus because someone had a different problem than the protocol before it:

| Protocol | The problem it solved |
|---|---|
| TCP | Reliable ordered delivery across an unreliable network |
| UDP | Fast, connectionless delivery where reliability isn't needed |
| HTTP | Request/response document retrieval over TCP |
| DNS | Mapping human-readable names to IP addresses |
| DHCP | Automatic IP configuration for hosts joining a network |
| FTP | File transfer between hosts |
| SMTP | Mail delivery between servers |
| TLS | Securing communication without pre-shared knowledge |
| gRPC | Efficient, typed RPC between services |
| Homa | Message-based transport optimized for data centers |

The lecturer also flags that Homa may be covered in a podcast attached to the course — "it's just a paper. It's not even an RFC" — since it's too new (2022) to be a classic.

---

## Unit 3 — What Are Protocol Properties?

### Direct Answer

Protocol properties are the **design dimensions** of a protocol — the categories of decisions a protocol designer makes when building it. The lecturer's framing: "a protocol is designed with this sets of properties, effectively the rules."

So the properties ARE the rules, organized into categories. They are not a checklist to memorize:

> "Don't memorize these properties, don't live by them, don't by these rules. You know what matters is at the end of the day performance. ... And whether your use case is solved with this protocol or not."

The properties are also explicitly personal/experiential — "collected from my personal experience, what I saw as a pattern and from certain resources as well" — not a formal specification.

### The eight properties

**1. Data format** — what the wire looks like.

- *Text-based:* readable on the wire (assuming unencrypted) — JSON, XML, HTTP plaintext
- *Binary:* not meant for human eyes, optimized for machine efficiency — protobuf, RESP (Redis serialization)
- Notable case: **HTTP/2 kept the "hypertext" name but its on-the-wire transfer became fully binary.** The API stayed identical (same `GET` requests, same behavior); only the wire format changed underneath.

**2. Transfer mode** — how data is framed.

- *Message-based:* discrete messages with a start and an end — UDP, CDB (a request has a start and end; a response has a start and end)
- *Stream-based:* a continuous flow of bytes, no start or end — TCP, RTP (audio/video)
- For message-based protocols, messages larger than the MTU are split across multiple IP packets with fragment IDs so the destination can reassemble them.
- The cost of stream mode: HTTP on top of TCP must parse the byte-stream to find message boundaries — expensive on both client and server. This limitation motivates message-based redesigns like [Homa](#unit-2--protocols-encode-assumptions-about-their-environment).

**Mechanics detail — message datagrams and MTU:** the transcript walks through the lifecycle of a message-based message: "each message sits in a nice tucked in IP packet." One message = one IP packet — that's the "nicely tucked in" part. But if the message **exceeds the MTU** (maximum transmission unit — the largest payload an IP packet can carry on a given link, typically ~1500 bytes on Ethernet), it can't fit in one packet. The sender then breaks it into multiple IP packets, each labeled with a **fragment ID** so the destination can group them back together. The receiving side reassembles the fragments into the original message. This is why message-based doesn't mean "unbounded size" — the message abstraction is preserved across multiple packets by the fragmentation/reassembly convention.

**3. Addressing** — where is this going, and from where?

- *DNS* (layer 7): `google.com` — human-readable names
- *IP address* (layer 3): `9.7.3.2` — network location
- *MAC address* (layer 2): hardware address, resolved via ARP
- The framing changes by layer: layer-2 frames carry source/destination MAC; IP packets carry source/destination IP; applications address via DNS.

**Mechanics detail — how addressing traverses the layers:** the transcript gives a concrete walkthrough of a single request, which shows the address *changing* as the message descends:

1. **You use a name** — `google.com` (DNS, layer 7)
2. **DNS resolution** turns the name into an IP address (layer 3)
3. **ARP** (Address Resolution Protocol, layer 2) resolves that IP into a MAC address, because delivery on the local link is by MAC
4. At the hop, the **source/destination pair changes by layer**: a layer-2 frame carries source MAC + destination MAC; inside it, the IP packet carries source IP + destination IP
5. At the destination, the IP packet "resolves eventually to a MAC" and is delivered to the device

The point: "where are you going?" has a **different answer at each layer**, and the protocol stack rewrites the addressing as the message travels. DNS is not an alternative to IP — it's a naming layer that *starts the chain* which ends in MAC-level delivery.

**4. Directionality** — who may send, and when?

- Unidirectional or bidirectional
- Full duplex (both send simultaneously) vs. half duplex (only one direction at a time) — WiFi is half duplex

**5. Protocol state** — does the protocol remember between exchanges?

- *Stateful:* gRPC, TCP, Apache Thrift
- *Stateless:* UDP — no state at the protocol level

**6. Routing** — how does it behave with gateways and proxies?

Example from the lecture: you send a request to `google.com`, but the TCP connection may go to a **proxy first**. Your final destination is Google, but your *immediate* destination is the proxy. This requires DNS resolution to find the real IP before the TCP connection can be established. Does your protocol play well with this indirection?

**7. Reliability, flow control, congestion control** — does it guarantee delivery?

- **UDP: none of it.** Just send and hope.
- **TCP: all of it** — flow control, congestion control, reliable delivery, guaranteed retransmission.

**8. Error management** — what happens when something goes wrong?

- Specific error messages/codes (CDP has dedicated ones)
- Timeout behavior — retry or not?

### The unifying idea: every property is a bet on the problem

The eight properties aren't independent checkboxes — they're the concrete expression of Unit 2's thesis:

| Property | The question it answers | Example tension |
|---|---|---|
| Data format | Machine speed or human debuggability? | protobuf (binary, fast) vs JSON (text, readable) |
| Transfer mode | Discrete messages or continuous flow? | UDP (messages) vs TCP (stream) |
| Addressing | Who is the destination, at which layer? | DNS vs IP vs MAC |
| Directionality | Can both talk at once? | Full duplex vs WiFi half duplex |
| Protocol state | Remember between exchanges? | TCP (stateful) vs UDP (stateless) |
| Routing | Survive proxies/gateways? | Immediate vs final destination |
| Reliability | Guarantee delivery? | UDP (no) vs TCP (yes) |
| Error management | Recover how? | Retry vs fail-fast |

UDP skips reliability because some problems prioritize speed. TCP bets on reliability because other problems prioritize correctness. The properties are where the philosophy of Unit 2 becomes concrete mechanics.

### What the Source Got Right

- The "grain of salt" framing — properties are vocabulary for analyzing protocols, not laws
- The list is experiential, not canonical — "there are ones you might discover that is not even listed here"
- The connection back to purpose: properties only make sense relative to the problem
- Performance and use-case fit as the real criteria, not rule-following

### What the Source Left Open (for later lectures)

- Each property is expanded in its own lecture: UDP (lecture 21) for statelessness/message mode, TCP (lecture 22) for reliability/flow/congestion control, HTTP lectures for routing/framing
- The "message-based TCP" idea (Homa) is a paper, not a standard — it may appear in a podcast referenced by the lecturer

---

## Unit 4 — HTTP Case Study: Transfer Mode and Data Format

### Direct Answer

HTTP answers the two properties differently at different layers — and this is the first concrete application of the properties framework:

| Property | HTTP/1.1 | HTTP/2 |
|---|---|---|
| **Data format** | Text (human-readable on wire) | Binary on wire, same API |
| **Transfer mode** | Messages over TCP's stream → framing needed | Binary frames over TCP stream |

### Data format: text, with a twist

HTTP/1.1's data format is **text-based** — the lecture lists it explicitly: "plaintext, JSON, XML, http normal text." On the wire (assuming no encryption):

```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
```

**The twist — HTTP/2 changed the wire format to binary** while keeping the API identical:

> "Although it should be [HTTP/2] is actually stands for hypertext... But in the wire, it's a completely binary. They changed the in wire transfer, but the compatibility stays at the API level. Nothing changed. You still make the same git request. They still do the same thing."

The lesson: **data format is a wire-level property, not an API-level one.** The two can evolve independently. (Why HTTP/2 made this bet is covered in lecture 27.)

### Transfer mode: message-based over stream-based

Subtlety the lecture leans on:

- **HTTP itself is message-based** — a request has a start and an end; a response has a start and an end.
- **But it rides on TCP, which is stream-based** — no start, no end, just a continuous river of bytes.

> "Another trick is if you build HTTP on top of TCP, now you have the client has to parse the stream, which is TCP stream, just a collection of bytes to actually understand where requests start and ends. That's the problem that we have here."

So HTTP must solve a **framing problem** TCP handed it: given a stream with no boundaries, how do you know where one message ends and the next begins? HTTP's answer is framing rules:

- **`Content-Length` header** — tells the receiver exactly how many bytes the body is, so it can count them off the stream
- **`Transfer-Encoding: chunked`** — body arrives in chunks, each with a size prefix, ending with a zero-length chunk

### The cost the lecture flags

> "That's why the cost of the client side and the server side to do this parsing, to find out these messages is actually is one of the limitation. ... That's why this new protocol that's called homo is actually tried to do a message based TCP."

That parsing overhead — extracting message boundaries from a raw byte stream — is paid on both ends of every HTTP request. It's precisely what motivated the [Homa redesign from Unit 2](#unit-2--protocols-encode-assumptions-about-their-environment).

### Do we do this manually? No — with caveats

HTTP libraries and frameworks handle framing automatically; you work with request/response objects, never raw streams. The "cost of parsing" is paid inside the library, invisibly. But the knowledge matters where the abstraction leaks (keep-alive framing bugs, proxies, raw protocol work, building your own protocols) — see [nested-http-library-framing.md](nested-http-library-framing.md).

### Clearing up the apparent contradiction

How can HTTP be "text" while its transport is a TCP stream? Because **properties belong to specific protocols at specific layers**: TCP (layer 4) chose stream transfer; HTTP (layer 7) chose text messages. The text lives *inside* the stream. Also, "text vs binary" is not "bytes vs not-bytes" — every format is bytes; the question is whether the bytes are structured for human readability. This mirrors the Lecture 15 insight (stateless HTTP over stateful TCP): layers compose freely. Full explanation in [nested-http-text-over-tcp-stream.md](nested-http-text-over-tcp-stream.md).

### The bet HTTP made

Connecting back to Unit 3's synthesis — the properties are levers, the problem decides the pull:

| HTTP's bet | Payoff | Price |
|---|---|---|
| Text data format | Human-readable, easy to debug, easy to extend | Larger messages, parsing cost |
| Message framing over TCP stream | Reuse battle-tested TCP | Boundary-parsing overhead on both ends |

HTTP/2 later re-bet on **binary framing** for efficiency (smaller messages, cleaner parsing) while deliberately keeping the API stable so nothing broke. Same problem, re-pulled levers.

### What the Source Got Right

- Data format as an independent, wire-level property (HTTP/2 proof)
- The stream-over-message tension between TCP and HTTP as a genuine limitation
- The parsing cost as a real motivation for new protocols (Homa)
- The framing problem as something solvable by convention (Content-Length / chunked) rather than by the transport

### What the Source Missed

- **Framing mechanics:** how `Content-Length` and chunked encoding actually work byte-by-byte was not detailed (briefly covered above)
- **Libraries hide framing entirely** — worth stating explicitly, since a learner might assume they must implement this
- **Pipelining:** HTTP/1.1 pipelining (sending multiple requests without waiting) made framing bugs even more damaging — connection-level ordering matters

---

## Open Questions Carried Forward

| # | Question | Where it will be answered |
|---|----------|--------------------------|
| 1 | How does congestion control actually detect and react to congestion? | TCP lecture (22) / possible podcast |
| 2 | What is Homa's concrete design, and does message-based transport beat stream-based in data centers? | Referenced podcast / further research |
| 3 | Why did HTTP/2 change the wire format to binary while keeping the same API? | HTTP/2 lecture (27) |

*Documented after discussing the transcript. All three questions from this session covered: (1) What is a protocol — Units 1-2 with the shared-rules contract and the TCP/Homa environment-assumption case study; (2) What are protocol properties — Unit 3 with all eight properties and the "every property is a bet on the problem" synthesis; (3) HTTP's transfer mode and data format — Unit 4 with the framing problem, text-vs-binary format, and two nested clarifications (library framing, text-over-stream layering).*

*Gap pass completed: three under-covered source points addressed in place — (a) the lecture's "classics" lineup (DHCP · UDP · HTTP · gRPC · Apache Thrift · FTP · SMTP) added to Unit 2; (b) the MTU/fragment-ID mechanics for message-based transfer expanded in Unit 3 property 2; (c) the DNS → IP → MAC layered addressing walkthrough expanded in Unit 3 property 3.*
