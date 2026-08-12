# Block 1a — The mask

> **What this answers:** Why does an IP address exist when a MAC address already
> identifies a machine, and what is the one decision the mask makes?

One sitting. Runs on `lx0r`. FortiGate sections flagged **[home PC]**.

Depends on: block 0 (the header stack, and the scope gradient).

---

## The problem layer 2 created

Block 0 left you with a working model of a frame: destination MAC, source MAC,
EtherType, payload. On one wire that is complete. A switch learns which port each
MAC lives behind and forwards accordingly, and nothing above layer 2 is needed
for two machines on the same segment to talk.

So why is there a layer 3 at all?

Not because MACs run out. 48 bits is 281 trillion addresses. The problem is not
quantity, it is **structure**.

A MAC address is **flat**. `00:1A:2B:3C:4D:5E` tells you the manufacturer (the
first 24 bits, the OUI) and *nothing whatsoever about where the device is*. Two
MACs one digit apart may be on opposite sides of the planet. Which means:

> **A forwarding device cannot summarise MAC addresses. To forward to any MAC, it
> must have learned that specific MAC.**

Follow that to its conclusion. A global layer-2 network would require every
switch to hold an entry for every device on Earth, learned by flooding, aged out
by a timer, re-flooded when it expires. The table would be billions of rows and
the flooding alone would consume the network. Layer 2 does not scale, and the
reason is not the address size — it is that flat addresses cannot be
**aggregated**.

The fix is to make addresses **carry location**. If an address's leading bits say
*where*, a forwarding device can hold one entry covering millions of hosts:
"anything starting with these bits, that way." One row instead of a million.

That is what an IP address is. Not "a better MAC" — a **hierarchical** address,
whose leading bits are a location and whose trailing bits are an individual.

Everything in this block follows from that one sentence, including the mask.

## An address is two fields wearing a disguise

`10.20.30.40` looks like one value. It is two, concatenated:

```
    10.20.30.40 / 24

    00001010 00010100 00011110 | 00101000
    └────── network ─────────┘ | └ host ┘
```

- The **network part** (prefix): which link this address lives on.
- The **host part**: which machine on that link.

The dotted-decimal notation actively hides this, because the boundary does not
have to fall on a dot. That is the single biggest reason people who "know
subnetting" still get it wrong under pressure: they have learned octet
arithmetic instead of bit arithmetic.

**The mask's only job is to say where the boundary is.** `/24` means "the first 24
bits are the network." Written the old way, `255.255.255.0` — 24 one-bits
followed by 8 zero-bits. Same statement, worse notation.

### The mask is not part of the address

This matters more than it sounds and it will bite you in the deployment.

An address is carried in the IP header. **The mask is not.** Look back at block
0's IPv4 header table — source address, destination address, four bytes each, no
mask anywhere. The mask is **local configuration on each host**, never
transmitted, and each host's copy is independent.

Consequences you will meet:

- Two hosts on one wire configured with different masks will disagree about who
  is local. Each is internally consistent. The result is one-way reachability:
  A sends direct, B replies via the router, and if the router doesn't have a
  path, the conversation half-works. Nothing logs an error, because neither host
  believes it is doing anything wrong.
- A device receiving a packet learns the sender's address but **not** the sender's
  mask. It cannot know how big the sender's subnet is. Anything that appears to
  know is inferring it from a routing table, not reading it off the wire.

## The one decision

Here is the sentence this whole block exists to install:

> **A host applies its mask to compare the destination address with its own. If
> the network parts match, the destination is on this link and it is delivered
> directly. If they differ, the packet is handed to a router.**

That is the fork. It is the reason the mask exists, and every layer-3 mechanism
in the rest of this course is an elaboration of one side of it or the other.

The arithmetic, done properly:

```
    my address    10.20.30.40   00001010 00010100 00011110 00101000
    my mask       /24           11111111 11111111 11111111 00000000
    AND           →             00001010 00010100 00011110 00000000  = 10.20.30.0

    destination   10.20.30.99   00001010 00010100 00011110 01100011
    my mask       /24           11111111 11111111 11111111 00000000
    AND           →             00001010 00010100 00011110 00000000  = 10.20.30.0

    equal  →  local  →  ARP for 10.20.30.99, frame goes straight to it
```

```
    destination   10.20.40.99   →  AND  →  10.20.40.0
    not equal  →  remote  →  ARP for the *gateway*, frame goes to the router
```

Note carefully what happens in the remote case, because it is the answer to
block 0's hands-on question 1 and it is the most commonly muddled thing in
networking:

> **The destination IP is the far host. The destination MAC is the router.**

Two headers, two scopes, two different answers to two different questions —
exactly as block 0's scope table said. The host does not "send the packet to the
router." It sends a packet *addressed to the far host* inside a frame *addressed
to the router*. The router strips the frame, looks at the IP header, and builds a
new frame. The IP header survives; the Ethernet header does not.

