# Lecture 11: Long Polling — Built Up, Unit by Unit

## Status: Complete ✅ (Units 1–7 studied)

> ### 📋 Read this before continuing
>
> | Units | State | What that means |
> |---|---|---|
| **1–7** | ✅ **Studied** | Worked through together. Concepts questioned and confirmed. |
>
> **Every claim in this lecture was tested rather than accepted.** Three transcript claims corrected, six of our own claims corrected, one transcript artifact identified. See *Unit 7.6*.
>
> **Unit 7 answers ❓ question 21** — the question Unit 6.6 created: if a readiness signal makes long polling a single request, what does it still buy over SSE? The answer is operational, and it's why long polling outlives fancier designs.
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
| **4** | **Why Kafka chose it** — backpressure, not the trick | ✅ Studied |
| **5** | **The remaining gap** — a signal, not data; and where SSE takes over | ✅ Studied |
| **6** | **The demo** — the event-loop trap, and a timeout that makes it worse | ✅ Studied |
| **7** | **Recap** — the three axes, and what long polling buys over SSE | ✅ Studied |

### Claims flagged for testing

| # | Claim from the transcript | Why it needs testing | Status |
|---|---|---|---|
| 1 | ✅ **ANSWERED in Unit 3.3** — true against a long-held request, **false against short polling**. Long polling turns Unit 10's *state* back into a *delivery*, so a disconnect can destroy the result | Closed |
| 2 | ✅ **ANSWERED in Unit 4** — neither efficiency nor trickery. **Backpressure**: push makes a slow consumer the broker's emergency; pull makes it a non-event. The log then makes long polling *safe* | Closed |
| 3 | ✅ **ANSWERED in Units 6.2–6.3** — not a slip. A busy wait starves the very event loop that would advance the job, so the `while` loop **deadlocks**. And `await` inside the loop is short polling in costume | Closed |

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

## Unit 4 — Why Kafka Actually Chose It

Unit 3 closed the transcript's "long polling is used by Kafka" claim by saying it was *beside the point*. This unit gives the real reason — and it turns out to be the better answer.

### 4.1 — The Reason He Gives

> *"They did not like the push model because… you have a consumer and you connect it to a topic, and every time the topic has data, it pushes the data to the clients' throats, literally. The consumer sometimes cannot handle the volume of messages."*
>
> *"Hey — let the client pull at their leisure."*

That is the whole idea in six words. But *why* can't the consumer handle the volume? Because **nobody consulted it about the rate.**

### 4.2 — Backpressure: Who Sets the Rate?

The volume problem isn't a Kafka detail. It's the fundamental difference between push and pull at the data layer:

```mermaid
flowchart TB
    subgraph PUSH["PUSH - producer sets the rate"]
        P1["producer: 10,000 msg/sec"] --> P2["consumer:<br/>can only handle 100"] --> P3["900 msg/sec<br/>must be buffered SOMEWHERE"]
        P3 --> P4["broker memory grows<br/>until it OOMs, or<br/>messages get dropped"]
    end

    subgraph PULL["LONG POLLING - consumer sets the rate"]
        Q1["producer: 10,000 msg/sec"] --> Q2["appended to the LOG"] --> Q3["consumer pulls at<br/>100/sec, its own pace"]
        Q2 --> Q4["log simply grows.<br/>Bounded by retention.<br/>No pressure on anyone."]
    end

    style P4 fill:#c62828,stroke:#c62828,color:#ffffff
    style Q4 fill:#388e3c,stroke:#388e3c,color:#ffffff
```

| | Push | Pull |
|---|---|---|
| Rate is set by | **Producer** | **Consumer** |
| Consumer is slow → | Broker buffers, then OOMs or drops | **Nothing. It waits.** |
| Who is the throttle? | The producer — unintentionally | **The consumer — deliberately** |
| Failure mode is visible as | Memory pressure, dropped messages | Log length |

> **Backpressure is a flow-control mechanism where the consumer's processing speed limits the producer's sending speed.** Pull doesn't *manage* backpressure — it makes violating it structurally impossible.

**The elegant consequence:** under push, a slow consumer is the broker's emergency. Under pull, a slow consumer is a non-event.

### 4.3 — Why This Is Worse for Kafka Than for Ordinary Push

[Lecture 8](lecture-08-push.md) taught that push's problems are your problems. Kafka makes three of them fatal:

**1. Volume is unbounded and unpredictable.** A topic can receive a traffic spike no consumer can absorb. Push requires the broker to buffer the difference — and its memory is finite.

**2. Push makes the broker stateful.** To push, the broker must track every consumer's position, decide routing, and handle in-flight messages per consumer. **Pull makes the consumer stateful instead** — the consumer holds an offset. Kafka moved that bookkeeping off the broker deliberately.

**3. Redelivery becomes trivial under pull.** Push: a consumer dies holding 500 in-flight messages — what happens to them? A distributed problem. Pull: restart from the **last committed offset** and re-read. At-least-once, trivially.

**4. Replay is free.** A new consumer joins, or you want last Tuesday's data for debugging — reset the offset and read again. Push cannot replay anything it already delivered.

> **Push is fine when consumer speed is predictable and roughly matched to producer speed. Kafka's whole premise is that it isn't.**

### 4.4 — The Log Repairs Unit 3's Regression

Now the payoff. Unit 3.3 concluded:

> *Long polling converts state back into delivery — so a disconnect can destroy the result.*

True in isolation. But Unit 3 also flagged ❓ question 8:

> *If the result must be stored durably anyway, isn't the honest design just SSE?*

Unit 4 answers it. Look at what the log does to the exact failure Unit 3.3 found:

```mermaid
flowchart LR
    A["UNIT 3's finding:<br/>long polling turns<br/>state into DELIVERY"] --> B["the LOG turns<br/>it back into STATE"]
    B --> C["disconnect?<br/>re-read from<br/>last offset"]
    B --> D["slow consumer?<br/>catch up when able"]
    B --> E["missed three events?<br/>all three are<br/>still in the log"]

    style A fill:#ffcdd2,stroke:#c62828
    style B fill:#c8e6c9,stroke:#388e3c
```

