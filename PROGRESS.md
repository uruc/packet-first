# PROGRESS

The ledger. A lesson is not done when it has been read — it is done when its
checkpoint has been answered **from memory**, and the answers are written below.

Statuses: `not started` · `reading` · `hands-on done` · **`checkpoint passed`**

Only `checkpoint passed` unlocks the next lesson.

---

## Track 00 — The tap

| Lesson | Status | Checkpoint passed |
|---|---|---|
| 00 — Placement: what is upstream of the tap | not started | — |
| 01 — The instrument lies: offload, stripping, stack position | not started | — |
| 02 — Records that are not packets: flow, log, schema | not started | — |

All three come before Track 01's hands-on. Track 00 has no prerequisites.

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
