# Lecture 11: Long Polling — Built Up, Unit by Unit

## Status: In Progress 🔄 (Units 1–3 studied · Units 4–7 not yet delivered)

> ### 📋 Read this before continuing
>
> | Units | State | What that means |
> |---|---|---|
> | **1–3** | ✅ **Studied** | Worked through together. Concepts questioned and confirmed. |
> | **4–7** | ⬜ **Not yet delivered** | Only an outline. Nothing here is written from discussion. |
>
> **Unit 1 is the mechanism, Unit 2 the payoff, Unit 3 the bill — and the bill contains this lecture's thesis.** Long polling is only safe when the server stores results durably. Unit 4 shows that's exactly why Kafka chose it.
>
> **Unit 3 tests a prediction.** [Lecture 10](lecture-10-polling.md) Unit 7.4 predicted that long polling relocates short polling's cost rather than removing it. Hussein appears to confirm this. Unit 3 will press it.
>
> **Two claims are flagged for testing.** See *Claims flagged for testing* below.

> **How to read this doc:** Each unit builds on the previous. Each unit ends with a **Checkpoint** — answer it in your own words before moving on.

---

## Lecture Map

| Unit | Content | State |
|---|---|---|
| **1** | **The trick** — same request, the server just doesn't reply | ✅ Studied |
| **2** | **Why it works** — the empty response is deleted | ✅ Studied |
| **3** | **The cost** — polling moved server-side, and disconnect is not a pro | ✅ Studied |
| 4 | Why Kafka chose it — backpressure, not the trick | ⬜ |
| 5 | The remaining gap — it is still not real time | ⬜ |
| 6 | The demo — the event-loop trap in the `while` loop | ⬜ |
| 7 | Recap — and the road to SSE | ⬜ |

### Claims flagged for testing

| # | Claim from the transcript | Why it needs testing | Status |
|---|---|---|---|
| 1 | ✅ **ANSWERED in Unit 3.3** — true against a long-held request, **false against short polling**. Long polling turns Unit 10's *state* back into a *delivery*, so a disconnect can destroy the result | Closed |
| 2 | "Long polling is used by Kafka" | [Lecture 10](lecture-10-polling.md) Unit 7.3 said this is true but beside the point. This transcript gives the *real* reason — **backpressure** — which is neither efficiency nor trickery | ⏳ Unit 4 |
| 3 | *"I wouldn't say simple, to be honest. There's more nuance here"* | The demo contains a `while` loop that **kills the Node event loop**. Is that a transcription slip, or is long polling in Node genuinely this hard? | ⏳ Unit 6 |

### And one prediction from Lecture 10 to verify

[Lecture 10](lecture-10-polling.md) Unit 7.4 claimed:

> **Short polling: stateless server, wasteful traffic. Long polling: efficient traffic, N held connections.** You choose which cost you pay.

Hussein's own summary in this lecture: *"We effectively move the polling from the client side to the server side."*

**If that holds, Unit 7.4 was right.** Unit 3 settles it.

---

## Unit 1 — The Trick

### 1.1 — One Sentence

> **The client sends the exact same request. The server just doesn't respond.**

That's the whole mechanism. Everything else is consequence.

> *"You will make the poll request. Same thing. A request to poll. But the server just doesn't respond. Doesn't write anything to the socket. Instead of saying immediately, 'hey, it's not ready', it just waits there."*

### 1.2 — The Same Problem, Two Behaviours

Recall [Lecture 10](lecture-10-polling.md) Unit 1's setup: you want to know if your video finished transcoding. The **request is identical** in both designs:

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    rect rgb(255, 235, 238)
    Note over C,S: SHORT POLLING - Lecture 10
    C->>S: GET /status?jobId=a7f3
    S-->>C: 200 {progress: 40%} immediately
    Note over C,S: client waits 5s, repeats
    end

    rect rgb(226, 242, 253)
    Note over C,S: LONG POLLING - this lecture
    C->>S: GET /status?jobId=a7f3
    Note over S: request stays open,<br/>nothing written to the socket
    Note over S: progress 40%, 50%, 60%...
    S-->>C: 200 {done: true, url: "..."}
    end
