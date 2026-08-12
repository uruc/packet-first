# Block 0 — What a packet is

> **What this answers:** What is actually on the wire, why is it arranged in
> layers, and how far does each part of it travel before something rewrites it?

One sitting. Runs on `lx0r`. One optional section is flagged **[home PC]**.

---

## Start with the problem, not the model

Everyone is taught the OSI model as though it came first and networks were built
to comply with it. Backwards. The layers are a *solution*, and you cannot judge a
solution you have not seen the problem for.

The problem: you have some number of **applications** that want to move data —
mail, file transfer, remote terminal — and some number of **physical media** that
can carry it — coax, twisted pair, fibre, radio, a virtual switch inside a
hypervisor. If each application has to know how to drive each medium, you write
*applications × media* implementations. Add one new medium and you rewrite every
application. Add one new application and you implement it for every medium.

Layering makes that *applications + media*. You define one interface in the
middle. Above it, applications produce data without knowing what carries it.
Below it, media carry bytes without knowing what they mean. Both sides can change
independently, forever.

That is the entire justification. Not elegance — **substitutability.** Wi-Fi
replaced a coax segment without a single application being recompiled, and that
is the layering paying out. Hold onto the word *substitutability*; it is the
thing every layer boundary is buying, and later in this course you will meet
boundaries where somebody deliberately broke it to buy something else.

## Encapsulation is how layering is implemented

Layering is the idea. **Encapsulation is the mechanism.**

Each layer takes whatever the layer above handed it, treats it as an **opaque
blob it must not look inside**, prepends its own header, and hands the result
down. The receiver reverses it: strip your header, hand the rest up, don't look
at it on the way.

```
    application data                              "GET / HTTP/1.1..."
    ├─ TCP prepends its header            [TCP][ application data ]
    ├─ IP prepends its header        [IP ][TCP][ application data ]
    └─ Ethernet prepends its header [Eth][IP ][TCP][ application data ][FCS]
```

Two words for one thing, worth separating because people use them
interchangeably and then get confused:

- A **header** is what a layer prepends. Fixed structure, defined by that layer's
  spec.
- The **payload** is everything after it. To that layer, structureless. To the
  layer above, the whole world.

So "what is a packet?" has a precise answer, and it is not "a message":

> **A packet on the wire is a stack of headers followed by whatever the topmost
> header's owner hasn't looked at yet.**

The outermost header is the most *local* — it concerns this hop only. The
innermost is the most *end-to-end*. That gradient is the single most useful thing
in this block, and section "Scope" below turns it into a table you will use for
the rest of the course.

### The bit that trips people: "opaque" is a promise, not a guarantee

Nothing physically prevents a layer from reading inside its payload. The
encapsulation contract is a *design discipline*, and a firewall's entire job is
violating it on purpose — reading the TCP header it is only supposed to carry,
then reading the HTTP inside that. Every "layer 7 firewall", every IPS engine,
every SSL inspection profile is a deliberate layering violation sold as a
feature.

Keep those separate in your head:

- **Layering violations by design** — a middlebox looking deeper than its role
  requires. Firewalls, load balancers, NAT. Powerful, and the reason this stack
  exists.
- **Layering violations in the protocols themselves** — cases where the spec
  itself makes one layer depend on another's fields. Rarer, uglier, and each one
  causes a specific class of bug you will meet later. The TCP pseudo-header is
  the canonical one, below.

## Why the model has the layers it has

Two models. You need both, for different reasons.

| | OSI | TCP/IP (what actually ships) |
|---|---|---|
| 7 | Application | **Application** — HTTP, DNS, SMB |
| 6 | Presentation | ” |
| 5 | Session | ” |
| 4 | Transport | **Transport** — TCP, UDP |
| 3 | Network | **Internet** — IP |
| 2 | Data link | **Link** — Ethernet, Wi-Fi |
| 1 | Physical | ” |

The honest position: **OSI is a vocabulary, TCP/IP is an implementation.** No
mainstream stack has separate session and presentation layers; those functions
live inside applications and inside TLS. Nobody has ever written a "layer 6
driver."

