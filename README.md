# Fundamentals of Backend Engineering

**Course by Hussein Nasser** | Udemy

## Course Overview

This repository documents the journey through the "Fundamentals of Backend Engineering" course — not just the what, but the why behind every concept.

## Section Progress

### Section 1: (Not started)
### Section 2: Backend Communication Design Patterns
- **Progress:** 7 / 11 lectures complete | ~5hr total
- **Completed:** Lecture 6 — Intro ✅, Lecture 7 — Request Response ✅, Lecture 8 — Push ✅, Lecture 10 — Polling ✅ (7 units), Lecture 11 — Long Polling ✅ (7 units), Lecture 12 — Server Sent Events ✅ (7 units), Lecture 13 — Publish Subscribe ✅ (6 units)
- **In progress:**
  - **Lecture 9 — Synchronous vs Asynchronous Workloads** ⏸️ paused
    - Units 1–7 studied ✅ | Units 8–10 explained only 📋 (revisit pending — 12 open questions logged)
- **Next:** Lecture 14 — Multiplexing vs Demultiplexing (15min)
- **Remaining:** 4 lectures

---

## 💡 Working Preferences

Recorded from the session so they survive across sessions.

### Code alongside explanation
**Boss likes seeing the actual code next to the explanation.** When teaching a mechanism —
framing, event loops, timeouts, connection handling — show the real code snippet *in the same
response* as the explanation. The code grounds the abstraction; the explanation gives it meaning.
Don't separate them into "here's the theory" and "here's the code later."