```

Same verb. Same URL. Same method. **Different server behaviour — silence instead of an answer.**

### 1.3 — What "Not Responding" Actually Means

This is where precision matters, because "the server doesn't respond" sounds like a hang or a bug. It isn't:

| It's **not** | It **is** |
|---|---|
| The server is down | The server is healthy and waiting |
| The request timed out | The request is **alive and counted** |
| The client gave up | The client is blocked, holding a connection |
| An error | **A deliberate, temporary state** |

> **An unanswered request is not a failed request. It's a pending one.**

The server has accepted the work of "answer this eventually" and holds the obligation open. That obligation has a cost — but so does every unanswered request everywhere.

### 1.4 — Why It Isn't the Same as Push

[Lecture 8](lecture-08-push.md) taught push, so the confusion is natural: *"isn't this just push?"*

```mermaid
flowchart LR
    A["PUSH - Lecture 8"] --> A1["server decides WHEN<br/>to send"]
    A --> A2["client has no say<br/>in the timing"]
    A --> A3["client may be flooded<br/>with volume it cannot absorb"]

    B["LONG POLLING"] --> B1["client decides WHEN<br/>to ask"]
    B --> B2["one question,<br/>one answer"]
    B --> B3["client sets the pace"]

    style A fill:#ffcdd2,stroke:#c62828
    style B fill:#c8e6c9,stroke:#388e3c
```

**The client still drives.** It asked, so it will get an answer. The server can't send anything unasked.

> **Push: the server talks, the client listens. Long polling: the client asks, the server waits for a reason to answer.**

This is why the name says *polling* and not *push*. The client is still the initiator of every exchange. The server's silence is not initiative — it's patience.

### 1.5 — The Detail That Sets Up Everything

Hussein is precise about what the server does while silent:

> *"Since it knows that the job is not ready, the response is not ready, it will just say, okay. I'll just do my own thing and I'll come back later."*

**The server keeps working.** It does not block, does not spin, does not sit in a busy loop. It registers "client `a7f3` wants to be told" and gets on with it.

That means:

- One silent request costs **one held connection**, not one held thread
- Many silent clients cost **many connections**, not many threads
- The server is *free to be busy* — but it is **not free to be absent**

Hold onto that last line. It's Unit 3.

---

## Unit 2 — Why It Works

Unit 1 was the mechanism. This unit is the payoff — and the payoff is **bigger than the transcript claims.**

### 2.1 — What Gets Deleted

[Lecture 10](lecture-10-polling.md)'s core finding was the waste:

> 47 of every 48 polls answered *"not ready"* — information the client already had, over a full HTTP round trip.

Hussein's claim:

> *"You did not really waste bandwidth by sending multiple pull requests. The moment Kafka gets a message, it writes back to the client's response. And that's the beauty here… we're less chatty."*

Correct — and the mechanism is subtraction:

```mermaid
flowchart LR
    A["SHORT POLLING<br/>4 minute job, 5 second interval"] --> A1["req 1 - not done"]
    A1 --> A2["req 2 - not done"]
    A2 --> A3["req 3 - not done"]
    A3 --> A4["...47 of these ..."]
    A4 --> A5["req 48 - DONE"]

    B["LONG POLLING<br/>same job, same interval"] --> B1["one request,<br/>held open 240 seconds"]
    B1 --> B2["DONE,<br/>arrives the instant it is ready"]

    style A fill:#ffcdd2,stroke:#c62828
    style B fill:#c8e6c9,stroke:#388e3c