So why does everyone still say layer 3 and layer 7? Because as a *vocabulary* it
is precise and vendor-neutral, and the numbers survive in the product names you
work with daily: layer-3 device, L2 switch, layer-7 inspection, L4 load
balancer. Use the numbers. Just don't believe that six and five are running
anywhere.

The layers that genuinely exist, and the one question each answers:

| Layer | The question it answers | Scope of the answer |
|---|---|---|
| **Link (2)** | Which box on *this wire* do I hand this to next? | One hop |
| **Internet (3)** | Which box *anywhere* is this ultimately for? | End to end |
| **Transport (4)** | Which *program* on that box, and is this conversation reliable? | End to end |
| **Application (7)** | What does the data mean? | End to end |

Read that table down the "scope" column. **The layers are ordered by scope, and
that is the actual organising principle** — not abstraction, not "closeness to
the user." Layer 2 is the shortest-lived and most local; each layer up survives
further. Once you see it as a scope gradient, the encapsulation order stops being
arbitrary: the header that has to be rewritten most often is the one that has to
be cheapest to reach, so it goes outermost.

That is also why the physical layer keeps getting merged away in the TCP/IP view.
"Voltage on a wire" and "frame on a wire" have the same scope — one hop — so
splitting them buys the model nothing.

## The bytes, concretely

Enough theory. Here is what is actually there, in order, for an ordinary IPv4 TCP
packet on Ethernet. Offsets are from the first byte the wire carries.

### Ethernet header — 14 bytes

| Offset | Size | Field |
|---|---|---|
| 0 | 6 | Destination MAC |
| 6 | 6 | Source MAC |
| 12 | 2 | EtherType |

That's it. Three fields. **Destination comes first, and that is not an accident**
— a switch can start a forwarding decision after six bytes have arrived, before
the rest of the frame has even been clocked in. Header field order is a hardware
decision.

EtherType says what the payload is, so the receiver knows which parser to hand it
to. The four you will meet constantly:

| EtherType | Payload |
|---|---|
| `0x0800` | IPv4 |
| `0x86DD` | IPv6 |
| `0x0806` | ARP |
| `0x8100` | 802.1Q VLAN tag (block 2) |

There is also a 4-byte **FCS** (frame check sequence) at the *end* of the frame —
a trailer, not a header, because the sender computes it over everything and can
only append it once it knows everything. Your capture tools usually will not show
it: the NIC verifies and strips it before the OS sees the frame. A capture with
no FCS is normal.

### IPv4 header — 20 bytes minimum

The fields that earn their place in your head:

| Field | Why you care |
|---|---|
| **Version / IHL** | IHL is header length in *32-bit words*, so 5 = 20 bytes. It exists because options can extend the header, so nothing downstream can assume 20. |
| **Total length** | Header + payload, in bytes. The only authority on where this packet ends. |
| **Identification, Flags, Fragment offset** | Fragmentation. Block 4. |
| **TTL** | Decremented every hop; hits zero, packet dies. Loop insurance. |
| **Protocol** | What's inside: 6 = TCP, 17 = UDP, 1 = ICMP, 50 = ESP. Same job as EtherType, one layer up. |
| **Header checksum** | Covers **the header only**, not the payload. |
| **Source address / Destination address** | 4 bytes each. Block 1. |

Two of those deserve a beat.

**The header checksum covers only the header.** That is a deliberate choice: every
router decrements TTL, so every router changes the header, so every router must
recompute the checksum. Making it cover the payload too would mean every router
checksumming every byte of every packet at line rate. The design pushed
payload integrity up to layer 4, where only the endpoints pay for it. This is the
scope gradient again — per-hop work stays cheap, end-to-end work happens once.

**The protocol field is the same idea as EtherType.** Each header names its
payload's type so the next parser can be selected. You will see this pattern at
every layer, and once you name it, protocol stacks stop needing memorisation:
each layer carries a "what's next" field and a "how long am I" field, and those
two are all a parser needs.

### IPv6 header — 40 bytes, fixed

Bigger, and simpler, and the differences are all corrections of v4 mistakes.

