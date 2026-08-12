# crash-course

A parallel fast lane. Ten blocks, 101 upward, no gaps, taught as **one connected
story**: each thing introduced as the fix for a problem the previous thing
created.

Started 2026-08-12.

## How this differs from the nine tracks

The tracks in the repo root answer *"I have been handed an artifact — what made
it, and where does it lie?"* They are aimed at reading evidence.

This lane answers a different question: **"I look at our stack and every design
decision makes sense — not that it works, but why it was done that way, and what
would happen if it were done differently."** It is aimed at building.

The two lanes share the repo and share `CLAUDE.md`'s rules, with two deliberate
exemptions granted for this directory only:

| Root rule | Status here |
|---|---|
| Out-of-scope list (HA, SD-WAN, IPsec, BGP, FMG/FAZ, STP…) | **Does not apply.** All of it is in the deployment, so all of it is in scope. |
| "Every lesson lands on an artifact" | **Relaxed** to mean config and CLI output, not only log fields. Still mandatory — a block with no artifact has failed. |

Everything else holds, and holds hard: no forward references, no fabricated
output, IPv6 alongside v4 rather than deferred, name the timers, hands-on
mandatory, **you run the commands**, destructive exercises ship the undo first.

The tracks, `PROGRESS.md` and the track map are untouched by this lane. This
directory keeps its own ledger in [LEDGER.md](LEDGER.md).

## How a block ends

Not with a quiz you grade yourself on. With a **prediction**:

> You write down what a command will output — *before* running it. Then you run
> it and diff your answer against reality.

That is the pacing mechanism. A prediction that lands means the block holds and
we move. A prediction that misses means we stop there and find out which
sentence in the block was wrong in your head, because that miss is worth more
than three more blocks of reading.

`diagnose ip route lookup`, `Find-NetRoute` and `ip route get` are the workhorses
— they ask the box what it *would* do without sending a packet, so a wrong
prediction costs nothing but the correction.

Each block also carries a short cold checkpoint. Answers go in
[LEDGER.md](LEDGER.md), from memory, before the next block.

## Where things run

| Box | Available |
|---|---|
| `lx0r` (the travelling laptop) | Hyper-V, WSL2, Wireshark, `pktmon`, PowerShell |
| Home PC | Hyper-V AD lab, Docker/WSL2 `lab-splunk`, **the physical FortiGates** |
| `VEGAS` | FortiManager `10.99.99.10`, FortiAnalyzer `10.99.99.20` |

Blocks 0–5 run almost entirely on `lx0r`. From block 6 the FortiGates start
carrying real weight. Any step needing gear you are not sitting at is flagged
**[home PC]** or **[VEGAS]** in the block, with the reading and the prediction
written so you can take them there and bring the answer back.

## The blocks

### 0 — What a packet is
Layering, encapsulation, headers, and why the model is built this way at all.
The scope-and-lifetime table: for every header field, who writes it, who reads
it, and how far it survives.

**After this you can:** open any capture and name, for each header, the exact
distance it travels before something rewrites it — and say why a firewall
carries a router's MAC and the sender's IP in the same record. Explain what a
layer *is* well enough to say what a tunnel does to the model. Read
`diagnose sniffer packet` at each verbosity and say what each level adds.

### 1 — Addressing
IPv4 and the mask maths — /32, /31, /30, /28, /24 — what the mask decides and
why it exists. IPv6 in the same breath. How an addressing plan is designed. Then
DHCP, ARP and ND. *Three sittings; the mask, the plan, then getting an address.*

**After this you can:** compute any prefix in your head, state the one decision
the mask makes and why every later mechanism hangs off that fork, design a plan
that summarises, and explain why a summarisable plan collapses a 300-rule policy
into 3. Say why a host has six IPv6 addresses. Name every cache introduced so
far and its timer.

### 2 — Layer 2
MAC and switching, broadcast domains and what they cost, VLANs, tagging and
trunks, inter-VLAN routing, link aggregation, and enough spanning tree to know
what a loop costs.

**After this you can:** say what a VLAN actually guarantees and what it doesn't,
read a trunk config and predict which frames carry a tag, explain why a loop
takes a switch down in seconds when a routing loop merely wastes bandwidth, and
choose an LACP hashing mode for a reason.

### 3 — Layer 3
The route table, longest prefix match, connected vs static vs dynamic, the
default route, what happens when two routes match, and OSPF and BGP at the depth
this stack actually uses them. *Probably two sittings, three with BGP.*

**After this you can:** resolve any route table by hand and predict the egress
before the box does, explain distance vs metric vs prefix length and their
precedence, say why OSPF is used where it is and BGP where it is, and read an
SD-WAN-adjacent route table without the SD-WAN part confusing you yet.

