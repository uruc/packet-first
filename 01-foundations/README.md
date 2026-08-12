# Track 01 — Foundations

**What this track answers:** what a link is, why we broke the network into
pieces, and how a box decides whether a destination is reachable directly or
has to be handed off.

Five lessons. They build to one question, and Lesson 04 is that question —
everything before it exists to make the question answerable, and every later
track is elaboration on what happens when the answer comes back "not local."

| # | Lesson | Answers |
|---|---|---|
| 01 | [The link](01-the-link.md) | What is a link, and what does a switch actually do? |
| 02 | Why one link doesn't scale | What breaks when a broadcast domain gets large — and which of those breakages is the real driver? |
| 03 | What an IP address solves that a MAC can't | Why two address systems? What does hierarchy buy? |
| 04 | **The fork: local or not?** | The mask test, and why it is the only branch that matters |
| 05 | From decision to frame | ARP, next-hop resolution, and what changes at each hop |

## Why start this low

Not because subnetting needs relearning. Because the two questions that turn
out to be load-bearing later — *why does routing exist at all?* and *what
exactly changes when a packet crosses a router?* — can only be answered from
here. Everything in track 05 (the FortiOS pipeline) is a sequence of operations
on the answer to Lesson 04.

Expect to move through 01–03 quickly. Expect 04 and 05 to take real time.