| v6 has | v4 had | Why it changed |
|---|---|---|
| Fixed 40 bytes, no IHL | Variable, IHL required | Routers can find the payload at a constant offset. Faster in hardware. |
| **No header checksum** | Header checksum | L2 already checks (FCS), L4 already checks. v4's was redundant work at every hop. |
| **Next header** | Protocol | Same job, but chains — extension headers link to each other before the real payload. |
| **Hop limit** | TTL | Honest rename. It was always a hop count, never a time. |
| No fragmentation fields | Fragmentation fields | Routers may not fragment in v6. Block 4, and it matters more than it sounds. |
| 16-byte addresses | 4-byte addresses | Block 1. |

The v6 header being *fixed* while carrying *more* address space is the trade the
whole design turns on: pay 40 bytes always, in exchange for parsing that never
branches.

### TCP header — 20 bytes minimum

| Field | Why you care |
|---|---|
| **Source port / Destination port** | 2 bytes each. Which program. Block 4. |
| **Sequence / Acknowledgement** | Byte counters. Block 4. |
| **Data offset** | Header length in 32-bit words — again, because options extend it. This is where MSS lives, block 4. |
| **Flags** | SYN, ACK, FIN, RST, PSH, URG. The state machine. |
| **Window** | Flow control. |
| **Checksum** | Covers header **and** payload — **and fields copied from the IP header.** |

That last one is the layering violation to remember. TCP's checksum includes a
**pseudo-header**: source IP, destination IP, protocol, TCP length — fields that
belong to layer 3, reached down into by layer 4.

Why? So that a packet delivered to the wrong host, or with a corrupted
destination address, fails the transport checksum rather than being accepted by a
machine it was never for. It closes a real hole. It also means:

> **Anything that rewrites an IP address must recompute the TCP and UDP
> checksums.**

Which is why NAT is not a header edit — it is a header edit plus a fix-up at a
layer that has no business knowing about it. Every NAT device in the world pays
this tax. Park it; block 6 collects.

UDP has the same pseudo-header. In IPv4 its checksum is optional (may be sent as
zero, meaning "not computed"). In IPv6 it is **mandatory** — because v6 deleted
the network-layer checksum, so if UDP skipped it too, nothing would be checking
anything above the link.

## Scope: how far each header survives

This is the payoff table for the block. Print it, or at least be able to
reconstruct it.

| Header | Written by | Rewritten | Survives |
|---|---|---|---|
| **Ethernet dst/src MAC** | Sending NIC, or the last router | **Every single hop** | One link |
| **EtherType** | Sending NIC | Every hop (with the header) | One link |
| **VLAN tag** | Switch or NIC on ingress to a trunk | Added and stripped per link | Often less than one hop |
| **IP src/dst** | Originating host | Only by a NAT device | End to end, unless NAT |
| **TTL / Hop limit** | Originating host | Decremented **every hop** | End to end, degrading |
| **IP protocol / next header** | Originating host | No | End to end |
| **TCP/UDP ports** | Originating host | Only by a **PAT** device | End to end, unless NAT |
| **TCP flags/seq** | Endpoints | Only by a proxy | End to end |
| **Payload** | Application | Only by a proxy or inspection | End to end |

Read the "rewritten" column and three things fall out that people otherwise
memorise as trivia:

1. **A firewall log carrying a MAC address is telling you about the last hop, not
   the sender.** If the traffic crossed a router, the MAC is the router's. Nothing
   is broken; the MAC's scope is one link and it has been rewritten at every one.
2. **TTL is a distance measurement you get for free.** It is the only field that
   changes predictably per hop, so the gap between a common initial value (64,
   128, 255) and the observed value counts routers.
3. **NAT is the only reason an IP address is not end-to-end**, and PAT is the only
   reason a port isn't. That is why "the source IP in this log" is a question
   about *where the log was taken* before it is a question about who sent it.

That third point is where this lane and the nine tracks shake hands. Same fact,
opposite direction: they read it out of an artifact, you are about to build the
thing that causes it.

## What a tunnel does to all of this

