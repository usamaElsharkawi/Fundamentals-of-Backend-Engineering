# Nested: How Can HTTP/1.1 Be "Text" While Riding on TCP's Stream?

> Nested question raised while discussing [Lecture 18, Unit 4](../lecture-18-protocol-properties.md): *"In the summary table you said HTTP/1.1 is Text but it is also built on top of TCP."*

## Direct Answer

**Properties belong to specific protocols at specific layers — not to the connection as a whole.** There's no contradiction because HTTP and TCP answer *different questions* at *different layers*.

| | Protocol | Layer | Property choice |
|---|---|---|---|
| Application | **HTTP/1.1** | Layer 7 | Data format: **text** |
| Transport | **TCP** | Layer 4 | Transfer mode: **stream** |

- TCP answers *"how do bytes travel?"* → as a continuous stream with no start or end.
- HTTP answers *"what do the messages look like?"* → as readable text.

HTTP's text messages live **inside** TCP's stream. The stream is the delivery mechanism; the text is the content.

## The envelope analogy

TCP is the postal system — it delivers a continuous flow of mail. HTTP is the language of the letters inside, and HTTP chose plain readable English over coded shorthand. The postal system doesn't care what language the letter is written in; sender and receiver agree on English.

## Everything is bytes on the wire

"Text" vs "binary" is **not** "bytes vs not-bytes." Every format is bytes. The distinction is: **are those bytes structured to be human-readable or not?**

```
HTTP/1.1 message (text — human-readable):
   G E T   /   H T T P / 1 . 1 \r \n ...

HTTP/2 message (binary — not human-readable):
   0x00 0x00 0x1c 0x01 0x05 0x00 0x0f 0x89 ...

Both travel through the same TCP byte-stream — the stream doesn't care.
```

## This mirrors the Section 2 insight

This is the same pattern from Lecture 15 (stateful vs stateless): HTTP is **stateless** while TCP below it is **stateful**. Different protocols freely compose — "you can build a stateless protocol on top of a stateful one and vice versa." Same principle here: text-over-stream, binary-over-stream, stateful-over-stateless (QUIC), message-over-datagram (DNS on UDP), and so on.
