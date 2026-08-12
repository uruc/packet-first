# crash-course ledger

This lane's own record. Separate from the repo's [PROGRESS.md](../PROGRESS.md),
which belongs to the nine tracks and is not touched from here.

A block is done when its **prediction has been made in writing, run, and diffed**,
and its checkpoint answered **from memory**. A missed prediction is not a failure
— it is the pacing mechanism doing its job, and the block repeats.

Statuses: `not started` · `reading` · `hands-on done` · `predicted` ·
**`checked`**

| Block | Status | Prediction diffed | Checkpoint |
|---|---|---|---|
| [0 — What a packet is](00-what-a-packet-is.md) | not started | — | — |
| [1a — The mask](01a-the-mask.md) | not started | — | — |
| [1b — The plan](01b-the-plan.md) | not started | — | — |
| [1c — Getting an address](01c-getting-an-address.md) | not started | — | — |
| 2–9 | not written | — | — |

---

# Predictions

Written **before** running anything. Leave wrong predictions in the file — the
record of what was wrong is the most useful thing here, and it says what to
re-test in a month.

Fill in the three lines under each item. Type after the arrow; nothing has to
line up. If **pred** and **actual** match, write `match` on the diff line and
move on — the diff line only earns its space when you were wrong.

## Block 0

**0.1 — Destination MAC: gateway ping vs `1.1.1.1` ping. Same or different, and whose?**
- pred →
- actual →
- diff →

**0.2 — Frame size of the `ping -n 1 1.1.1.1` echo request, in bytes**
- pred →
- actual →
- diff → _(if off: which header was mis-sized?)_

**0.3 — Reply TTL from `1.1.1.1`; inferred initial TTL and hop count**
- pred →
- actual →
- diff →

## Block 1a

**1a.A1 — Local destination: local/remote, next hop, interface**
- pred →
- actual →
- diff →

**1a.A2 — Remote destination: local/remote, next hop, interface**
- pred →
- actual →
- diff →

**1a.A3 — Boundary-case destination: local/remote, next hop, interface**
- pred →
- actual →
- diff →

**1a.B — Five random prefixes: how many correct, and seconds per prefix**
- pred →
- actual →
- diff →

**1a.C — [home PC] `diagnose ip route lookup`: interface and gateway for each**
- pred →
- actual →
- diff →

**Total IPv6 addresses on `lx0r` right now** _(hands-on 4)_ →

## Block 1b

**1b.A — Can the six /24s be summarised into one route? Answer on instinct, before computing.**
- pred →
- actual →
- diff →

**1b.B1 — [home PC] connected subnet count**
- pred →
- actual →
- diff →

**1b.B2 — [home PC] total route count**
- pred →
- actual →
- diff →

**1b.B3 — The gap between B1 and B2, and which cause you'd bet on**
- pred →
- actual →
- diff →

**1b.C — [home PC] largest address group, member count**
- pred →
- actual →
- diff →

**Plan audit** _(hands-on 2)_ — sites or zones that are NOT one prefix →

## Block 1c

**1c.A1 — v4 neighbour count; Reachable vs Stale split**
- pred →
- actual →
- diff →

**1c.A2 — v6 neighbour count; Reachable vs Stale split**
- pred →
- actual →
- diff →

**1c.B — ARP frames between the flush and the first ping, and their direction**
- pred →
- actual →
- diff →

**1c.C — [home PC] ARP entry count; largest age; which rows vanish in 60 s**
- pred →
- actual →
- diff →

**1c.D — [home PC] DHCP pool range predicted from the plan; lease count**
- pred →
- actual →
- diff →

**`ping -6 ff02::1` reply count on this segment** _(hands-on 2)_ →

**Router advertisements observed on this segment, and from what** _(hands-on 4 —
"none observed" is a real finding; record the date)_ →

---

# Checkpoint answers

From memory, before scrolling back. Wrong answers stay.

## Block 0 — What a packet is

> 1. Layering is justified by a specific arithmetic argument. State it, then
>    state what property it buys and name one place in this stack where somebody
>    broke that property deliberately and what they got for it.

_(answer here)_

> 2. Every header you met carries two structural fields serving the same two
>    purposes. Name the purposes, and give the field that serves each in the
>    Ethernet, IPv4 and IPv6 headers.

_(answer here)_

> 3. The IPv4 header checksum covers the header only; the TCP checksum covers the
>    header, the payload, and fields from the IP header. Give the reason for each
>    choice, and name one device class that has to do extra work because of the
>    second.

