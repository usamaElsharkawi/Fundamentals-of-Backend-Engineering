# Fundamentals of Backend Engineering

**Course by Hussein Nasser** | Udemy

## Course Overview

This repository documents the journey through the "Fundamentals of Backend Engineering" course — not just the what, but the why behind every concept.

## Section Progress

### Section 1: (Not started)
### Section 2: Backend Communication Design Patterns
- **Progress:** 4 / 11 lectures complete | 3hr 54min total
- **Completed:** Lecture 6 — Intro ✅, Lecture 7 — Request Response ✅, Lecture 8 — Push ✅, Lecture 10 — Polling ✅ (7 units)
- **In progress:**
  - **Lecture 9 — Synchronous vs Asynchronous Workloads** ⏸️ paused
    - Units 1–7 studied ✅ | Units 8–10 explained only 📋 (revisit pending — 12 open questions logged)
  - **Lecture 11 — Long Polling** 🔄 current — Units 1–4 of 7 studied (mechanism, payoff, cost, backpressure)
- **Remaining:** 7 lectures
- **Lectures:**
  1. ✅ [6. Backend Communication Design Patterns Intro](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-06-intro.md) (2min)
  2. ✅ [7. Request Response](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-07-request-response.md) (28min)
  3. ✅ [8. Push](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-08-push.md) (20min)
  4. [9. Synchronous vs Asynchronous workloads](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-09-sync-vs-async.md) (43min)
  5. ✅ [10. Polling](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-10-polling.md) (14min)
  6. 🔄 [11. Long Polling](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-11-long-polling.md) (10min) — Units 1–4 of 7 studied
  7. [12. Server Sent Events](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-12-sse.md) (13min)
  8. [13. Publish Subscribe (Pub/Sub)](Section%202%3A%20Backend%20Communication%20Design%20Patterns/lecture-13-pubsub.md) (17min)
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