### How to apply
- In each unit, include the relevant code block (server-side and client-side where both matter)
- Explain *why each line is there*, not just what it does
- Call out the lines that are load-bearing vs. cosmetic
- If a line is wrong or missing (like Hussein's missing `onmessage` handler), say so

### Why this works
The boss learns by connecting code to mental models. A mechanism without code is a definition;
code without explanation is a recipe. Both together build understanding.

---

## 🗂️ Next Session: Section Synthesis

**Both lectures are complete and pushed. Nothing is mid-flight.**

The next session has one job: **recap the whole section, link the lectures into one model, and work the backlog.**

### 1. Link the eight lectures into one story

| # | Lecture | The idea it contributes |
|---|---|---|
| 6 | Intro | Each pattern answers a **different problem** — categorize, don't compete |
| 7 | Request/Response | A request is **bytes**, not an object. Framing creates boundaries |
| 8 | Push | Push breaks *"the client drives"* — and inherits every consequence |
| 9 | Sync vs Async | **Blocked vs awaiting**: is the *thread* suspended, or just this function? |
| 10 | Polling | Return a **handle**, not a result. Read **state**, not delivery |
| 11 | Long Polling | Make the server **wait**. Safe only with a durable store |
| 12 | SSE | One request, an **unending response** — real-time over plain HTTP, paid for with a held connection |
| 13 | Pub/Sub | Move initiation to a **broker**. Trade held connections for held state |

### 2. The spine to build toward

> **Return a reference instead of a result** — then buy the result back as cheaply as your infrastructure allows. Every pattern in this section is a different answer to *how the client learns what happened*.

### 3. The unified model we already earned

Two corrections we made to Lecture 10's ladder, plus the third axis found in Lecture 11:

| | Axis 1: who initiates | Axis 2: connection held | Axis 3: failure mode |
|---|---|---|---|
| Short polling | Client | Never | Impossible |
| Long polling | Client | During the wait | Self-healing |
| SSE | Server | Always | Needs a protocol |
| Pub/Sub | Broker | Never | Invisible |

### 4. Work the backlog — **45 open ❓ , 10 already closed**

| Source | Open | Notes |
|---|---|---|
| [Lecture 9](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-09-sync-vs-async.md) | **12** | Units 8–10 still **explained only** — the biggest single debt |
| [Lecture 10](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-10-polling.md) | **8** | TTL, aliasing, autoscaling, the Unit 4.3/5.3 contradiction |
| [Lecture 11](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-11-long-polling.md) | **22** | Includes 3 carried-forward questions that **challenge our own conclusions** |
| [Lecture 13](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-13-pubsub.md) | **4** | Producer ack management, dead letter queues, consumer group coordination, QoS/prefetch |
| [Lecture 12](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-12-sse.md) | **3** | Backpressure for slow consumers, the demo's interval leak, HTTP/2 stream-limit defaults |

Priority order: **Lecture 9 Units 8–10** → the 3 challenges to our own claims → TTL and retention → Lecture 12's backpressure/disconnect questions → the rest.

### 5. The 3 questions that test our own reasoning

These are the most valuable — they question conclusions we asserted, not facts we absorbed:

| # | Question | Where |
|---|---|---|
| **A** | Axis 3 claims *complexity follows from what can break*. But Pub/Sub has **no connection** yet needs a broker — more infrastructure than SSE. Does Axis 3 **predict** complexity, or only **correlate**? | L11 #24 |
| **B** | The in-memory-store demo made short polling's failure **loud**. Under long polling it's **silent**. Does that change our earlier judgment that the demo was "elegant"? | L11 #22 |
| **C** | Unit 4.3 said long polling is "worse than short polling" without a durable store — then Unit 4.6 admitted it was analysing a component in isolation. **Which verdict stands?** | L11 #7, #21 |

### 6. Then continue

Lecture 14 (Multiplexing vs Demultiplexing, 15min) — combining multiple logical streams onto one connection, and splitting them apart on the other side.
- **Lectures:**
  1. ✅ [6. Backend Communication Design Patterns Intro](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-06-intro.md) (2min)
  2. ✅ [7. Request Response](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-07-request-response.md) (28min)
  3. ✅ [8. Push](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-08-push.md) (20min)
  4. [9. Synchronous vs Asynchronous workloads](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-09-sync-vs-async.md) (43min)
  5. ✅ [10. Polling](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-10-polling.md) (14min)
  6. ✅ [11. Long Polling](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-11-long-polling.md) (10min)
  7. ✅ [12. Server Sent Events](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-12-sse.md) (13min)
  8. ✅ [13. Publish Subscribe (Pub/Sub)](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-13-pubsub.md) (17min)
  9. [14. Multiplexing vs Demultiplexing](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-14-multiplexing.md) (15min)
  10. [15. Stateful vs Stateless](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-15-stateful-vs-stateless.md) (23min)
  11. [16. Sidecar Pattern](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-16-sidecar.md) (22min)

## How This Works

- Watch the lecture → tell me what you understood → I clarify, correct, and deepen
- After each lecture, say **"document it"** to generate a comprehensive knowledge base entry
- After each section, say **"we finished the lecture"** to mark it complete
- All documentation gets pushed to GitHub as our personal backend engineering knowledge base

## Core Philosophy

> Don't just finish the course. Become a better backend/full-stack engineer.

Understand the **why**, not just the **what**. Build strong mental models. Connect concepts to real-world applications.

## Key Mental Model (From Lecture 6)

**Each pattern answers a different engineering problem.** Don't compete patterns — categorize them:
- **Communication**: Request/Response, Polling, Long Polling, SSE, Push
- **Messaging/Distribution**: Pub/Sub
- **Architecture/State**: Stateful, Stateless
- **Transport/Connection**: Multiplexing
- **Deployment**: Sidecar

> **"What problem am I solving, and what trade-offs am I willing to accept?"**

## Reference Docs

Supplementary material documented alongside the lectures:

| Doc | Covers | Referenced from |
|---|---|---|
| [Message Brokers — RabbitMQ vs Kafka](Section%202%3A%20Backend%20Communication%20Design%20Patterns/message-brokers-rabbitmq-vs-kafka.md) | What a broker is, message brokers as decoupling, RabbitMQ queue model vs Kafka log model, delivery guarantees, consumer groups | Lecture 8 Unit 9, Lecture 13 (Pub/Sub) |
| [HTTP Versions Comparison](Section%202%3A%20Backend%20Communication%20Design%20Patterns/http-versions-comparison.md) | HTTP/1.1 vs HTTP/2 vs HTTP/3, framing, HPACK/QPACK, QUIC, HOL blocking | Lecture 7 (Request/Response) |