**Both branches end in the same next step: the host needs a MAC address it does
not have.** That is block 1c's problem, and ARP and ND are its answer.

## The maths, fast

You need this automatic, not derivable. Derivable is not good enough at 2 a.m.
with a change window closing.

### The only table you have to memorise

Powers of two, and what each mask length does to the last octet it touches:

| Mask | Last-octet value | Block size | Addresses | Usable hosts (v4) |
|---|---|---|---|---|
| /24 | 0 | 256 | 256 | 254 |
| /25 | 128 | 128 | 128 | 126 |
| /26 | 192 | 64 | 64 | 62 |
| /27 | 224 | 32 | 32 | 30 |
| /28 | 240 | 16 | 16 | 14 |
| /29 | 248 | 8 | 8 | 6 |
| /30 | 252 | 4 | 4 | 2 |
| /31 | 254 | 2 | 2 | **2** (special, below) |
| /32 | 255 | 1 | 1 | 1 (special, below) |

Two patterns make this recall rather than arithmetic:

- **Block size = 256 − mask octet.** /28 → 256 − 240 = 16. Subnets therefore start
  at multiples of 16: .0, .16, .32, .48…
- **Addresses double each time you shorten by one bit.** /26 is 64, so /25 is 128,
  so /24 is 256.

### Working a prefix in your head, in four steps

Take `172.16.34.100/26`. Which subnet, and what is its range?

1. **Which octet does the boundary fall in?** /26 → bits 25–32 are the last octet,
   so the interesting octet is the fourth. (/1–/8 → first, /9–/16 → second,
   /17–/24 → third, /25–/32 → fourth.)
2. **Block size** = 256 − 192 = 64.
3. **Round the interesting octet down to a multiple of the block size.** 100 → 64.
   Network is `172.16.34.64`.
4. **Range** = network to network + block − 1 → `.64` to `.127`. Broadcast is
   `.127`, usable hosts `.65`–`.126`.

Practise until step 3 is instant. Everything else in layer 3 assumes it.

### Why "minus two"

`.0` (all host bits zero) is the **network address** — the name of the subnet
itself, the thing that appears in a route table. `.127` in the example above (all
host bits one) is the **broadcast address** for that subnet, meaning "every host
here."

That is where the two go. It is also where the exceptions come from.

### /31 — and why it exists

A point-to-point link has exactly two devices. With /30 you burn four addresses to
address two: network, host, host, broadcast. Fifty percent waste, and on a router
with hundreds of WAN links that is real space.

RFC 3021 observed that on a link with exactly two devices, **a broadcast address
is pointless** — "everyone on this link" and "the other end" are the same thing,
and a unicast reaches it. So /31 defines both addresses as usable hosts. No
network address, no broadcast address, 100% efficiency.

Two addresses, both usable. That is the whole rule, and it only makes sense on
point-to-point links.

### /32 — the address with no subnet

All 32 bits are network, no host bits at all. It names exactly one address. It
appears in three places you will actually see:

- **Loopbacks.** A router's loopback is a /32 because it isn't on a link at all —
  it is an address that belongs to the *box*, reachable no matter which physical
  interface survives. This is why loopbacks are used as router IDs and as IPsec
  and BGP endpoints. Block 3 and block 7 both cash this.
- **Host routes.** "This one address, this way." Longest prefix match (block 3)
  means a /32 always wins.
- **Firewall address objects** for a single server.

### /30, /28, /24 in practice

- **/30** — point-to-point links, pre-RFC-3021 habit, still extremely common
  because some platforms and some operators never adopted /31.
- **/28** — 14 hosts. The natural size for a small server segment or a DMZ, where
  you want the failure and broadcast domain small and you know the host count.
- **/24** — 254 hosts. The default human-scale unit. Not because 254 is a magic
  number, but because it falls on an octet boundary so a human can read it
  without doing binary. That is genuinely the reason, and it is a good one:
  operational legibility is worth address space.

## IPv6, in the same breath

Not a separate topic. The same two-part structure with different numbers and
three v4 mistakes corrected.

### Notation

128 bits, written as eight groups of four hex digits, and two compression rules:

1. Leading zeros in a group may be dropped: `2001:0db8:0000:0042` →
   `2001:db8:0:42`.
2. **One** run of all-zero groups may be replaced by `::`. Once only, because
   twice would be ambiguous — a parser reconstructs the missing groups by
   counting, and it cannot split an unknown total between two gaps.

`2001:0db8:0000:0000:0000:0000:0000:0001` → `2001:db8::1`.

### The prefix boundary is fixed at /64

In v4 you choose the mask per subnet, sizing it to the host count. In v6 you
essentially do not:

> **Every ordinary subnet is a /64.** 64 bits of network, 64 bits of interface
> identifier.