| Unit 3.3's failure | With a log |
|---|---|
| Disconnect after response written | **Nothing lost** — re-read from last offset |
| Client can't tell what it missed | **It knows exactly** — offsets are sequential |
| Slow client forces a decision | **It just takes longer** to catch up |

**So long polling alone is unsafe, and the log is precisely what makes it safe.** The two aren't adjacent features — the log is the other half of the design.

### 4.5 — The Log vs. a Queue

This is where the parked [broker doc](message-brokers-rabbitmq-vs-kafka.md) earns its place. The difference is small in code and enormous in consequences:

```mermaid
flowchart TB
    subgraph Q["QUEUE - RabbitMQ"]
        A1["consumer reads msg 5"] --> A2["msg 5 is REMOVED"]
        A2 --> A3["a new consumer<br/>cannot see it.<br/>replay: IMPOSSIBLE"]
    end

    subgraph L["LOG - Kafka"]
        B1["consumer reads offset 5"] --> B2["msg 5 is RETAINED"]
        B2 --> B3["a new consumer<br/>reads from 0.<br/>replay: RESET THE OFFSET"]
    end

    style A3 fill:#ffcdd2,stroke:#c62828
    style B3 fill:#c8e6c9,stroke:#388e3c
```

| | Queue | Log |
|---|---|---|
| After a consumer reads | **Deleted** | **Retained** |
| Two independent consumers | Each gets a copy, split the work | **Each reads everything** |
| New consumer joins | Seeks only what's left | **Reads from the beginning** |
| Replay for debugging | ❌ Impossible | ✅ One config change |
| Slow consumer | Messages pile up at the broker | **Log grows** — bounded by retention |
| **Who owns position** | The **broker** | **The consumer** |

> **A queue delivers each message once. A log *keeps* it and lets each reader decide how far to read.** Kafka chose "keeps," and nearly every capability above follows from that one decision.

**And the slow-consumer row explains Unit 3.1's apparently damning cost.** A waiting client's memory grows — so you *worry* about 10,000 held connections. But Kafka's held connection is cheap: an open TCP socket waiting on a read. The load is on the *log*, which is disk, retention-bounded, and cheap. **Kafka moved the pressure off RAM and onto disk.** That's a deliberate trade, not an accident.

### 4.6 — Two Corrections to Our Own Framing

**1. [Lecture 10](lecture-10-polling.md) Unit 7.3 was wrong in emphasis.** We wrote:

> *"Long polling gives you efficiency. Kafka's log gives you replay. **Different problems.**"*

Wrong. They are **one design's two halves**. Long polling without the log is unsafe (Unit 3.3); the log without long polling gives no live notification at all. They're not separable concerns — they're a mechanism and its safety net.

**2. Unit 3 was analysing a component in isolation.** We said "long polling without a durable store is strictly worse than short polling." True *as a standalone design* — but it implicitly assumed long polling was meant to be standalone. It isn't. **Long polling is a wire protocol, not a system design.**

The distinction matters: Unit 3's verdict was correct about the mechanism and wrong about its completeness.

### 4.7 — The Spine of This Unit

> **Pull costs the producer nothing, so the consumer is free to be as slow as it needs to be.** Everything durable in Kafka — replay, at-least-once, independent consumers, debugging — falls out of that one choice.

And the offset deserves its moment, because [Lecture 10](lecture-10-polling.md) Unit 7.7 predicted it:

> **"Kafka offsets — a resumable handle into a log."**

That's exactly what it is, and it's the same primitive as Unit 1's job ID, one level up:

| | Handle | Points into | Resumable? |
|---|---|---|---|
| Job ID (L10 Unit 1) | a record | A store | No — binary done/not-done |
| **Kafka offset** | a position | **An ordered log** | **Yes — read forward from any point** |

Unit 1's handle said *"this thing exists."* Kafka's handle says *"I've read up to here."* The second is strictly more powerful, and it's why Unit 7.7 called the handle the idea worth carrying to every lecture.

---

## Unit 5 — The Remaining Gap

Units 2 and 3 priced long polling. Unit 4 explained why anyone uses it. This unit closes the last conceptual hole: **what long polling fundamentally cannot do**, and where SSE takes over.

