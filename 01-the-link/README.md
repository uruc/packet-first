# Track 01 — The link, and what it leaks

**What this track answers:** what a link is, what identity means at layer 2, and
why the layer with the strongest identity is also the one whose evidence
survives the shortest distance.

Layer 2 is where a machine is least anonymous — it announces its address
unprompted, answers questions from anyone, and trusts every answer it gets. It
is also where that evidence stops: one router hop and the identity is gone,
replaced by the router's own. Both halves of that matter, and the second is why
a firewall log so often names a device you don't care about.

| # | Lesson | Answers |
|---|---|---|
| 01 | [The link](01-the-link.md) | What is a link, and what does a switch actually do? |
| 02 | The scope of a MAC | Why the address is rewritten at every hop, and what that does to evidence |
| 03 | Asking who is there — ARP and ND | Address resolution as one problem solved twice, and what each attempt trusts |
| 04 | Being told who is there — DHCP, RA, and the v6 problem | Configuration you did not ask for, and the record it leaves |
| 05 | What this leaks | Spoofing on both stacks, why "we don't run IPv6" is not a defence, and the artifacts each attack produces |

## The shape of it

Lessons 01–02 establish the mechanism and the boundary: a switch learns from
sources, forwards on destinations, floods when it doesn't know — and a router
rewrites the L2 header entirely, which is why a MAC address answers "which
device on my segment" and never "which device on my network."

Lessons 03–04 are the two ways a host finds out about its neighbours: it asks
(ARP, ND), or it is told (DHCP, DHCPv6, Router Advertisements). Both are covered
on both stacks in the same breath — not because v6 is coming, but because
comparing ND against ARP is the clearest available explanation of what ARP is
*for*, and because RA can hand a host a resolver with no DHCPv6 involved at all.

Lesson 05 is where the track lands on artifacts. ARP spoofing, ND spoofing,
rogue RA and the name-resolution fallbacks are the same trust failure with four
names, and each leaves a different, findable trace. The mechanism from 03–04
explains why every one of them works; the point of the lesson is what each one
puts in a capture or a log that you could actually alert on.

## Why start this low

Not because subnetting needs relearning. Because two questions that turn out to
be load-bearing later — *what identity does a packet carry, and how far does it
carry it?* — can only be answered from here, and every attribution problem in
Track 02 is downstream of the answer.

Expect to move through 01–02 quickly. Expect 03–05 to take real time.

## Note on Lesson 01

Lesson 01 was written under the repo's previous framing and predates two current
rules: it has no **Artifact** section, and it does not carry IPv6 alongside. It
is mechanically sound on switch behaviour and is worth working as-is; it will be
brought into conformance before Lesson 02 is written.