One more idea before the hands-on, because it reframes the whole model and you
need it before block 7 rather than after.

Nothing says a payload has to be application data. If a payload is *another
complete packet, headers and all*, you have a **tunnel**:

```
[Eth][IP outer][ESP][IP inner][TCP][ application data ]
 └── what the internet sees ──┘└──── what the endpoints think is happening ───┘
```

The stack is no longer four layers. It is however many you nested. And the scope
table above now applies **twice, independently**: the outer IP header is
end-to-end between the *tunnel endpoints*, the inner one is end-to-end between
the *real* endpoints, and a device in the middle can only see and act on the
outer one.

Two consequences, stated now and cashed later:

- **A middlebox sees only as deep as the outermost encapsulation it can parse.**
  Your firewall's inspection is scoped to what it can peel. That is block 6 and 7.
- **The bytes are additive.** The outer headers are real bytes on a real link with
  a real size limit, and they were not there before you built the tunnel. That is
  block 4, and it is exactly why a tunnel breaks things that worked fine the day
  before.

## Caches and timers introduced by this block

**None.** Nothing here has state, and nothing here can go stale.

Note that, because it is the last time it will be true. Every block from here
introduces at least one cache with an expiry — MAC tables, ARP entries, DHCP
leases, route adjacencies, session tables, DNS TTLs, SA lifetimes, log receive
lag. A packet is a pure value; everything that *handles* a packet is a cache with
a timer disagreeing with reality.

---

## Artifact

Where this block surfaces in something you would actually type.

### Wireshark's packet detail pane *is* the encapsulation stack

The collapsible tree is not a UI convenience — it is literally the header stack,
outermost at the top, each node the header whose payload is the node below. When
you expand `Ethernet II → Internet Protocol → Transmission Control Protocol →
HTTP`, you are walking the de-encapsulation the receiving host walks.

The status bar tells you the same thing numerically: `Frame: 74 bytes` vs the
bytes each layer claims. 74 = 14 (Eth) + 20 (IP) + 20 (TCP) + 20 (options).

### `diagnose sniffer packet` verbosity levels **[home PC]**

FortiOS's on-box sniffer takes a verbosity argument that maps directly onto how
much of the header stack it prints:

```
diagnose sniffer packet <interface> '<filter>' <verbosity> <count> <timestamp>
```

| Verbosity | What it prints |
|---|---|
| 1 | IP header only — addresses, protocol |
| 2 | IP header + payload bytes |
| 3 | IP header + Ethernet header + payload |
| 4 | as 1, plus the **interface name** |
| 5 | as 2, plus the interface name |
| 6 | as 3, plus the interface name |

The pattern is `1/2/3` = how deep, `+3` = also name the interface. That the
Ethernet header is the *last* thing added, at level 3, is the scope table showing
up in a product decision: the least-durable header is the least often useful.

Levels 4–6 are what you almost always want, and the reason is worth stating —
without the interface name you cannot tell ingress from egress, and half of
firewall troubleshooting is "did it come in, and did it go out."

---

## Hands-on

About 35 minutes on `lx0r`. Nothing here changes any configuration, so there is
nothing to undo.

### 1. Build the stack from one ping

Open Wireshark on your active adapter, filter `icmp`, and run this in another
window:

```powershell
ping -n 2 1.1.1.1
```

*Expected:* four packets, two echo requests and two replies.

Click one request and expand every node in the detail pane. Then answer, from the
pane itself:

- How many bytes is the frame, and what does each layer contribute?
- What is the EtherType, and what does its value tell you the next parser is?
- What is the IP protocol number?
- Is the destination MAC the MAC of `1.1.1.1`? (It is not. Be able to say why in
  one sentence — that sentence is most of block 1.)

### 2. Read the raw bytes and find a field by hand

With that same packet selected, look at the hex pane. Count to the field rather
than clicking it:

- Bytes 0–5: destination MAC
- Bytes 6–11: source MAC
- Bytes 12–13: EtherType
- Byte 14: version and IHL packed into one byte — expect `0x45`
- Byte 22: TTL
- Bytes 26–29: source IP
- Bytes 30–33: destination IP