```

> **Long polling doesn't make the empty response cheaper. It deletes it.**

| | Short polling | Long polling |
|---|---|---|
| Empty responses | 47 per job | **Zero** |
| Meaningful responses | 1 | 1 |
| **Ratio of waste to signal** | **47 : 1** | **0 : 1** |

That is the entire trick. Everything else in this unit is a consequence.

### 2.2 — The Arithmetic

Same job — 4 minutes, 5-second interval:

| | Short polling | Long polling |
|---|---|---|
| Requests | **48** | **1** |
| Bytes on the wire | ~28.8 KB | ~0.6 KB |
| Connection-handshake events | 48 (or 48 keep-alive events) | 1 |
| Server lookup operations | 48 | 1 |

**48× fewer requests. ~48× fewer bytes.** Against [Lecture 10](lecture-10-polling.md) Unit 5.2 — 10,000 users × 2,000 rps — this is the difference between a fleet you scale for emptiness and one you don't.

### 2.3 — The Bigger Win Nobody Mentions

Here's the part the transcript undersells. The bandwidth saving is real, but it isn't the best part:

> **Long polling deletes the polling interval as a source of latency.**

Recall [Lecture 10](lecture-10-polling.md) Unit 2.4: short polling's latency floor *is* the interval, because the job could finish at any moment between polls.

| | Short polling (5s interval) | Long polling |
|---|---|---|
| Best case | 0s | ~0s |
| **Average case** | **2.5s** | **~0s** |
| **Worst case** | **5s** | ~0s |
| What the client experiences | *usually waiting* | *notified* |

With short polling, the job finished at second 237 of 240 and the client waited 3 more seconds anyway. **The work was done and nobody knew.** With long polling, the response is written the instant the job completes.

> **That is a change in kind, not degree.** Short polling means the client is *usually behind*. Long polling means the client is *only ever as late as the network*.

For a chat app, a live dashboard, or anything where "did it just happen?" is the question — this matters more than bandwidth.

**Two wins, and they're independent:**

| Win | What it removes |
|---|---|
| **Bandwidth** | Empty responses |
| **Latency** | The polling interval as a delay source |

Most treatments of long polling mention only the first.

### 2.4 — The Structural Change

Now the deepest form of this. Look at the *formula*:

```mermaid
flowchart TB
    A["SHORT POLLING<br/>requests = duration / interval<br/>4 min job at 5s = 48 requests<br/>6 hour job at 5s = 4,320 requests"]
    B["LONG POLLING<br/>requests = 1<br/>no matter how long the job runs"]

    A -->|"eliminate the<br/>division, not the<br/>numerator"| B
    B --> C["Lecture 10 Unit 5.3's pathology -<br/>'long jobs waste the MOST' -<br/>is DELETED, not reduced"]

    style A fill:#ffcdd2,stroke:#c62828
    style B fill:#c8e6c9,stroke:#388e3c
    style C fill:#fff9c4,stroke:#fbc02d
```

| Job duration | Short polling (5s) | Long polling |
|---|---|---|
| 30 seconds | 6 requests | **1** |
| 4 minutes | 48 requests | **1** |
| 6 hours | 4,320 requests | **1** |
| 3 days | 518,400 requests | **1** |

> **Long polling doesn't reduce the waste ratio — it removes the formula.** Request count stops being a function of duration.

Recall [Lecture 10](lecture-10-polling.md)'s most uncomfortable finding: **the workloads polling is *for* are the workloads where it wastes most**, because waste = `1 − interval/duration` *rises* with duration. Long polling **kills that pathology outright.** A three-day job costs the same single held connection a three-second job does.

**That's why long polling is the correct answer for long work — not just a cheaper version of the wrong one.**

### 2.5 — The Objection, and Why the Runtime Matters

The obvious objection: *aren't you just re-creating the "client stuck holding a request" problem from* [Lecture 9](lecture-09-sync-vs-async.md)?

Good instinct — and the answer is Lecture 9's own distinction:

```mermaid
flowchart LR
    A["LONG POLL = AWAITING<br/>thread freed,<br/>connection held"] --> C["in Node.js this is<br/>nearly free:<br/>event loop, no thread<br/>per connection"]
    B["LONG POLL = BLOCKED<br/>thread held,<br/>connection held"] --> D["in thread-per-request<br/>servers this is<br/>expensive"]

    style A fill:#c8e6c9,stroke:#388e3c
    style D fill:#ffcdd2,stroke:#c62828