_(answer here)_

> 4. Order these by how far they survive unmodified, and give the reason for the
>    ordering rather than the ordering alone: source MAC, source IP, source port,
>    TTL, TCP sequence number.

_(answer here)_

> 5. IPv6 deleted the header checksum. What made that safe, and what would have
>    become unsafe if UDP's checksum had stayed optional in v6?

_(answer here)_

## Block 1a — The mask

> 1. MAC addresses are 48 bits — 281 trillion of them. Explain, without using the
>    word "enough", why they cannot be used to build a global network, and name
>    the specific property IP addresses have that fixes it.

_(answer here)_

> 2. State the one decision the mask makes, then trace what a host does
>    *differently* on each branch — being precise about what goes in the IP header
>    and what goes in the Ethernet header in the remote case.

_(answer here)_

> 3. Two hosts share a wire. One is configured /24, the other /25. Describe a
>    concrete pair of addresses where they disagree about locality, what the
>    observable symptom is, and why no device logs an error.

_(answer here)_

> 4. Give the reason /31 is valid and /30 is wasteful, then give the reason /64 is
>    effectively mandatory in IPv6. One is about a header field being pointless;
>    the other is about a later mechanism's requirement. Say which is which.

_(answer here)_

> 5. `172.16.34.100/26` and `172.16.34.200/26`: same subnet or different? Give the
>    answer and the four steps that produced it, in under thirty seconds.

_(answer here — note how long it actually took)_

> 6. Why is `2001:db8::1:0:0:1` ambiguous to compress but unambiguous to expand,
>    and what does that cost anyone searching logs for a v6 address?

_(answer here)_

## Block 1b — The plan

> 1. Address efficiency is nearly worthless in RFC 1918 space, but bit-boundary
>    alignment is critical. Those sound contradictory. Explain why they are not.

_(answer here)_

> 2. What is the test for whether an organisational tier deserves bits in the
>    address plan? Give the test, and give an example of a tier that would fail it.

_(answer here)_

> 3. Consistent function encoding across sites buys something specific in firewall
>    policy. State it, and state what it means for the work of adding a new site.

_(answer here)_

> 4. Why is growth planned as reserved adjacency rather than as space at the end?
>    Describe the exact failure that appending causes.

_(answer here)_

> 5. An IPv6 plan deletes one entire step of the v4 method. Name it, name what
>    made it deletable, and say why v6 planning is genuinely easier despite
>    feeling harder.

_(answer here)_

> 6. Name two events that force NAT between internal networks regardless of how
>    good your plan was, and name the single design choice that most reduces the
>    chance of the first one.

_(answer here)_

## Block 1c — Getting an address, and finding a neighbour

> 1. An ARP request is broadcast and its reply is unicast. Give the reason for
>    each, from what each party knows at that moment.

_(answer here)_

> 2. ARP has no IP header. Name two consequences of that fact — one about where
>    ARP can go, one about what it doesn't need.

_(answer here)_

> 3. ND messages are sent with hop limit 255 and discarded if they arrive with
>    anything else. Explain what that check proves and why it was necessary for ND
>    but not for ARP.

_(answer here)_

> 4. Derive the solicited-node multicast address for
>    `2001:db8:10:20::5eaf:c0de`, and say how many hosts on a 500-host segment
>    process the resulting frame compared to an ARP request.

_(answer here)_

> 5. Why is the DHCP Request broadcast rather than unicast to the chosen server?
>    What breaks if it is unicast?

_(answer here)_

> 6. A DHCP relay inserts `giaddr`. What is it for, and what silently changes if
>    somebody re-addresses the relaying interface?

_(answer here)_

> 7. State the identity consequence of a lease timer in one sentence, then say
>    what a short guest-wireless lease trades away and what it buys.

_(answer here)_

> 8. Name every cache introduced in block 1, its timer, and the specific symptom
>    when it disagrees with reality. Five of them.

_(answer here — this one gets asked again at the end of block 6, with more rows)_

---

# Open questions

Noticed mid-block, not belonging to the current block. Park them rather than
chasing — most get answered by a later block, and the ones that don't are worth
revisiting deliberately.

- _(none yet)_

---

# Course changes

`2026-08-12` — Lane created. Ten blocks, ordering as specified by the owner, with
one subdivision: block 1 split into 1a (the mask), 1b (the plan) and 1c
(DHCP/ARP/ND). Reason recorded in [README.md](README.md#order-and-one-change-to-it).
Blocks 0 and 1 written; 2–9 deliberately not written in advance.