*Expected:* byte 14 reads `45` — version 4, IHL 5, meaning 5 × 4 = 20 bytes of
IPv4 header. Doing this once by hand is worth more than reading it three times,
because after this the header layout is a place you can navigate rather than a
diagram you half-remember.

### 3. See the scope gradient move

Ping something on your own link (your default gateway) and something far away
(`1.1.1.1`). Get the gateway with:

```powershell
Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Select-Object NextHop, InterfaceAlias
```

Capture both. Compare, across the two captures:

- The **destination MAC** — same or different?
- The **destination IP** — same or different?
- The **TTL on the replies** — same or different?

*Expected:* the destination MAC is the **same** for both (your gateway's) while
the destination IP differs; and the reply TTL from the far host is noticeably
lower than from the gateway. That is the entire scope table in one observation:
layer 2 is about the next hop, layer 3 is about the destination, and TTL counts
the distance between them.

### 4. See IPv6's fixed header

```powershell
ping -6 -n 2 ipv6.google.com
```

If that resolves and replies, capture it and compare header sizes against the v4
ping. *Expected:* 40 bytes of IPv6 header against 20 of IPv4, no checksum field
in the v6 header, and `Next Header: ICMPv6 (58)` where v4 had `Protocol: ICMP
(1)`.

If it does **not** resolve or does not reply, that is a finding rather than a
failure — note in the ledger which it was (no AAAA record vs no route vs no
address). Block 1 explains it, and "we don't run IPv6" is not the explanation.

---

## Checkpoint

Answer from memory in [LEDGER.md](LEDGER.md), without scrolling back.

1. Layering is justified by a specific arithmetic argument. State it, then state
   what property it buys and name one place in this stack where somebody broke
   that property deliberately and what they got for it.
2. Every header you met carries two structural fields serving the same two
   purposes. Name the purposes, and give the field that serves each in the
   Ethernet, IPv4 and IPv6 headers.
3. The IPv4 header checksum covers the header only; the TCP checksum covers the
   header, the payload, **and** fields from the IP header. Give the reason for
   each choice, and name one device class that has to do extra work because of
   the second.
4. Order these by how far they survive unmodified, and give the reason for the
   ordering rather than the ordering alone: source MAC, source IP, source port,
   TTL, TCP sequence number.
5. IPv6 deleted the header checksum. What made that safe, and what would have
   become unsafe if UDP's checksum had stayed optional in v6?

---

## Prediction

Do this **before** running anything. Write the answers in the ledger, then run
and diff.

You are going to ping your default gateway and then `1.1.1.1`, capturing both.

**Predict, in writing:**

1. The exact destination MAC in each capture — the same value or two different
   values, and whose it is.
2. The exact total frame size of the echo request, in bytes, from
   `ping -n 1 1.1.1.1` on Windows. (Windows' default ICMP payload is 32 bytes.
   Add up the stack yourself.)
3. The reply TTL from `1.1.1.1`, to within a few — and from that number, state
   the initial TTL you think the responder used and how many hops away it is.

**Then run:**

```powershell
ping -n 1 1.1.1.1
```

and read the reply's TTL from the console output, plus the frame size from
Wireshark's frame line.

**Then check the prediction cheaply, without sending anything:**

```powershell
Find-NetRoute -RemoteIPAddress 1.1.1.1
Find-NetRoute -RemoteIPAddress <your gateway IP>
```

*Expected:* both return the same interface and the same next hop, which is the
mechanism behind prediction 1. `Find-NetRoute` is the Windows equivalent of
`diagnose ip route lookup` — it asks the stack what it *would* do. It becomes the
main instrument of this course from block 3 onward; this is the introduction.

The Linux form, if you would rather run it in WSL2:

```bash
ip route get 1.1.1.1
```

**Grade yourself on prediction 2 specifically.** If the byte count was off,
find which header you mis-sized before moving to block 1 — the arithmetic of a
header stack is load-bearing for MTU and MSS in block 4, and being 4 bytes out
there is the difference between a working tunnel and an intermittently broken one.