```

From [Lecture 9](lecture-09-sync-vs-async.md) Unit 1:

| | Thread | This request |
|---|---|---|
| **Blocked** | ❌ Suspended | ❌ Stopped |
| **Awaiting** | ✅ **Freed** | ✅ Paused, connection held |

**A long poll is *awaiting*, not *blocked*** — provided your runtime models it that way.

| Runtime | Long poll cost |
|---|---|
| **Node.js** (event loop, no thread per connection) | Cheap — this is Unit 6's world |
| **Go** (goroutine per connection, cheap) | Cheap |
| **Java / thread-per-request** | Expensive — a real thread parked per client |
| **PHP** (no shared memory, process model) | Awkward — long polls fight the model |

> **The same long polling is nearly free in Node.js and ruinous in a thread-per-request stack.** [Lecture 10](lecture-10-polling.md) Unit 4's "no new protocol" claim holds — but *"cheap"* is a property of the mechanism *and your runtime*.

That last row is why Unit 6's demo — a promise in Node.js — matters, and why it apparently needed extra work to get right.

### 2.6 — What Does NOT Improve

Planting Unit 3, because a unit that only lists wins isn't an analysis:

| | Short polling | Long polling | Verdict |
|---|---|---|---|
| Empty responses | 47 | **0** | ✅ Fixed |
| Detection latency | up to the interval | ~0 | ✅ Fixed |
| Requests scale with duration | Yes | **No** | ✅ Fixed |
| **Connections held** | ~1 briefly | **1 for the whole wait** | ❌ **Worse** |
| **Where the wait is stored** | nowhere (stateless) | **in the request** | ❌ **Worse** |
| **Survives server restart** | Yes (job in shared store) | **No — the wait dies with it** | ❌ **Worse** |
| **Client disconnect mid-wait** | Nothing lost, just stop polling | **Result may be undeliverable** | ❌ **New failure** |

Hussein calls disconnect a **pro** — Unit 3 tests that, because the table above suggests it might be the sharpest edge of all.

---

## Unit 3 — The Cost

Unit 2 listed five things long polling makes worse. This unit prices them — and lands on a conclusion that reframes the whole mechanism.

It settles two claims: [Lecture 10](lecture-10-polling.md) Unit 7.4's prediction, and the "we can disconnect" pro.

### 3.1 — The Scales Differently

The single most important economic fact about long polling:

```mermaid
flowchart TB
    subgraph SP["SHORT POLLING"]
        A["cost scales with<br/>REQUEST RATE<br/>users / interval"] --> A2["an idle client costs<br/>almost nothing"]
    end

    subgraph LP["LONG POLLING"]
        B["cost scales with<br/>WAITING CLIENT COUNT"] --> B2["each waiter holds a<br/>connection, a file<br/>descriptor, and buffers"]
    end

    style SP fill:#e3f2fd,stroke:#1976d2
    style LP fill:#ffcdd2,stroke:#c62828
```

**Different denominators.** That single fact makes them incomparable without numbers:

| | Short polling | Long polling |
|---|---|---|
| 10,000 users, jobs not yet done | ~a handful of requests in flight | **10,000 connections held** |
| Per-waiting-client cost | ~0 | One socket + fd + buffers |
| Idle client | Free | Free (not polling) |
| Waiting client | Cheap and brief | **Held for the whole duration** |

And here's the hard wall — **every connection is a file descriptor.** Remember [Lecture 9](lecture-09-sync-vs-async.md)'s fd clarification? It's back, and it's the constraint that decides your ceiling:

| Resource | Per held long poll |
|---|---|
| File descriptor | 1 — bounded by `ulimit -n` (often 1024, sometimes 65535) |
| Socket buffers | ~4–64 KB of memory |
| Event-loop bookkeeping | Connection object, timers, pending-response state |

10,000 waiting clients ≈ **10,000 fds and roughly 160–640 MB of buffer memory**, depending on runtime.

> **You don't out-scale this with better hardware. You hit a file-descriptor limit, and then you fail.**

### 3.2 — The Relocation, Confirmed

The [Lecture 10](lecture-10-polling.md) Unit 7.4 prediction was:

> *Long polling gives you efficient traffic but N held connections. You choose which cost you pay.*

Hussein's own words:

> *"We effectively move the polling from the client side to the server side. Which is… polling. But hey, it works."*

**Prediction confirmed.** And the loop's ownership visibly changes:

```mermaid
flowchart TB
    subgraph CL["CLIENT-DRIVEN POLLING"]
        P1["client"] -->|"asks every 5s"| P2["server answers immediately"]
    end

    subgraph SV["SERVER-DRIVEN POLLING"]
        Q1["client asks once"] --> Q2["server waits"] --> Q3["server answers when ready"]
    end

    CL -->|"the same loop,<br/>a different owner"| SV

    style CL fill:#ffcdd2,stroke:#c62828
    style SV fill:#e3f2fd,stroke:#1976d2