### 4 — Transport
TCP and UDP, ports, handshake and teardown, what a "session" is, and MTU, MSS
and fragmentation — including why a tunnel breaks things that worked before it.

**After this you can:** explain the difference between a session on the wire and
a session in a firewall's memory, and why only one of them has a timeout.
Diagnose the PMTUD black hole from symptoms alone, and say exactly where MSS
clamping puts its thumb on the scale and why it is a workaround rather than a
fix.

### 5 — Names and time
DNS resolution end to end. NTP, and why time is load-bearing for certificates,
logs and authentication.

**After this you can:** trace a name to the resolver that actually answered,
say which caches could be lying and for how long, and explain the three separate
failures a clock skew of ten minutes causes in this stack.

### 6 — Firewalls
Stateful vs stateless, the session table, interfaces and zones, policy match
order, NAT and where it sits relative to routing and policy, and where
inspection profiles run in the pipeline. *Two sittings, likely three.*

**After this you can:** state the ordered pipeline from ingress to egress from
memory and place every feature in it. Explain why a policy that "should match"
doesn't. Predict whether a rule change breaks established sessions. Say why NAT
happening before or after policy changes what you write in the rule.

### 7 — Encryption
Certificates and the trust chain, TLS, what inspection does and what it breaks.
Then IPsec: IKE, why phase 1 and phase 2 are separate, what is negotiated in
each. *Two sittings — TLS, then IPsec.*

**After this you can:** follow a chain from leaf to root and name every way it
breaks. Say what inspection sees, what it can't, and which applications it
breaks and why. Explain what a phase-2 rekey does to a session that a phase-1
rekey doesn't, and why the two lifetimes are separate numbers.

### 8 — Availability
HA and what actually fails over, what happens to existing sessions, and SD-WAN:
multiple paths, SLA measurement, and the order rules are evaluated in.

**After this you can:** predict which sessions survive a failover and which die,
and why. State the SD-WAN evaluation order from memory and predict which member
a given flow takes — then explain what would change if the strategy were
different.

### 9 — Operations
How logs are produced and shipped, what FortiManager and FortiAnalyzer are each
for, and a systematic method for finding where traffic died.

**After this you can:** run one repeatable procedure that finds the drop point,
in order, without guessing — and say which stage of which pipeline wrote every
field you used to do it.

## What this costs

Honest estimate, at 60–90 minutes a sitting:

| Block | Sittings |
|---|---|
| 0 — Packets | 1 |
| 1 — Addressing | 3 |
| 2 — Layer 2 | 3 |
| 3 — Layer 3 | 3–4 |
| 4 — Transport | 3 |
| 5 — Names and time | 2 |
| 6 — Firewalls | 3–4 |
| 7 — Encryption | 4 |
| 8 — Availability | 3 |
| 9 — Operations | 3 |
| **Total** | **28–32** |

Read that as a **floor, not a forecast.** It assumes predictions mostly land. It
does not include repeats, and repeats are the design working rather than
failing — a block that gets re-taught because a prediction missed is the single
highest-value hour in the whole course. Budget 30–36 and be pleasantly surprised.

Two honest caveats on the estimate:

- **Blocks 3, 6 and 7 are where it will actually slow down**, and they are also
  where most of your "why was it done this way" questions live. If the schedule
  slips, it slips there, and that is the right place for it to slip.
- **The estimate assumes the foundation checks out.** Which parts of it are solid
  is unverified by design. If block 1's predictions come back clean the middle
  blocks get faster; if they don't, the total goes up and that is information
  worth having on sitting three rather than sitting twenty.

## Order, and one change to it

Taught in the order you listed, with a single deviation, flagged as required:

**Block 1 is split into three sittings** — the mask, the plan, then DHCP/ARP/ND.
Not a reordering, a subdivision. The mask maths and IPv6 addressing alone is a
full sitting if it is going to be done properly rather than skimmed, and the
plan-design material only makes sense once the maths is automatic. Everything
else stays where you put it.

Nothing else moved. The dependency chain you sketched already holds: broadcast
domains have to cost something before VLANs are a fix, VLANs have to hit a wall
before routing is a fix, addressing has to run out before NAT is a fix, and the
tunnel has to add bytes before MSS clamping is a fix.

## Blocks written so far

| Block | File | Status |
|---|---|---|
| 0 | [00-what-a-packet-is.md](00-what-a-packet-is.md) | written |
| 1a | [01a-the-mask.md](01a-the-mask.md) | written |
| 1b | [01b-the-plan.md](01b-the-plan.md) | written |
| 1c | [01c-getting-an-address.md](01c-getting-an-address.md) | written |
| 2–9 | — | not written |

Blocks are written as they are reached, not in advance. Predictions set the pace.
