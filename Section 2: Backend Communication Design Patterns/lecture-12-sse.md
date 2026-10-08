# Lecture 12: Server-Sent Events — Built Up, Unit by Unit

## Status: In Progress 🔄 (No units studied yet)

> ### 📋 Read this before continuing
>
> | Units | State | What that means |
> |---|---|---|
> | **1–7** | ⬜ **Not yet delivered** | Outline only. Nothing here is written from discussion. |

> **How to read this doc:** Each unit builds on the previous. Each unit ends with a **Checkpoint** — answer it in your own words before moving on.

---

## Lecture Map

| Unit | Content | State |
|---|---|---|
| 1 | **The trick** — one request, an unending response | ⬜ |
| 2 | How it works — `text/event-stream`, `data:` format, `EventSource` | ⬜ |
| 3 | The problem it solves — real-time without WebSocket | ⬜ |
| 4 | The cost — connection limits, backpressure, no resume built-in | ⬜ |
| 5 | The HTTP/1.1 ceiling — 6 connections per domain | ⬜ |
| 6 | The HTTP/2 fix — multiplexing saves it | ⬜ |
| 7 | Recap — where SSE sits on the three axes | ⬜ |

### Claims flagged for testing

| # | Claim from the transcript | Why it needs testing |
|---|---|---|
| 1 | *"I think this is elegant design... whoever designed SSE is brilliant"* | Elegant for the server. What about the connection limit? The missing resume? |
| 2 | *"It works with vanilla HTTP... you don't even need to be a WebSocket server"* | True for HTTP/1.1 **until** you hit the 6-connection limit. Then it breaks the page. |
| 3 | *"The browser does the parsing for us... EventSource ships with every browser"* | True, but **no automatic reconnect with `Last-Event-ID`** in all browsers, and no binary support. |
| 4 | *"The cause is that client must be online... same problem as Push"* | Long polling solved this with a durable store. Does SSE? |

### Prediction from Lecture 11 to verify

[Lecture 11](lecture-11-long-polling.md) Unit 7.3 placed SSE on the three-axis model:

| | Axis 1: initiator | Axis 2: connection | Axis 3: failure mode |
|---|---|---|---|
| **SSE** | **Server** | **Always** | **Needs protocol** |

**If that holds, Unit 7 will confirm it.** Unit 4 presses the "needs protocol" claim — SSE has no built-in resume, heartbeat, or ordering guarantees.

---

*No units studied yet. Awaiting transcript discussion.*