[Unit 3.5](#35--it-is-still-not-real-time) introduced the re-poll gap. This unit goes further — into a limitation that gap only describes the symptoms of.

### 5.1 — The Burst Problem: A Signal, Not Data

Unit 3.5 said long polling is "a sequence of snapshots." Here's the sharper consequence, and it's the real ceiling on the mechanism:

```mermaid
flowchart TB
    E1["event 1"] --> E2["event 2"] --> E3["event 3"] --> E4["event 4"] --> E5["event 5<br/>all five arrive<br/>during ONE long poll"]
    E5 --> R["the poll returns<br/>ONE answer"]
    R --> Q["client learns:<br/>SOMETHING HAPPENED"]
    Q -.->|"but not how many,<br/>which ones, or whether<br/>more are still coming"| H["the other four wait<br/>for the NEXT poll"]

    style Q fill:#fff9c4,stroke:#fbc02d
    style H fill:#ffcdd2,stroke:#c62828
```

> **Long polling is an existential quantifier — "∃ something new?" — not a universal one — "here is everything new."** You get a *signal*, not *data*.

This is not a fixable inefficiency. It's the shape of the mechanism. One request, one answer, **by construction.**

| The client can learn | The client cannot learn |
|---|---|
| That *at least one* event occurred | How many |
| One event's contents | Whether four more are queued |
| That the wait ended | When the *next* event will arrive |

**Progress reporting died here** — ❓ question 6, answered in Unit 3.5. This is the same wall from the other side.

### 5.2 — How Kafka Works Around It

Kafka's `Fetch` doesn't return a single record. It returns **a batch**, and it adjusts its own waiting based on whether that batch came back full:

```mermaid
flowchart TB
    A["consumer has nothing<br/>to read"] --> B["LONG POLL<br/>wait for the first record"]
    B --> C["records arrive"]
    C --> D{"was the batch<br/>FULL or short?"}
    D -->|"full -<br/>backlog exists"| E["fetch AGAIN immediately,<br/>no waiting.<br/>Drain fast."]
    E --> D
    D -->|"short -<br/>caught up"| F["long poll again<br/>and wait"]

    style E fill:#c8e6c9,stroke:#388e3c
    style F fill:#e3f2fd,stroke:#1976d2
```

**So a Kafka consumer is doing both.** It long-polls when idle, and short-polls hard when there's a backlog.

That corrects a simplification in Units 2 and 3. We treated long polling as "wait for one thing." Kafka treats it as:

> **Wait for *something*, then take *everything*, then decide whether to wait again.**

And that's why Kafka still meets a burst of 5,000 messages well: the first fetch returns a full batch, the loop keeps firing without waiting, and only when a batch comes back *short* has it genuinely caught up.

**The re-poll gap is therefore not a constant — it's conditional.** Under backlog, it's zero. Only when idle does the client wait.

### 5.3 — The Two Gaps, Named

Pulling it together, long polling has **two** distinct gaps, and Unit 3 blurred them:

| Gap | What it is | When it costs you |
|---|---|---|
| **The re-poll gap** | Client RTT between receiving an answer and issuing the next request | Always, right after every event |
| **The burst blindness** | Events during one poll are invisible until the next | Only when events arrive in bursts |

The re-poll gap is the honest price of client-driven design. It's small — milliseconds locally, worse on mobile — but **nonzero**, and it reappears after *every single event*. For a chat app that's a visible stutter; for a nightly report it's irrelevant.

### 5.4 — Answering ❓ Question 8: Long Polling vs. SSE

Unit 3 asked it directly:

> *If the result must be stored durably anyway, isn't the honest design just SSE?*

**No — and the reason is client-side agency.**

| | Long polling | SSE |
|---|---|---|
| Who initiates each notification? | **The client** | The server |
| Connection while idle | **None** — no request outstanding | One per client, permanently |
| Client decides *when to look* | ✅ **Yes, fully** | ❌ Server decides |
| Burst handling | Only if the response carries a batch | **Natural — any number** |
| Proxies / load balancers | ✅ **Nothing special required** | ⚠️ Buffering off, long timeouts, sometimes header stripping |
| Infrastructure dependency | **None** | **Significant** |
| Client complexity | A simple loop | `EventSource`, parser, reconnect, `Last-Event-ID` |
| Per-idle-client server cost | **Zero** | One connection |

**Long polling's unique offer is that the client keeps control of its own attention.** With SSE you're told. With long polling you *ask* — and you can decide that a user who hasn't looked at the tab in ten minutes shouldn't generate traffic.

```mermaid
flowchart TB
    Q{"does the client need<br/>to know the MOMENT<br/>something happens?"}
    Q -->|"yes"| S["SSE<br/>zero gap,<br/>needs infrastructure<br/>support"]
    Q -->|"no, and the client<br/>should decide when to look"| L["LONG POLLING<br/>client sets the pace,<br/>zero idle cost,<br/>no infra needs"]
    Q -->|"no, and you cannot<br/>hold a connection"| SP["SHORT POLLING<br/>simplest,<br/>stateless server"]

    style S fill:#e1bee7,stroke:#8e24aa
    style L fill:#e3f2fd,stroke:#1976d2
    style SP fill:#fff9c4,stroke:#fbc02d
```

> **SSE when immediacy matters and you control the infrastructure. Long polling when the client should set the pace, or when you can't trust the path.**

That second branch is quietly the more common one in the wild — far more deployments can hold an HTTP request than can negotiate a long-lived bidirectional stream.

### 5.5 — The Ladder Was One Axis Too Simple

[Lecture 10](lecture-10-polling.md) Unit 7.5 put all four mechanisms on a single axis — *how long is the connection held?* Unit 5 shows that's **two axes**:

```mermaid
flowchart TB
    subgraph AX1["AXIS 1 - who initiates each notification?"]
        A["client polls:<br/>SHORT POLLING,<br/>LONG POLLING"] --> B["server pushes:<br/>SSE"] --> C["broker holds it:<br/>PUB/SUB"]
    end

    subgraph AX2["AXIS 2 - how long is a connection held?"]
        D["never:<br/>SHORT POLLING"] --> E["during the wait:<br/>LONG POLLING"] --> F["always:<br/>SSE"] --> G["never:<br/>PUB/SUB"]
    end

    style A fill:#e3f2fd,stroke:#1976d2
    style B fill:#e1bee7,stroke:#8e24aa
    style C fill:#c8e6c9,stroke:#388e3c
```

| | Axis 1: initiator | Axis 2: connection |
|---|---|---|
| **Short polling** | Client | Never |
| **Long polling** | Client | During the wait |
| **SSE** | Server | Always |
| **Pub/Sub** | Broker | Never |

Note that **short polling and long polling share a cell on Axis 1** — both are client-driven. That's exactly why long polling is a *refinement* of short polling rather than a different animal: it improves Axis 2's connection cost and the empty-response rate, **not** Axis 1's client-driven nature.

> **And that's the clean answer to ❓ question 8: SSE moves you across Axis 1. Long polling only moves you along Axis 2.** They aren't competitors on one scale — they differ in kind.

**One footnote on Unit 4.4's "nothing lost":** ❓ question 13 is still open. If a consumer stays behind forever, **retention eventually evicts messages it never read** — silent data loss, which qualifies that claim. Deferred to [Lecture 13](lecture-13-pubsub.md), where retention lives.

### 5.6 — "This Might Not Be a Problem for You"

Hussein is refreshingly honest here:

> *"This might not be a problem for you. And this is where the trade-off comes in."*

**Staleness is only a bug if someone is watching.** That deserves taking seriously, not arguing past:

| The gap is **irrelevant** when | The gap is **fatal** when |
|---|---|
| A report that regenerates every few minutes | Chat / collaborative editing |
| A job the user checks hourly | Live trading or auction bidding |
| A CI/CD status page nobody watches live | "Did my payment go through?" |
| Analytics dashboards polling every 30s anyway | A stopwatch |
| Overnight batch exports | Telemetry with sub-second requirements |

> **Ask what the user is doing while they wait.** If the answer is *"nothing, and they wouldn't know if it were slow,"* long polling is complete.

The most important judgment in this lecture is knowing which column you're in — not minimizing the gap.

### 5.7 — A Correction on Offset Ownership

❓ question 11 was right to doubt Unit 4.3. The precise position:

| | Where it lives |
|---|---|
| The **decision** of what to read next | **Client-side** |
| The **record** of the last committed offset | **Broker-side** (`__consumer_offsets`) |

> **The authority is consumer-side; the storage is broker-side.** The state is *split*, not eliminated.

Unit 4.3's conclusion survives in weakened form. What pull avoids is not *state* — it's **in-flight delivery state**: which message is currently on the wire to which consumer. Push must track that per consumer; pull requires only a position. That's the real difference, and it's smaller than "stateful vs. stateless."

---

## Unit 6 — The Demo

Hussein's own verdict: *"Very simple. I wouldn't say simple, to be honest. There's more nuance here."*

He undercuts his own demo, which is to his credit. This unit answers the ❓ flagged before Unit 1 — and finds the demo is **worse than the thing it replaces.**

### 6.1 — What the Demo Does

Same job store as [Lecture 10](lecture-10-polling.md) Unit 6. One new function:

```javascript
async function checkJobComplete(jobId) {
  if (jobs[jobId] >= 100) return true;
  await new Promise(r => setTimeout(r, 1000)); // ~1s "breathing"
  return false;
}

app.get('/checkstatus', async (req, res) => {
  const done = await checkJobComplete(req.query.jobId);
  res.send(`job:${req.query.jobId} complete: ${done}`);
});
```

The run:

```
$ curl -X POST localhost:8080/submit
job:1759650000000

$ curl "localhost:8080/checkstatus?jobId=1759650000000"
 ... we didn't get a response. We're just waiting here.

# meanwhile, the server console:
50%  60%  70%  80%

# a few seconds later:
job:1759650000000 complete: true
```

> *"So every 3 seconds we get a job complete. That's how you do long polling."*

The request now blocks until the job finishes. That's the mechanism.

### 6.2 — The `setTimeout` Is Load-Bearing

Hussein explains it:

> *"It waits for 1000 milliseconds. And the reason why is because if you don't wait, then the event main loop will be blocked… adding that extra second breathing gave it some time to breathe."*

**That one line is the whole lesson**, and it isn't cosmetic — it's the only thing keeping the process alive:

```mermaid
flowchart TB
    A["a SYNCHRONOUS loop<br/>spinning on job progress"] --> B["the event loop never turns"]
    B --> C["the timer that advances<br/>the job never fires"]
    C --> D["the job never completes.<br/>The loop waits forever."]
    D --> E["DEADLOCK.<br/>The loop waits for work<br/>that only it could unblock."]

    F["the await and the setTimeout<br/>give the event loop<br/>a chance to turn"] --> G["the timer fires,<br/>progress advances,<br/>the condition can<br/>become true"]

    style E fill:#c62828,stroke:#c62828,color:#ffffff
    style G fill:#388e3c,stroke:#388e3c,color:#ffffff
```

Name the cycle precisely: **the loop waits for the job; the job needs the event loop; the event loop is waiting for the loop to finish.** Each waits on the other. That's a genuine deadlock, not slowness.

### Why this matters more than it looks

> **Node.js cannot implement a long poll with a busy wait, because a busy wait starves the very loop that would do the work.**

So the `await` isn't style. It's the mechanism. Which is [Lecture 9](lecture-09-sync-vs-async.md) Unit 2.5's distinction arriving as a hard constraint:

| | Thread | This function |
|---|---|---|
| **Blocked** (busy wait) | ❌ Starved | ❌ Deadlock |
| **Awaiting** (`await` / `setTimeout`) | ✅ Freed | ✅ Paused, work proceeds |

**Long polling in Node is only possible in the awaiting state.** The "extra second of breathing" is the whole price of admission.

### 6.3 — The `while` Loop Is a Dead End in Both Directions

Hussein warns against it:

> *"You do a while loop… Just keep looping if you do this right. Which is a bad idea. The Node.js will just sit there and run… It will never finish."*

But fixing the deadlock doesn't save it, because there are only two ways to write it:

```mermaid
flowchart TB
    Q["why not just loop<br/>until the job is done?"]
    Q -->|"write it synchronously"| A["event loop starves,<br/>DEADLOCK"]
    Q -->|"await inside the loop"| B["it runs, but it is<br/>SHORT POLLING<br/>with a wasted held connection"]
    A --> C["either way,<br/>don't loop"]
    B --> C

    style A fill:#ffcdd2,stroke:#c62828
    style B fill:#ffcdd2,stroke:#c62828
    style C fill:#fff9c4,stroke:#fbc02d
```

The second road is worse because it *looks* like a fix. `while (!done) await check()` **does** work — and it's exactly short polling, except each iteration also holds a connection open while doing nothing useful.

> **A client-side `while` loop around a long poll is short polling wearing a costume.**

That's Unit 3.4's arithmetic reasserting itself: **the timeout still governs, so the client must still re-request.** Long polling isn't something the *client* loops over. It's one request that the *server* holds.

### 6.4 — The Finding: This Demo Is Worse Than Short Polling

Here's the part that isn't in the transcript, and it's the unit's real payload.

The demo's server waits ~3 seconds, then answers. Apply Unit 3.4's formula to a 50-second job — 10 × 5s progress steps, as in [Lecture 10](lecture-10-polling.md) Unit 6:

```mermaid
flowchart LR
    A["SHORT POLLING<br/>Lecture 10 demo<br/>5s client interval"] --> A2["50s job =<br/>10 requests"]
    B["LONG POLLING<br/>this demo<br/>3s server timeout"] --> B2["50s job =<br/>17 requests"]
    A2 --> C["the upgrade makes<br/>MORE requests,<br/>not fewer"]
    B2 --> C

    style A2 fill:#fff9c4,stroke:#fbc02d
    style B2 fill:#ffcdd2,stroke:#c62828
    style C fill:#c62828,stroke:#c62828,color:#ffffff
```

$$\left\lceil \frac{50}{3} \right\rceil = 17 \qquad\text{vs.}\qquad \frac{50}{5} = 10$$

> **The demo issues about 70% more requests than the short polling it was supposed to improve on** — because the *server* timeout is shorter than the *client* interval it replaced.

**And who chose it?** The server did. That's Unit 3.4's relocation in action: *"the timeout, not the client, is now the polling interval."* The mechanism doesn't choose a good interval — **you** do, and this demo picked a poor one.

#### The steelman — why a 3-second timeout is defensible

The choice deserves a fair defence:

| Benefit of a tight timeout | Why it matters |
|---|---|
| **Proxies never kill it** | 3s is far below the 30–60s of any LB or CDN |
| **Server restarts reclaim fast** | No long-lived obligations to survive |
| **Dead clients detected quickly** | No zombie connections accumulating |
| **No fd pressure** | Exactly ❓ question 1 from Unit 1 |

**That's a real trade**: robustness and cheap failure detection, paid for in request count. [Unit 2.5](#25--the-objection-and-why-the-runtime-matters) says Node.js handles long polls cheaply — so in Node specifically, a tight timeout is *defensible* and may genuinely be right.

> **So the finding isn't "3 seconds is wrong." It's "request count is now a server decision, and this demo didn't make that decision consciously."**

### 6.5 — It Inherits Lecture 10's Bugs, and Makes Them Worse

The `jobs{}` dictionary is still in memory. [Lecture 10](lecture-10-polling.md) Unit 6.4 covered this. Long polling **doesn't fix it — it degrades the failure**:

| | Short polling, unknown job | Long polling, unknown job |
|---|---|---|
| Server responds | **`200 undefined`** — *you get an error* | **Nothing** — the request hangs |
| Client learns | Immediately, wrongly | **Nothing at all** |
| Failure type | **Loud, wrong** | **Silent, hanging** |

```mermaid
flowchart TB
    A["client long-polls job<br/>that lives on server A"] --> B["request lands<br/>on server B"]
    B --> C["server B's jobs{}<br/>has no such job"]
    C --> D["short polling:<br/>200 undefined<br/>client SEES the problem"]
    C --> E["long polling:<br/>nothing written<br/>client WAITS, learning nothing"]

    D --> F["fail loudly,<br/>fixable"]
    E --> G["fail silently<br/>until the timeout"]

    style F fill:#fff9c4,stroke:#fbc02d
    style G fill:#c62828,stroke:#c62828,color:#ffffff
```

> **Long polling converts a detectable error into an undetectable hang.** A `200 undefined` is debuggable; a request that never answers is just… normal waiting.

Every remaining [Lecture 10](lecture-10-polling.md) Unit 6 bug carries over too — no TTL, no cancellation, no ownership check, no auth, `Date.now()` IDs — and the ownership and IDOR bugs are the more dangerous now, because a long poll **enables continuous surveillance** of a job ID that short polling made awkward.

### 6.6 — What the Real Implementation Needs

Hussein admits it himself:

> *"I wouldn't say simple, to be honest. There's more nuance here… because this is a contrived example, you might be lucky and you have like some sort of a readiness, some sort of a publish subscribe where you can effectively get a push notification immediately when you get that and then you respond right there."*

He is describing what the mechanism *actually* needs — a **readiness signal** that wakes the waiter the instant the job finishes. That is:

| The demo's approach | The real approach |
|---|---|
| **Sleep for 3s, then check** | **Wait on an event that fires** |
| Polls, just server-side | **Zero polls** |
| Request rate = `1 / timeout` | **Request rate = 1** |
| Detects completion within the timeout | Detects completion **immediately** |

```mermaid
flowchart LR
    A["DEMO - timeout-based"] --> A1["sleep 3s,<br/>check, answer"]
    A1 --> A2["~17 requests<br/>for a 50s job"]

    B["REAL - readiness-based"] --> B1["await a signal<br/>that the job finished"]
    B1 --> B2["1 request<br/>for a 50s job"]

    style A2 fill:#ffcdd2,stroke:#c62828
    style B2 fill:#c8e6c9,stroke:#388e3c
```

> **A readiness signal is what makes long polling a single request instead of a series.** The demo substituted a sleep for the signal — which is why it needs so many.

And what mechanism carries that signal? **Publish/Subscribe.** In-process it's an `EventEmitter`; across a fleet it's Redis pub/sub, Kafka, or any broker.

> **So Lecture 11 ends where Lecture 13 begins.** The correct long poll is pub/sub held for exactly one request's lifetime.

### 6.7 — The Verdict

| | |
|---|---|
| ✅ **Proves** | The mechanism works and is small — one `async` function, one route |
| ✅ **Proves** | The event loop is the whole difficulty; `await` isn't optional |
| ⚠️ **Hides** | That a readiness signal — not a timeout — is the real design |
| ⚠️ **Hides** | That request count is now the *server's* choice |
| 💀 **Gets wrong** | The timeout makes it ~70% *worse* than the short polling it replaces |
| 💀 **Inherits** | Every [Lecture 10](lecture-10-polling.md) Unit 6 bug, with the failure made silent |

> **The demo is the mechanism in ten lines. It is not the pattern.** The difference between them is a readiness signal — which is the subject of [Lecture 13](lecture-13-pubsub.md).

---

## Unit 7 — Recap

This unit answers ❓ #21, and the answer turns out to be the most interesting thing in the lecture.

### 7.1 — Seven Units, One Argument

```mermaid
flowchart TB
    U1["Unit 1 - THE TRICK<br/>same request,<br/>the server just doesn't reply"]
    U2["Unit 2 - THE PAYOFF<br/>the empty response<br/>is deleted"]
    U3["Unit 3 - THE BILL<br/>polling moved server-side;<br/>disconnect is not a pro"]
    U4["Unit 4 - THE REASON<br/>backpressure, and<br/>the log as safety net"]
    U5["Unit 5 - THE CEILING<br/>a signal, not data"]
    U6["Unit 6 - THE DEMO<br/>deadlock, and a timeout<br/>that makes it worse"]

    U1 --> U2 --> U3 --> U4 --> U5 --> U6

    style U1 fill:#e3f2fd,stroke:#1976d2
    style U3 fill:#ffcdd2,stroke:#c62828
    style U5 fill:#fff9c4,stroke:#fbc02d
    style U6 fill:#ffcdd2,stroke:#c62828
```

### 7.2 — The Spine

> **By making the server wait instead of answering, long polling deletes the empty response — and inherits the fragility of delivery.** It becomes **safe** only when a durable store restores state, and **efficient** only when a readiness signal ends the wait.

Three conditions, each from a different unit, none of them optional:

| | Requirement | From |
|---|---|---|
| **Works at all** | A request the server can hold | Unit 1 |
| **Worth it** | A readiness signal, not a sleep | Unit 6.6 |
| **Safe** | A durable store the client can re-read | Units 3.3, 4.4 |

### 7.3 — Answering ❓ Question 21: What Does Long Polling Buy Over SSE?

Unit 6.6 created the pressure:

> If a readiness signal makes long polling a **single** request, and SSE also delivers immediately over **one** connection — what's left?

**Architecturally, almost nothing.**

```mermaid
flowchart TB
    A["IDEAL long polling:<br/>readiness signal<br/>plus a durable store"] --> B["immediate delivery,<br/>one request per result,<br/>resumable after disconnect"]
    C["SSE"] --> B

    B --> D["OBSERVABLY EQUIVALENT.<br/>Same latency.<br/>Same one request per result.<br/>Same resumability."]
    D --> E["So they do not differ in<br/>what the client SEES.<br/>They differ in what BREAKS,<br/>what they REQUIRE,<br/>and what they COST while idle."]

    style D fill:#c8e6c9,stroke:#388e3c
    style E fill:#fff9c4,stroke:#fbc02d
```

**What remains is entirely operational.** Three real differences.

#### 1. Connection failure mode — the big one

| | Long polling | SSE |
|---|---|---|
| Connection lives | **One wait**, then closes | Indefinitely |
| When it breaks | It **dies every time it succeeds** | Can stay broken silently |
| Recovery | **Free** — the next request is fresh | **A protocol**: heartbeat, `Last-Event-ID`, reconnect |
| Detection | Immediate | TCP keepalive — minutes, or never behind a proxy |

> **A long-polling connection is self-healing because it ends on every result.** A dead SSE connection is indistinguishable from an idle one until you build machinery to tell them apart — and that machinery is most of SSE's complexity.

#### 2. Infrastructure requirements

| | Long polling | SSE |
|---|---|---|
| Needs proxy streaming support | ❌ **No** | ✅ Yes |
| Needs buffering disabled | ❌ No | ✅ Yes |
| Survives corporate proxies | ✅ **Ordinary HTTP** | ⚠️ Often needs config |
| Works over HTTP/1.1 | ✅ Yes | ✅ Yes — unusual for a stream |

> **Long polling works anywhere HTTP works.** SSE works only where you've verified streaming. That's [Lecture 10](lecture-10-polling.md) Unit 4.1's safety-net property, and it's the strongest reason long polling outlives fancier designs.

#### 3. Idle cost

| | Long polling | SSE |
|---|---|---|
| Cost for a client that isn't looking | **Zero** — no request outstanding | One connection, held |
| Scales with | **Waiting** clients | **Connected** clients |
| Many idle users | ✅ Cheap | ❌ Expensive |
| Few active users | ❌ Repeated handshakes | ✅ Cheapest |

> **Long polling charges for waiting. SSE charges for connecting.** If your users mostly *aren't* looking, long polling is cheaper. If they always are, SSE is.

#### The answer, in one line

> **Long polling and SSE converge on behavior and diverge on failure modes.** Same result delivered; different blast radius when the network, a proxy, or a client goes wrong.

### 7.4 — The Final Map: Three Axes

[Lecture 10](lecture-10-polling.md) Unit 7.5 gave one axis. Unit 5.5 corrected it to two. **Unit 7 adds the third** — the one that decides operational maturity:

| | Axis 1: who initiates | Axis 2: connection held | **Axis 3: connection failure mode** |
|---|---|---|---|
| **Short polling** | Client | Never | **Impossible** — connections are transient |
| **Long polling** | Client | During the wait | **Self-healing** — dies on every result |
| **SSE** | Server | Always | **Needs a protocol** — heartbeat + resume |
| **Pub/Sub** | Broker | Never | **Invisible** — no connection to lose |

```mermaid
flowchart TB
    subgraph AX3["AXIS 3 - what happens when the connection breaks?"]
        A["SHORT POLLING<br/>nothing to break:<br/>worst case is one lost poll"] --> B["LONG POLLING<br/>self-healing,<br/>dies on every success"]
        B --> C["SSE<br/>needs a protocol:<br/>heartbeat, resume token,<br/>Last-Event-ID"]
        C --> D["PUB/SUB<br/>no connection<br/>exists to break"]
    end

    style A fill:#e3f2fd,stroke:#1976d2
    style B fill:#c8e6c9,stroke:#388e3c
    style C fill:#e1bee7,stroke:#8e24aa
    style D fill:#f9c4d0,stroke:#c2185b
```

> **Axis 3 is the maturity ladder.** It explains a real observation: short polling needs no libraries, SSE needs `EventSource` plus resume handling, pub/sub needs a broker. **Complexity follows directly from how much can break.**

That's the honest answer to "which is better." Neither. **Each buys something by accepting something, and Axis 3 tells you what you're accepting.**

### 7.5 — When to Use It

| ✅ Long polling when | ❌ Not when |
|---|---|
| You need immediacy but **can't verify streaming support** | You control the infrastructure and want maximum efficiency |
| Clients are flaky — mobile, VPN, proxies. Re-requests are free | Connections are cheap and most clients are idle |
| You want **self-healing** connections | You can afford heartbeat + resume machinery |
| One event per result — a **signal**, not a stream | You need many events per connection → SSE |
| The client should decide **when to look** | The user must know regardless of attention |

> **Default to long polling when you can't be sure the fancy option will work. Graduate when you know it will.**

That inverts the usual framing. Most advice says "prefer SSE." The operational reality is that **SSE's requirements are invisible until the day a proxy silently buffers your stream** — and long polling has no such day.

### 7.6 — The Audit

**Six corrections to our own claims:**

| # | Unit | What we said | What we corrected |
|---|---|---|---|
| 1 | 3.4 | Request count = **1** | `⌈duration / timeout⌉` |
| 2 | 3.5 | *"only ever as late as the network"* | True per event, not between — staleness is the **re-poll gap** |
| 3 | 4.6 | Long polling and the log are *"different problems"* | **One design's two halves** |
| 4 | 5.2 | Long polling is *"wait for one thing"* | **"Wait for something, take everything, then decide"** |
| 5 | 5.7 | Pull *"makes the consumer stateful"* | State is **split** — authority client-side, storage broker-side |
| 6 | 6.4 | The demo improves on short polling | It's **~70% worse** in request count |

**Three transcript claims tested:**

| # | Claim | Verdict |
|---|---|---|
| 1 | "Clients can disconnect" as a **pro** | ⚠️ True against a long-held request, ❌ **false against short polling** |
| 2 | "Long polling is used by Kafka" | ⚠️ True, but **backpressure** is the reason, and the log makes it safe |
| 3 | *"I wouldn't say simple, to be honest"* | ✅ **Honest, and correct** — he knew the demo wasn't production code |

**And one artifact:** *"40 years ago… when I got interested in Kafka."* Kafka arrived ~2008. Long polling is old; learning it from Kafka isn't. Assume a transcript error.

> **Six corrections, three claims tested, one artifact. Zero accepted on faith.**

### 7.7 — Self-Test — Whole Lecture

1. In one sentence: what does the server do differently in long polling?
2. Is an unanswered request a failed request? What is the server's obligation?
3. What single thing does long polling delete? Give the waste-to-signal ratio before and after.
4. What's the **bigger** win — bandwidth or latency? Give the average detection latency for each.
5. Long polling's cost scales with what? Short polling's?
6. **Name the hard limit** 10,000 waiting clients will hit.
7. Run the disconnect test: job finishes at t=237, connection drops at t=237.02.
8. Give the real request-count formula. Plug in 4 minutes, 30-second timeout.
9. What does the timeout *reintroduce*?
10. Define **backpressure**. Who is the throttle under push? Under pull?
11. Why is a **queue** different from a **log**? Name three things the log gives you.
12. What must the server have for long polling to be **safe**?
13. Why does a **busy wait** deadlock the Node event loop? Name the cycle.
14. Why is `while (!done) await check()` still wrong?
15. What is a **readiness signal**, and why does the demo lack one?
16. Name the **three axes**. Which one explains why SSE needs libraries?
17. What does long polling buy over SSE — and name the three things it costs instead.
18. **One sentence: the difference between short and long polling, in terms of ownership.**

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
| **Backpressure** | Consumer processing speed limiting producer send rate |
| **Throttle** | Whoever sets the rate. Push: the producer. Pull: the consumer |
| **Offset** | A consumer's position in a log; the resumable handle |
| **Consumer group** | A set of consumers sharing one partition's read position |
| **Log vs. queue** | Queues delete on read; logs retain, and each reader picks its own position |
| **Retention** | How long a log keeps messages — the bound on growth |
| **Signal, not data** | One answer per request: you learn *that* something happened, not everything |
| **Existential quantifier** | "∃ something new?" — the logic long polling implements |
| **Burst blindness** | Events arriving mid-poll are invisible until the next request |
| **Full vs. short batch** | How Kafka's consumer knows whether a backlog remains |
| **Client-side agency** | The client choosing *when* to look — long polling's unique offer |
| **Two axes** | Who initiates (Axis 1), how long a connection is held (Axis 2) |
| **In-flight delivery state** | Which message is currently on the wire — what pull avoids tracking |
| **Deadlock** | The loop waits for the job; the job waits for the loop to yield |
| **Busy wait** | Spinning synchronously until something changes — fatal on an event loop |
| **Readiness signal** | An event that fires when the job finishes — what a real long poll waits on |
| **Timeout-based polling** | Substituting a sleep for a readiness signal; costs `duration / timeout` |
| **Silent hang** | A long poll on a missing job fails invisibly, where short polling fails loudly |
| **Surveillance** | What a held long poll enables: repeated observation of one job ID |
| **Axis 3** | What happens when the connection breaks — the maturity ladder |
| **Self-healing connection** | One that dies on every success, so failure needs no detection |
| **Failure-mode divergence** | Two mechanisms that behave identically but break differently |
| **Idempotent re-request** | The cheapest recovery in HTTP — just ask again |

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
20. Define backpressure in one sentence. Under push, who is the throttle? Under pull?
21. Producer at 10,000 msg/sec, consumer handling 100/sec. What happens under push? Under pull?
22. Why is "the log grows" a **feature** under pull and a **bug** under push?
23. Name three reasons push is wrong for Kafka specifically.
24. **How does the log repair Unit 3's regression?** Be precise: state, delivery, and what the reconnecting consumer does.
25. Queue vs. log: what happens to a message after a consumer reads it? What follows from that?
26. Whose state does pull move to the consumer, and what operational consequence does that have for a broker operator?
27. Five events arrive during one long poll. What does the client learn, and what can it not learn? Name the logical distinction.
28. Why is progress reporting structurally impossible here?
29. How does Kafka's fetch loop adapt? What does a *full* batch tell it to do, and a *short* one?
30. **Two gaps.** Name them. Which one only bites during bursts?
31. **Answer ❓ question 8.** What does long polling offer that SSE doesn't? Why isn't SSE just strictly better?
32. Walk the 5.4 decision tree for: (a) a live trading app you fully control, (b) a mobile app behind corporate proxies, (c) a nightly report.
33. **Lecture 10 Unit 7.5 used one axis. Why is two axes more accurate?**
34. Which two of the four mechanisms share a cell on Axis 1 — and what does that tell you about their relationship?
35. Give one example where the re-poll gap is irrelevant, and one where it's fatal. **What question decides which column you're in?**
36. Reconstruct `checkJobComplete` in words. Why does it need the `setTimeout`?
37. What exactly deadlocks in a synchronous `while` loop? Name the three-party cycle.
38. A synchronous loop deadlocks. An `await` inside one works. **Why is the second one still wrong?**
39. Request counts for a 50-second job: short polling at 5s vs. this demo's long poll at 3s. Which is worse, and by how much?
40. **Whose choice now determines the request rate?** What does Unit 3.4's formula say?
41. Give the steelman for a 3-second timeout. Is it defensible?
42. The demo inherits the in-memory `jobs{}` bug. How does long polling make that failure **worse** than short polling?
43. What is the *readiness signal* he admits the real implementation needs — and which mechanism carries it?
44. Which Lecture 9 unit is this demo's `await` load-bearing for?
45. In one sentence: what does this demo teach, and what does it hide?

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

### Unit 4

| # | Question |
|---|---|
| 11 | ✅ **ANSWERED in Unit 5.7** — the state is *split*, not eliminated: authority is client-side, storage is broker-side. Unit 4.3 survives only in weakened form, and the real distinction is **in-flight delivery state** | Closed |
| 12 | ❓ Consumer groups were mentioned but not explained. If N consumers share one partition, how do they avoid all reading the same offsets? What does that cost, and does it change Unit 4.2's flow-control story? |
| 13 | ❓ If a consumer is genuinely slower than real time **forever**, the log grows without bound until retention evicts messages the consumer hasn't read. **That is silent data loss.** How does Kafka handle this, and should it change Unit 4.4's "nothing lost" claim? |
| 14 | ❓ Unit 4.5 says pull "costs the producer nothing." But a fast producer still pays for **retention storage** for messages nobody reads. Is that cost invisible? |

### Unit 5

| # | Question |
|---|---|
| 15 | ❓ Unit 5.2 says the re-poll gap "is zero under backlog." True only while the backlog lasts. What happens the instant a burst ends — does the last few events get stranded behind a long poll that then waits 30s for nothing? |
| 16 | ❓ Unit 5.1 argues the burst blindness is *structural*. But Kafka's batch fetch partially defeats it. Is the ceiling "one answer per request," or "one *batch* per request"? Which phrasing is actually correct? |
| 17 | ❓ Unit 5.4 claims long polling needs "no infrastructure support" and SSE needs "significant" support. Is that fair in 2026? Modern proxies and CDNs handle SSE routinely. Has the gap closed enough to flip the default recommendation? |
| 18 | ❓ The re-poll gap is described as client RTT. But on a **battery-saving mobile browser**, timers are throttled to once a minute or worse. Does that quietly make long polling useless on mobile, independent of any server-side concern? |

### Unit 6

| # | Question |
|---|---|
| 19 | ❓ Unit 6.4 claims the demo issues ~17 requests where short polling issued 10. That assumes the client re-requests on every `false`. Does the demo actually show that, or does it use a *single* request that only ends on `true`? **Which is it?** The transcript says *"every 3 seconds we get a job complete,"* which is ambiguous. |
| 20 | ❓ Unit 6.4's steelman says a 3-second timeout is defensible because proxies won't kill it. But ❓ question 10 (Unit 3) asked whether proxy timeouts invalidate the request-count formula. Does a tight timeout **defuse** that question, or just hide it? |
| 21 | ❓ If a readiness signal makes long polling a single request, is long polling still meaningfully distinct from SSE? SSE also delivers immediately over one connection. **What does long polling's single request buy that SSE's permanent connection doesn't?** Unit 7 must answer this. |
| 22 | ❓ Unit 6.5 claims a held long poll enables "surveillance" of a job ID. But short polling allows the same reads at 5s intervals. Is long polling genuinely worse for enumeration, or just more continuous? |

### Carried forward from this lecture

| # | Question | Status |
|---|---|---|
| 23 | ❓ Unit 7.3 concludes long polling and SSE are *operationally* different but behaviourally equivalent. So does that mean SSE is simply the better long polling — worth migrating to once you can verify your infrastructure? **Or is there a case where long polling is genuinely superior long-term?** | Open — discuss |
| 24 | ❓ Axis 3 claims complexity follows from what can break. But Pub/Sub has *no* connection and yet requires a broker — arguably more infrastructure than SSE. **Does Axis 3 actually predict complexity, or does it only correlate with it?** | Open — test the claim |
| 25 | ❓ Unit 6.6 concluded the correct long poll is pub/sub held for one request's lifetime. If that's the shape, is [Lecture 13](lecture-13-pubsub.md) then redundant — just the "correct" version of this lecture? | Deferred to Lecture 13 |

### Unit 1

| # | Question |
|---|---|
| 1 | ❓ The server holds an obligation to answer eventually. **What happens if the client disconnects during the silence?** Is the result then undeliverable — Lecture 10 Unit 3.3's lost-response problem returning? |
| 2 | ❓ Long polling holds the waiting state *in the request itself*. Is that the least durable place possible? Does it inherit Lecture 10 Unit 6's shared-store problem wholesale? |
| 3 | ❓ A timeout must eventually fire, or the request hangs forever. What does the server return when it does — an empty `200`, or `408`? Does the client immediately re-poll, and does that create a tight loop of long polls with zero idle time? |

---

*All 7 units studied together. Three transcript claims corrected, six of our own claims corrected, one transcript artifact flagged.*