```

**One refinement the prediction missed.** It's slightly *worse* than [Lecture 8](lecture-08-push.md)'s push, not merely equal:

| | Push connection | Long-poll connection |
|---|---|---|
| Who closes it | The **client** can always hang up | The **server** must decide |
| Duration | Client-controlled | **Open-ended — the server can't bound it** |

> **A long poll gives the client no way to cap the server's obligation.** The server holds N sockets of unknown remaining duration, and it — not the client — owns their lifetime.

### 3.3 — "We Can Still Disconnect" Is Not a Pro

Hussein lists this twice, and it deserves a hard test:

> *"But it says we can disconnect… That's basically it, right? So that's the beauty. We can still disconnect."*

#### First, the charitable reading — he's right about *one* comparison

Against the **one-big-request** model of [Lecture 10](lecture-10-polling.md) Unit 1, he's correct. There the client was locked in for 4 minutes and *could not* leave without losing the result. Long polling **preserves the freedom to leave** that the original model destroyed.

So the claim is valid **against long-held requests**.

#### But the comparison is against short polling — and there it fails

```mermaid
flowchart TB
    subgraph SP["SHORT POLLING - disconnect is free"]
        A1["job finishes,<br/>result STORED on the server"] --> A2["client's connection dies"]
        A2 --> A3["client reconnects<br/>and asks again"]
        A3 --> A4["result is STILL THERE.<br/>State survives the disconnect."]
    end

    subgraph LP["LONG POLLING - disconnect can destroy it"]
        B1["job finishes,<br/>server WRITES to the one socket"] --> B2["client's connection dies<br/>before the bytes land"]
        B2 --> B3["result was a DELIVERY.<br/>Written, never read. GONE."]
        B3 --> B4["client cannot tell whether<br/>it missed something or<br/>nothing happened"]
    end

    style A4 fill:#c8e6c9,stroke:#388e3c
    style B3 fill:#c62828,stroke:#c62828,color:#ffffff
    style B4 fill:#fff9c4,stroke:#fbc02d