Not a convention someone can override casually — SLAAC (block 1c) constructs the
host part from a 64-bit interface identifier, so a prefix longer than /64 breaks
automatic addressing outright. A /64 holds 18 quintillion addresses for a segment
with 30 machines on it, and that waste is the *point*: address space was made
enormous precisely so that sizing could stop being a design activity.

Sit with that, because it is the design lesson of v6 and it is not about
addresses. **They spent abundance to buy simplicity.** Every hour anyone has ever
spent computing whether a segment needs a /27 or a /26 was an hour spent
compensating for scarcity.

Two exceptions, both mirroring v4:

| v6 | v4 analogue | Use |
|---|---|---|
| **/127** (RFC 6164) | /31 | Point-to-point router links |
| **/128** | /32 | Loopbacks, host routes, single-host objects |

### There is no broadcast address

v6 removed broadcast entirely. Nothing to reserve, nothing to subtract. Where v4
broadcasts, v6 uses **multicast** to a scoped group — so only interested hosts
process the frame instead of every NIC on the segment interrupting its CPU.
Block 1c shows what this does to ARP.

The address ranges to know now:

| Prefix | Name | What it is |
|---|---|---|
| `2000::/3` | GUA | Global unicast — routable internet addresses |
| `fc00::/7` (in practice `fd00::/8`) | ULA | The rough analogue of RFC 1918 |
| `fe80::/10` | **Link-local** | Automatic, **always present**, never routed |
| `ff00::/8` | Multicast | Includes the ones that replace broadcast |

**Link-local is the one to internalise.** Every v6-capable interface that is up
has an `fe80::` address whether anyone configured v6 or not, and it works without
a router, a DHCP server or an administrator. That has a direct consequence for
your deployment which is the subject of block 1c's warning, and it is why "we
don't run IPv6" is a statement about what you are *watching*, not about what is
*happening*.

### Multiple addresses per interface is normal

In v4 an interface has one address and a second is an anomaly. In v6 an interface
routinely has:

- a link-local `fe80::` address, always;
- a global address from SLAAC;
- possibly a **temporary/privacy** address that rotates on a timer;
- possibly a DHCPv6-assigned address;
- plus the multicast groups it has joined.

Which one gets used as a source is decided by rules (RFC 6724), not by
configuration. That is block 1c. For now, just stop expecting one address per
interface — the expectation is a v4 habit, not a networking fact.

## Caches and timers introduced by this block

**None yet.** An address and a mask are static configuration.

But note what is now *pending*: the mask told the host whether the destination is
local, and in both branches the host now needs a MAC it does not have. The
mechanism that gets it is a cache, with a timer. That is block 1c, and it is
where the timer theme starts for real.

---

## Artifact

### FortiOS **[home PC]**

Configuration — the mask entered the old way, which is worth noticing as a
reminder that the two notations are the same statement:

```
config system interface
    edit "port2"
        set ip 10.20.30.1 255.255.255.0
        set ip6-address 2001:db8:20:30::1/64
    next
end
```

Reading it back:

```
get system interface port2
diagnose ip address list
diagnose ipv6 address list
```

*Expected:* `diagnose ip address list` prints one line per configured address with
its prefix — note it shows the interface index and address, and that the mask
appears here as configuration rather than as anything received from the network.

The thing to actually look for: **configuring that address silently created a
route.** Check with:

```
get router info routing-table connected
```

*Expected:* a connected route for `10.20.30.0/24` via `port2`, which nobody typed.
That is the mask's second job — it is not only the local/remote test, it is the
statement that *generates* the connected route. Block 3 builds the whole route
table on top of this.

