# PROGRESS

The ledger. A lesson is not done when it has been read — it is done when its
checkpoint has been answered **from memory**, and the answers are written below.

Statuses: `not started` · `reading` · `hands-on done` · **`checkpoint passed`**

Only `checkpoint passed` unlocks the next lesson.

---

## Track 00 — The tap

| Lesson | Status | Checkpoint passed |
|---|---|---|
| [00 — Placement](00-the-tap/00-placement.md) | not started | — |
| [01 — The instrument lies](00-the-tap/01-the-instrument-lies.md) | not started | — |
| [02 — Records that are not packets](00-the-tap/02-records-not-packets.md) | not started | — |

All three come before Track 01's hands-on. Track 00 has no prerequisites.
**Start here.**

## Track 01 — The link, and what it leaks

| Lesson | Status | Checkpoint passed |
|---|---|---|
| [01 — The link](01-the-link/01-the-link.md) | reading | — |
| 02 — The scope of a MAC | not started | — |
| 03 — Asking who is there: ARP and ND | not started | — |
| 04 — Being told who is there: DHCP, RA, and the v6 problem | not started | — |
| 05 — What this leaks | not started | — |

## Tracks 02–08

Not started. Lessons are written as they're reached — see
[README.md](README.md) for the map and the per-track capability claims.

---

# Checkpoint answers

Write answers here before looking back at the lesson. Wrong answers stay in the
file — the record of what was misunderstood is more useful than a clean sheet,
and it shows what to re-test in a month.

## Track 00 / Lesson 00 — Placement

> 1. Why does a single capture file contain two populations of packets with
>    different evidentiary value, and what determines which population a given
>    packet belongs to?

_(answer here)_

> 2. A packet you expected is not in the capture. You are asked to determine
>    which of the possible causes it was, using only that capture. Explain why
>    you cannot, and say what additional evidence would separate them.

_(answer here)_

> 3. Two capture points on the same path show different source addresses for the
>    same packet. Which one is wrong, and why is that the wrong question?

_(answer here)_

> 4. In the same capture file, the source address of an IPv4 packet and the
>    source address of an IPv6 packet do not answer the same question. Explain
>    the difference and what causes it.

_(answer here)_

> 5. You are handed a capture file with no note saying where it was taken. State
>    precisely what you can still conclude from it and what you cannot.

_(answer here — this one gets asked again at the end of Track 08)_

## Track 00 / Lesson 01 — The instrument lies

> 1. A capture on a 1500-byte link shows a 40 KB frame. What happened, where
>    exactly did it happen relative to your tap, and what does it tell you about
>    the wire?

_(answer here)_

> 2. Every outbound packet in your capture has a bad checksum; every inbound one
>    is fine. Give the explanation, and give the reasoning that gets you there
>    without checking any settings.

_(answer here)_

> 3. An on-box sniffer goes silent during a transfer you can prove is running.
>    What is the mechanism, and why does it produce silence rather than errors?

_(answer here)_

> 4. What can `pktmon` tell you that a NIC-level capture structurally cannot, and
>    why is it a property of *where it attaches* rather than of the tool's
>    features?

_(answer here)_

> 5. Two numbers decide whether a capture supports the claim "X never happened."
>    Name them and say what each one being wrong would hide.

_(answer here)_

## Track 00 / Lesson 02 — Records that are not packets

> 1. A three-hour connection appears as six flow records. Explain the mechanism,
>    and say what would have to be true for it to appear as three instead.

_(answer here)_

> 2. A "large upload" detection thresholds bytes per flow record and never fires
>    on a known slow exfiltration. Explain the failure without using the word
>    "threshold" — what is the detection actually measuring?

_(answer here)_

> 3. A firewall log line carries a `duration` field. What does that fact alone
>    tell you about when the record was written, and why does it follow
>    necessarily rather than by convention?

_(answer here)_

> 4. An endpoint says 14:03 and a firewall says 14:41 for the same connection.
>    Give the arithmetic that reconciles them, and name a second, independent
>    offset that could also be involved.

_(answer here)_

> 5. Rank packet capture, flow records and firewall logs by how much they
>    preserve — then give one question each of the other two can answer that the
>    richest one cannot.

_(answer here)_

**Measured clock offset** (Lesson 02, step 5): _(record it here)_

## Track 01 / Lesson 01 — The link

> 1. A switch has an empty MAC table. Traffic still flows correctly. Explain
>    why — and what is worse about it.

_(answer here)_

> 2. Why does a switch learn from source addresses rather than destination
>    addresses? What would break if it tried the reverse?

_(answer here)_

> 3. Two machines on the same switch, same broadcast domain. Every router on
>    earth is unplugged. Can they talk? Why?

_(answer here)_

> 4. A frame arrives with a destination MAC belonging to no device on the
>    segment. Trace what happens.

_(answer here)_

> 5. In one sentence: why is a MAC address unsuitable for identifying a
>    machine's *location* in a large network?

_(answer here)_

---

# Open questions

Things noticed mid-lesson that don't belong to the current lesson. Park them
here rather than chasing them — most get answered by a later track, and the
ones that don't are worth revisiting deliberately.

- _(none yet)_

---

# Map changes

`2026-08-12` — Repo re-aimed. Goal changed from "operate the Fortinet stack on
mechanism" to "read a capture, a firewall log, a proxy decision or a detection
at the mechanism level." Track map redesigned against that goal and reviewed
cold by the `curriculum-critic` agent. Old Track 00 split (tap mechanics stayed
at the front, the epistemics became the Track 08 capstone); tracks 02–08 of the
old map replaced. Old Track 01 Lesson 01 survives unchanged and unworked — it
predates the **Artifact** section rule and the IPv6-alongside rule, and will be
brought into conformance before Lesson 02 is written.