```

#### Why: long polling converts state back into delivery

Recall [Lecture 10](lecture-10-polling.md) Unit 3.3 — the central distinction of that lecture:

| | Vanilla request/response | Short polling | **Long polling** |
|---|---|---|---|
| Result is | Delivery | **State** | **Delivery** |
| Readable twice? | ❌ | ✅ | ❌ |
| Disconnect consequence | **Lost** | **Harmless** | **Lost** |

> **Long polling is a partial regression toward the fragile delivery model.** It keeps short polling's efficiency but re-adopts the exact fragility polling was invented to remove.

#### And the recovery is the sting

What does a long-polling client do after a disconnect? It **re-polls.** Which is… short polling.

> **Long polling's failure mode degrades it into short polling — and you only discover it by hitting the failure.**

Worse, the failure is **silent**. With short polling, asking again always gives a definitive answer. With long polling, a disconnected client knows only that it lost its connection — not whether anything happened in the gap.

#### The verdict

> **Long polling without a shared durable store is strictly worse than short polling.** You pay the connection cost, you keep the delivery fragility, and you get *less* reliability.

### 3.4 — The Timeout Is Mandatory — and It Resurrects the Arithmetic

Hussein is right that timeouts exist, and doesn't follow through:

> *"You cannot pull forever… there is a timeout, the client timeout, there is a server timeout and stuff like that so that you don't wait forever."*

**Mandatory, and not optional polish.** Reasons beyond politeness: your load balancer will kill the connection anyway (typically 30–60s), dead clients can't be reliably detected otherwise, and a restart must reclaim everything.

#### This answers ❓ question 4 from Unit 2

Unit 2 said request count = **1**. That was conditional. The honest formula:

$$\text{requests} = \left\lceil \frac{\text{duration}}{\text{server timeout}} \right\rceil$$

| | Short polling (5s) | Long polling (30s timeout) |
|---|---|---|
| 4-minute job | 48 requests | **8 requests** |
| 6-hour job | 4,320 requests | **720 requests** |
| 3-day job | 518,400 requests | **8,640 requests** |

Still vastly better — but **not 1**, and **Unit 2's "the formula is deleted" was overstated.** It's reduced by a factor of `interval ÷ timeout`.

#### And the timeout reintroduces what Unit 2 deleted

```mermaid
flowchart LR
    A["4 minute job,<br/>30 second server timeout"] --> B["client must re-poll<br/>8 times"]
    B --> C["the timeout, not the client,<br/>is now the polling interval"]
    C --> D["the knob moved from<br/>client to server"]
    D --> E["but the EMPTY RESPONSE<br/>is back - at the rate<br/>of duration / timeout"]

    style B fill:#fff9c4,stroke:#fbc02d
    style E fill:#ffcdd2,stroke:#c62828
```

> **Long polling doesn't eliminate the empty response. It raises its interval from client-chosen to server-chosen.**

That's the honest version of Unit 2's claim, and it's still a big win. But "eliminated" was too strong, and ❓ question 4 is **closed with this formula**.

### 3.5 — It Is Still Not Real Time

Hussein's second con, and he's more precise here than most treatments:

> *"It's not really real time… if we got a message between the time I responded and then I got a message, the client has to still make a request, a pull request to check if there are messages in this period."*

The gap: after each response, the client must **issue the next request**. Events arriving in that gap are missed *by that long poll*.

```mermaid
flowchart LR
    A["long poll 1<br/>returns ANSWER at t=0"] -->|"client must<br/>now send a new request"| B["GAP<br/>an event arriving here<br/>is missed by poll 1"]
    B --> C["long poll 2<br/>catches it, but only<br/>because it was issued<br/>after the gap"]

    style B fill:#ffcdd2,stroke:#c62828
```

> **Long polling converts a stream into a sequence of snapshots.** Each long poll delivers one answer at one instant. Between them there is silence, and the gap length is the client's round-trip time.

#### This corrects our own Unit 2.3 claim

Unit 2 wrote that long polling makes you *"only ever as late as the network."* That's right **per event** and wrong **between events**. Real staleness is:

$$\text{staleness} = \text{re-poll gap} \approx \text{client RTT}$$

Local: milliseconds. Mobile radio: hundreds of milliseconds to seconds. Not zero.

**And Unit 2.6's own ❓ question 6 gets its answer here:** long polling delivers the *final result* beautifully and **intermediate progress not at all**. One long poll, one answer. A progress bar is fundamentally incompatible with it — which is precisely why [Lecture 10](lecture-10-polling.md)'s demo's progress reporting can't survive long polling.

### 3.6 — The Verdict

| | Short polling | Long polling | Winner |
|---|---|---|---|
| Empty responses | 47 per job | 0, or 1 per timeout | **Long** |
| Requests | 48 | 8 | **Long** |
| Detection latency | up to the interval | ~0 per event | **Long** |
| Requests scale with duration | Yes, badly | Yes, mildly | **Long** |
| **Connections held** | Brief | **Whole duration** | Short |
| **Per-waiter memory + fd** | Transient | **Persistent** | Short |
| **Disconnect** | Harmless | **Can destroy the result** | Short |
| **Progress reporting** | ✅ Possible | ❌ Impossible | Short |
| **Server owns connection lifetime** | No | **Yes, unbounded** | Short |

> **Long polling wins on every efficiency measure and loses on every reliability measure.** And the reliability losses are the ones that page you at 3am.

#### The condition under which it's actually safe

Pulling 3.3 and 3.6 together gives one rule:

> **Long polling is safe only when the server stores the result durably — so a reconnecting client can read state instead of relying on a single delivery.**

Now look at what that means for Kafka — and it closes [Lecture 10](lecture-10-polling.md) Unit 7.3 from a completely new angle:

```mermaid
flowchart TB
    A["long polling ALONE"] --> A1["efficiency, but<br/>the delivery<br/>fragility returns"]
    A1 --> A2["Net: WORSE than<br/>short polling"]

    B["long polling +<br/>a retained log"] --> B1["client re-polls after<br/>a disconnect and reads<br/>from its last offset"]
    B1 --> B2["disconnect becomes<br/>harmless, because<br/>state survives"]
    B2 --> B3["Net: SAFE"]

    style A2 fill:#c62828,stroke:#c62828,color:#ffffff
    style B3 fill:#388e3c,stroke:#388e3c,color:#ffffff