### Windows

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength
Get-NetIPAddress -AddressFamily IPv6 | Select-Object InterfaceAlias, IPAddress, PrefixLength, SuffixOrigin, AddressState
```

*Expected on the v6 line:* several addresses per interface, including at least one
`fe80::`, and a `SuffixOrigin` column distinguishing `Link`, `Random` and `Dhcp`.
That column is the multiple-addresses story made visible.

### Linux (WSL2)

```bash
ip -br addr
ip -br -6 addr
```

*Expected:* the `-br` (brief) form prints prefix lengths inline. Note that WSL2's
adapter is behind a NAT the hypervisor manages, so its addressing tells you about
WSL2's virtual network, not about `lx0r`'s.

---

## Hands-on

About 40 minutes on `lx0r`. Read-only throughout — nothing to undo.

### 1. Ten prefixes, by hand, on paper

No calculator, no subnet website. For each, write network address, broadcast
address, first and last usable host, and total usable hosts:

```
192.168.4.130/26
10.55.200.19/28
172.16.9.77/30
10.0.0.1/31
203.0.113.45/27
10.128.0.0/9
192.168.100.200/25
172.20.14.3/22
10.10.10.10/32
198.51.100.129/29
```

Two are deliberately awkward: `/22` moves the boundary into the third octet, and
`/9` moves it into the first. If those two took noticeably longer than the rest,
that is the finding — it means the four-step method is being applied to the
fourth octet by habit rather than to the interesting octet by rule.

Check yourself afterwards with:

```powershell
Get-NetIPAddress | Get-NetRoute -ErrorAction SilentlyContinue
```

or simply by verifying that your ranges tile without gaps or overlaps.

### 2. Find your own boundary

```powershell
Get-NetIPConfiguration
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength
```

From the output, compute by hand — before checking — your subnet's network
address, its broadcast address, and how many hosts it can hold. Then find one
address that is on your link and one that is not, differing by as little as
possible. On a /24 those will be one apart across a boundary; that pair is the
demonstration that the mask, not the address, decides.

### 3. Compress and expand IPv6

Compress:

```
2001:0db8:0000:0000:0abc:0000:0000:0001
fe80:0000:0000:0000:0204:61ff:fe9d:f156
2001:0db8:0000:0042:0000:0000:0000:0000
```

Expand:

```
2001:db8::1:0:0:1
::1
ff02::1:ff00:42
```

The first compression is the instructive one: there are **two** runs of zeros and
you may only replace one. Convention says replace the longer; if they are equal
length, replace the first. Write down which you chose and why — the point is that
one address has more than one legal textual form, which is exactly why any log
search matching v6 addresses as strings is unreliable.

The third expansion is a solicited-node multicast address. You do not need to
know what that means yet; note the form, because block 1c is going to explain it
and it will land better if you have already seen it.

### 4. Look at your v6 addresses even though "we don't run IPv6"

```powershell
Get-NetIPAddress -AddressFamily IPv6 | Format-Table InterfaceAlias, IPAddress, PrefixLength, SuffixOrigin, AddressState
```

*Expected:* every up interface has at least one `fe80::/64` link-local address,
present with no configuration anywhere. Count how many total v6 addresses this
machine holds right now. Write the number in the ledger — it is the number that
makes the point.

---

## Checkpoint

From memory, in [LEDGER.md](LEDGER.md).

1. MAC addresses are 48 bits — 281 trillion of them. Explain, without using the
   word "enough", why they cannot be used to build a global network, and name the
   specific property IP addresses have that fixes it.
2. State the one decision the mask makes, then trace what a host does
   *differently* on each branch — being precise about what goes in the IP header
   and what goes in the Ethernet header in the remote case.
3. Two hosts share a wire. One is configured /24, the other /25. Describe a
   concrete pair of addresses where they disagree about locality, what the
   observable symptom is, and why no device logs an error.
4. Give the reason /31 is valid and /30 is wasteful, then give the reason /64 is
   effectively mandatory in IPv6. One is about a header field being pointless;
   the other is about a *later* mechanism's requirement. Say which is which.
5. `172.16.34.100/26` and `172.16.34.200/26`: same subnet or different? Give the
   answer and the four steps that produced it, in under thirty seconds.
6. Why is `2001:db8::1:0:0:1` ambiguous to compress but unambiguous to expand,
   and what does that cost anyone searching logs for a v6 address?

---

## Prediction

Before running anything, write in the ledger:

**A. Locality.** From your own address and prefix length, pick three destinations:
one you are certain is on your link, one you are certain is not, and one just
across your subnet boundary. For each, predict:

- local or remote,
- the **next hop** the stack will use,
- the **interface** it will leave by.

Then check, without sending a packet:

```powershell
Find-NetRoute -RemoteIPAddress <each address> | Select-Object IPAddress, NextHop, InterfaceAlias, RouteMetric
```

*Expected:* for a local destination the next hop is `0.0.0.0` — which is the stack
saying "directly connected, no router involved." For a remote destination the next
hop is your gateway's address. **That `0.0.0.0` is the fork, printed.**

**B. Prefix arithmetic under time pressure.** Have someone (or a script) give you
five random address/prefix pairs. Predict the network address for each before
computing carefully. Grade for speed as well as correctness — the target is
under ten seconds each, out loud.

**C. [home PC] — the same question on the FortiGate.** Bring this to the box:

```
diagnose ip route lookup root 8.8.8.8
diagnose ip route lookup root <an address inside a directly connected subnet>
```

Predict, in writing, before you run it: which interface each resolves to, and
whether the connected one shows a gateway at all. *Expected:* the connected
lookup resolves to the interface with no gateway, the remote one resolves via a
next-hop address — the identical `0.0.0.0`-vs-gateway distinction Windows just
showed you, on a completely different vendor's stack, because it is not a vendor
behaviour.

If A comes back clean, block 1b is quick. If the boundary case in A missed, we
re-run the maths before touching plan design, because a plan is nothing but this
arithmetic applied at scale.