```

**So Kafka didn't choose long polling because it's efficient. It chose it because it had a durable log to make it safe.** The log isn't a separate feature bolted on — it is the thing that makes long polling viable.

That is why [Lecture 10](lecture-10-polling.md) Unit 7.3 said the offset log is what makes Kafka Kafka. Unit 3 shows the causal chain: *no log → no safe long polling → don't use it.*

---

## Vocabulary

| Term | Meaning |
|---|---|
| **Long polling** | A poll where the server withholds its response until it has something to say |
| **Pending request** | A request the server has accepted but not yet answered |
| **Silence** | The deliberate absence of a response while the server waits |
| **Held connection** | A connection kept open by an unanswered request |
| **Long poll timeout** | The limit after which the server gives up waiting and returns empty anyway |
| **Client-driven** | The client initiates every exchange; the server never speaks unasked |
| **Backpressure** | Letting the consumer set the pace so it isn't flooded — the reason Kafka chose this |
| **`maxWaitMillis`** | Kafka's long-poll ceiling on a `Fetch` request; 500ms by default |
| **Empty response** | A poll's answer of "not ready" — deleted entirely by long polling |
| **Waste-to-signal ratio** | How many empty responses per meaningful one; 47:1 becomes 0:1 |
| **Detection latency** | Time between the work finishing and the client learning of it |
| **Connection budget** | How many open connections a server can hold — the currency long polling spends |
| **Awaiting vs. blocking** | Whether a held request costs a thread or just a connection |
| **Held request** | A long poll occupying a connection while the server waits |
| **Connection budget** | The number of open connections a server can hold — long polling's currency |
| **`ulimit -n`** | The file-descriptor ceiling; the hard wall on concurrent long polls |
| **Socket buffer** | Per-connection memory held by the OS/runtime; scales with waiter count |
| **Re-poll gap** | The interval between a long poll's response and the client's next request |
| **Sequence of snapshots** | Long polling's real shape — not a stream, one answer per request |

---

## Checkpoints

### Unit 1

1. In one sentence, what is long polling?
2. What's identical between a short poll and a long poll? What exactly differs?
3. **Is an unanswered request a failed request?** What's the difference in what the server owes the client?
4. Long polling vs. push: who decides *when* the exchange happens? Why does that make it polling and not push?
5. While a long poll is silent, is the server blocked? What is it actually doing?
6. If 500 clients are sitting on silent long polls, what does the server owe them — threads, or something cheaper? Name the thing.

### Unit 2

1. What single thing does long polling delete? Give the waste-to-signal ratio before and after.
2. Same 4-minute job, 5-second interval: requests, and roughly bytes, for each design?
3. **The bigger win.** With a 5-second interval, what's the average detection latency for short polling? For long polling? Why is the second one a *categorical* improvement?
4. Request count for a 6-hour job: short polling vs long polling? Which lecture's finding does that kill?
5. Is a long poll *blocked* or *awaiting*? Which unit decides that, and what does it depend on?
6. **Name three things long polling makes worse.** Which one worries you most?
7. A 3-day job on short polling: how many requests? Now — what's the honest problem with the long-polling answer, given your answer to #4?
8. Short polling's cost scales with one thing; long polling's with another. Name both denominators.
9. Per held long poll, name three consumed resources. Which one is a **hard limit** you cannot buy your way out of?
10. **Steelman and then refute "we can still disconnect."** Against which design is he right? Against which is he wrong?
11. Run the disconnect test: job finishes at t=237, the client's connection drops at t=237.02. What was written, what was read, and what does the client know?
12. In [Lecture 10](lecture-10-polling.md)'s delivery-vs-state table, where does long polling sit — and why is that a regression?
13. What does a long-polling client do after a disconnect? Why is that the sting?
14. **Give the real request-count formula** for a long-polling job with a server timeout. Plug in 4 minutes and a 30-second timeout.
15. What does the timeout *reintroduce* that Unit 2 claimed to delete?
16. Per Hussein, why is long polling "not really real time"? **Does that correct our Unit 2.3 claim?** Say how.
17. What does the progress-bar problem have to do with this?
18. **The rule:** under what single condition is long polling safe?
19. Using Unit 3.6's causal chain — why did Kafka choose long polling? What would go wrong if it had no log?

---

## Open Questions

Logged as we go. ❓ = unverified, first pass.

### Unit 2

| # | Question |
|---|---|
| 4 | ✅ **ANSWERED in Unit 3.4 — and we were partly wrong.** With a mandatory server timeout the count is `⌈duration / timeout⌉`, not 1. The structure survives (the server's timeout replaces the client's interval) but "the formula is deleted" was overstated. Corrected in place, not quietly patched | Closed |
| 5 | ❓ Unit 2.5 claims long polling is cheap in Node.js because of the event loop. But each held request still consumes a socket, a buffer, and an entry in the event loop's bookkeeping. **At what connection count does Node.js also start to suffer?** Is "cheap" merely "cheap until it isn't"? |
| 6 | ✅ **ANSWERED in Unit 3.5** — long polling delivers one answer per request, so **progress reporting is impossible** by construction. That's why Lecture 10's demo can't survive it | Closed |

### Unit 3

| # | Question |
|---|---|
| 7 | ❓ Unit 3.3 concludes long polling without a durable store is *worse than short polling*. Then what is long polling **for**, in a system that has a shared store? Is its advantage only latency (Unit 2.3)? |
| 8 | ❓ If the result must be stored durably anyway, isn't the honest design just **SSE** — one persistent connection, every update, no re-poll gap? What does long polling uniquely offer that SSE doesn't? Unit 7 must answer this. |
| 9 | ❓ The 160–640 MB figure in 3.1 is an estimate from assumed buffer sizes. What actually dominates per-connection memory in Node.js, and does it change the ceiling materially? |
| 10 | ❓ Hussein never mentions proxies, but load balancers and CDNs terminate idle connections aggressively. Does that make the *effective* server timeout much shorter than configured — and would it invalidate the request-count formula? |

### Unit 1

| # | Question |
|---|---|
| 1 | ❓ The server holds an obligation to answer eventually. **What happens if the client disconnects during the silence?** Is the result then undeliverable — Lecture 10 Unit 3.3's lost-response problem returning? |
| 2 | ❓ Long polling holds the waiting state *in the request itself*. Is that the least durable place possible? Does it inherit Lecture 10 Unit 6's shared-store problem wholesale? |
| 3 | ❓ A timeout must eventually fire, or the request hangs forever. What does the server return when it does — an empty `200`, or `408`? Does the client immediately re-poll, and does that create a tight loop of long polls with zero idle time? |

---

*Units 1–3 of 7 studied together. One of our own claims corrected in Unit 3.4. Units 4–7 awaiting delivery.*