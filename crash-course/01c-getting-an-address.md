# Block 1c — Getting an address, and finding a neighbour

> **What this answers:** The plan exists on paper and the host knows nothing about
> it. How does a host get an address, and once it has one, how does it get the
> MAC address it still needs to build a single frame?

One sitting. Runs on `lx0r` and the Hyper-V lab. FortiGate sections flagged
**[home PC]**.

Depends on: block 0 (the header stack), 1a (the mask and its one decision), 1b
(the plan).

---

## Two problems, and they are not the same problem

Block 1a ended with a host that knows its address, its mask, and therefore
whether a destination is local or remote. In **both** branches it needs to build
an Ethernet frame, and an Ethernet frame needs a destination MAC — for the far
host if local, for the router if remote.

It does not have one. Nothing so far has produced a MAC address for anything.

And step back further: how did the host get its own address in the first place?
1b designed a plan. Nobody delivered it.

So there are two distinct questions, with two distinct mechanisms, and conflating
them is a common source of muddle:

| Question | v4 mechanism | v6 mechanism |
|---|---|---|
| **What is my address?** | DHCP | SLAAC, or DHCPv6, or both |
| **What MAC goes on this frame?** | ARP | ND (Neighbour Discovery) |

The second one first, because it is the more fundamental — and because the design
difference between ARP and ND is the clearest explanation available of what ARP
actually *is*.

## ARP: the crudest possible protocol, and why it works

The host knows `10.20.30.99` is on its link. It needs that host's MAC. There is no
directory and no server. So it does the only thing available on a shared medium:
**it shouts.**

```
Request  (broadcast, dst MAC ff:ff:ff:ff:ff:ff)
    "Who has 10.20.30.99? Tell 10.20.30.40, whose MAC is aa:bb:cc:dd:ee:ff"

Reply    (unicast, straight back to aa:bb:cc:dd:ee:ff)
    "10.20.30.99 is at 11:22:33:44:55:66"
```

Four things about this, each of which matters later:

**1. The request is a broadcast; the reply is a unicast.** The question has to
reach everyone because nobody knows who the answer belongs to. The answer knows
exactly who asked — the requester put its own address and MAC in the request — so
it goes direct. Every NIC on the segment interrupts its CPU for the question;
only one machine hears the answer.

**2. ARP is not carried in IP.** EtherType `0x0806`, not `0x0800`. Look at block
0's header stack and notice what that means: an ARP frame has *no IP header at
all*. It cannot be routed, it cannot cross a broadcast domain, and it has no TTL
because it does not need one. ARP lives strictly at layer 2 while being *about*
layer 3, which is why it is usually called "layer 2.5" and why it does not fit
the model cleanly.

**3. Everyone who hears it learns from it.** Not just the target. The request
contains the sender's IP and MAC, so every host on the segment can populate its
own cache for free. Efficient. Also the vulnerability: **the cache is populated by
unsolicited assertion, from anyone, with no verification of any kind.**

**4. The reply is believed.** There is no authentication, no challenge, no check
that the replier is the address's owner, and on many stacks no check that anyone
asked. That is not an implementation weakness, it is the protocol as specified —
ARP was designed for a cooperative local wire in 1982 and nothing about it has
changed.

### Gratuitous ARP

An ARP *request* for your own address, broadcast, that you do not expect an answer
to. It exists to make everyone else update their cache:

- **On boot or address change** — "I am here now, at this MAC."
- **On duplicate detection** — if anyone answers, someone else has your address.
- **On failover** — the surviving node in an HA pair sends gratuitous ARP so every
  device on the segment re-points to its MAC. This is the mechanism block 8's HA
  failover depends on, and it is the reason a failover that "should be instant"
  can be visibly slow when a switch or a host ignores gratuitous ARP.

Gratuitous ARP is also, unchanged, the mechanism of ARP spoofing. **Legitimate
failover and attack use the identical packet**, which is exactly why detecting one
without the other is hard.

### Proxy ARP

A router answers an ARP request for an address that is *not* its own, offering its
own MAC, because it knows how to reach the target. Hosts then send it frames for
an address it does not have.

FortiGate does this routinely for **VIPs** — a virtual IP on an interface's subnet
that is really an inbound NAT to something behind the box. The firewall ARPs for
it so the upstream device can find it. Block 6 covers what happens after the
frame arrives; the ARP behaviour is the part that makes it reachable at all, and
it is also the reason "which device owns that address?" is not always answerable
from the ARP table.

### The cache, and its timer

Every ARP result is cached, because broadcasting for every packet would be
absurd. **This is the first cache in the course.**

- Typical Windows and Linux reachable lifetime: on the order of tens of seconds,
  then a transition to a stale state, then re-validation on next use.
- FortiOS ARP timeout is configurable, commonly five minutes.
- Entries can be static; almost nobody does this outside specific hardening.

What goes wrong when the timer and reality disagree:

- **A device's MAC changes but its IP doesn't** — HA failover, NIC replacement, a
  VM moving hosts. Anyone with a live cache entry keeps sending frames to a MAC
  that is gone. They go to the wrong place, or nowhere. This resolves itself on
  expiry, which is why "it fixed itself after five minutes" is a diagnosis and not
  a mystery.
- **The cache is right and the switch's MAC table is wrong**, or vice versa. Two
  independent caches, two independent timers, and they are usually configured with
  different values. The classic asymmetry: an ARP entry that outlives the switch's
  MAC table entry means unicast frames get flooded, because the sender knows the
  MAC and the switch has forgotten which port it is on.

That second one is worth holding onto. **Two caches with different timers,
describing the same fact, is the general shape of most confusing network
behaviour.** You will meet this exact pattern again with sessions, with routes,
with DNS, and with IPsec SAs.

## ND: the same job, redesigned with the mistakes known

IPv6 replaced ARP with **Neighbour Discovery**, part of ICMPv6. It does the same
job, and every difference is a deliberate correction.

### It runs over IP

ND is ICMPv6 — so it has an IPv6 header, and therefore a hop limit. ND messages
are sent with hop limit **255** and receivers **discard any ND message whose hop
limit is not 255**. Since every router decrements, a hop limit of 255 on arrival
proves the sender was on this link, because no forwarded packet can arrive at 255.

That is elegant enough to be worth pausing on: a protocol that must be
link-local uses a field designed for loop prevention to *prove* locality. ARP got
this property structurally, by not being routable. ND got it back deliberately,
after choosing to run over IP.

### It uses multicast, not broadcast

Instead of shouting at everyone, ND asks a **solicited-node multicast** group:
`ff02::1:ff` followed by the **last 24 bits of the target address**.

```
    target      2001:db8:20:30::abc:1234
    last 24 bits                 bc:1234
    group       ff02::1:ffbc:1234
```

A host joins the solicited-node group for each of its own addresses. So a
neighbour solicitation reaches only the hosts whose address shares those low 24
bits — in practice, one host — and the group maps down to an Ethernet multicast
MAC (`33:33:` + the last 32 bits of the multicast address) that NICs filter **in
hardware**.

The result: on a segment of 500 hosts, an ARP request interrupts 500 CPUs and an
equivalent ND solicitation interrupts roughly one. That is the fix for the cost
of broadcast, and it is the same idea block 2 applies to broadcast domains at a
larger scale.

### It tracks reachability — ARP never did

This is the difference that matters operationally and nobody teaches.

ARP caches an answer and holds it until a timer expires. It has **no concept of
whether the neighbour is still there**. ND has **NUD** — Neighbour Unreachability
Detection — a real state machine per entry:

| State | Meaning |
|---|---|
| `INCOMPLETE` | Solicitation sent, no answer yet |
| `REACHABLE` | Confirmed recently — traffic flows, no probing |
| `STALE` | The confirmation aged out; still usable, will be checked on next use |
| `DELAY` | In use after being stale; giving upper layers a moment to confirm |
| `PROBE` | Actively soliciting to re-confirm |

Crucially, ND takes **hints from upper layers**. If TCP is receiving ACKs, the
neighbour is demonstrably reachable and ND does not probe. So v6 gets active
failure detection nearly free, while v4 waits for a timer regardless of evidence.

Practical consequence: when a MAC changes underneath a live conversation, IPv6
notices and recovers; IPv4 waits out its ARP timer. **This is a case where the
dual-stack host behaves visibly better on v6**, and it is worth knowing before
someone blames v6 for an inconsistency it actually fixed.

### DAD — duplicate address detection

Before using any address, a v6 host sends a neighbour solicitation *for that
address* from the unspecified source `::`. An answer means someone else has it, and
the host refuses to use it. Every address, every time, mandatory.

v4's equivalent — gratuitous ARP on boot — is optional, inconsistently
implemented, and frequently ignored. Duplicate v4 addresses are a real and common
outage cause. Duplicate v6 addresses essentially do not happen. That is DAD.

### Router discovery is part of ND, and v4 has no equivalent

ND also carries **Router Solicitation** and **Router Advertisement**. A host
multicasts an RS to `ff02::2` (all routers); routers reply with an RA — and
routers also send RAs unsolicited, periodically.

An RA carries the prefix, the default gateway, the MTU, flags about how to get an
address, and often DNS servers. **This is the piece v4 genuinely lacks**: v4 has no
native way to discover a router, which is why the gateway arrives via DHCP option
3 as a *lease attribute* rather than from the router itself.

Notice the architectural consequence, because it is the whole shape of the v6
security problem in a v4-managed network:

> **In IPv4, the gateway is announced by a server you control. In IPv6, the
> gateway is announced by anything on the wire that sends an RA.**

A host that receives an RA configures itself and starts routing through the
sender. No server, no lease, no administrator. Which is why an unmanaged v6
stack on a managed v4 network is a path that exists, works, and is watched by
nobody — the substance behind "we don't run IPv6 is a blind spot, not a defence."
RA Guard on the switching layer is the control, and it is block 2's material.

## Getting an address: DHCP

The host has no address at all. It cannot be addressed, so it cannot be asked. So
DHCP starts from broadcast too.

**DORA**, four messages, UDP, client port 68, server port 67:

| | Message | From → to | Carries |
|---|---|---|---|
| **D** | Discover | `0.0.0.0` → `255.255.255.255`, broadcast | "Anyone there? Here's my MAC" |
| **O** | Offer | server → client | A proposed address, mask, gateway, DNS, lease time |
| **R** | Request | client → **broadcast** | "I accept *that* one, from *that* server" |
| **A** | Ack | server → client | Confirmed; lease starts |

Two details that look redundant and are not:

**Why is the Request broadcast?** Because there may have been several Offers. The
Request names the chosen server, and broadcasting it tells the *other* servers
they were not chosen so they can release their reservations. The apparent
redundancy is how multiple servers stay consistent without talking to each other.

**Why four messages instead of two?** Offer/Request separates *proposing* from
*committing*. Without it, two servers offering simultaneously would both commit,
and the client would have two addresses with no way to decline one.

### Relay — the consequence of having more than one broadcast domain

DHCP starts with a broadcast. Broadcasts do not cross a router. So a DHCP server
would need to be on every segment.

The fix is a **relay agent** — usually the router or firewall on the segment. It
receives the broadcast, rewrites it as a unicast to the real server, **and inserts
the address of the interface it received it on** (the `giaddr` field). That last
part is the entire trick: the server has no idea which segment the client is on,
so the relay tells it, and the server uses `giaddr` to pick the right scope.

This is the first time a mechanism from block 1b's plan becomes visible in a
protocol field. **The relay address is how the addressing plan gets executed at
runtime** — the segment's identity, in a header, deciding which pool to draw from.

### The lease is a timer, and it is the identity timer

An address is **borrowed**, for a stated time.

| Timer | Default | What happens |
|---|---|---|
| **T1** | 50% of lease | Client tries to renew, unicast, to the original server |
| **T2** | 87.5% of lease | Renewal failed; client broadcasts to any server (rebinding) |
| **Lease** | 100% | Client must stop using the address |

The consequence that matters most in your job:

> **An IP address identifies a machine only for the duration of a lease, and only
> if you have the record of that lease.**

An alert says `10.16.10.53` at 14:02. Which machine? The DHCP lease active at
14:02 — not the one active now. Miss the lease log and you have an address, not a
host. Short leases on guest wireless make this dramatically worse; that is the
tradeoff for reclaiming addresses quickly, and it is a decision someone made
without necessarily knowing what it cost the investigation side.

This is where the crash-course lane and the nine tracks meet again. Same
mechanism, opposite direction: they read the ambiguity out of an alert, you are
choosing the lease time that creates it.

### DHCP options, and the ones that matter

The address is the smallest thing DHCP delivers. Options carry:

| Option | Content | Why it matters |
|---|---|---|
| 1 | Subnet mask | Block 1a's entire fork, handed over |
| 3 | Router | The default gateway — v4's substitute for router discovery |
| 6 | DNS servers | Block 5 |
| 51 | Lease time | The timer above |
| 82 | Relay agent information | Which switch port, on some deployments |
| 121 | Classless static routes | Additional routes pushed to clients |

Option 121 is worth flagging: a DHCP server can install **routes** on your hosts.
That is a routing decision made by a service most people classify as addressing —
and a thing worth checking before believing a host's route table reflects anyone's
design.

## Getting an address in IPv6: two mechanisms, often both

v6 has DHCPv6 (UDP 546/547, multicast, similar in spirit). It also has **SLAAC**,
which v4 has no equivalent of.

**SLAAC**: the host hears an RA carrying a /64 prefix, and constructs its own
address by appending a 64-bit interface identifier it generates itself. No server.
No lease. No record anywhere of who took what.

Which mechanism a host uses is signalled by two flags in the RA:

| Flag | Meaning |
|---|---|
| **M** (managed) | Use DHCPv6 for the address |
| **O** (other) | Use DHCPv6 for other config (DNS etc.) but SLAAC for the address |
| neither | Pure SLAAC |

And hosts do not necessarily pick one. A typical Windows machine ends up with:

- `fe80::…` link-local — always, unconditional, from block 1a;
- a SLAAC global address from the RA prefix;
- one or more **temporary (privacy) addresses** — RFC 8981 — with a random
  interface ID that **rotates on a timer**, typically daily, used as the *source*
  for outbound connections;
- possibly a DHCPv6 address as well;
- plus every solicited-node multicast group for each of those.

Hence "one laptop, six addresses in a week." Not a misconfiguration. The design,
working as specified.

And which address gets used as the source for a given connection is decided by
**RFC 6724** — a rules table, evaluated in order, weighing scope match, preferred
lifetime, precedence and prefix-length match. It is why the source address of an
outbound v6 connection is not a thing you configured. Block 3 covers the
destination-selection half; the piece to hold now is simply that **it is a rules
engine, not a setting.**

## Every cache and timer so far

The theme, consolidated at the end of block 1 because this is where it becomes
real:

| Cache | Lives on | Typical timer | Symptom when it disagrees with reality |
|---|---|---|---|
| Switch MAC table | Switch | ~5 min aging | Unicast gets flooded to every port |
| **ARP cache** | Every host and router | tens of seconds to ~5 min | Frames sent to a MAC that has moved or gone |
| **ND neighbour cache** | Every v6 host | State machine, not a flat timer | Recovers faster than ARP; states visible |
| **DHCP lease** | Server *and* client | Hours to days; T1 at 50%, T2 at 87.5% | An IP maps to the wrong machine in an investigation |
| **v6 temporary address** | Host | ~1 day rotation | One machine, many source addresses over a week |

Read the last column. **Every one of those is a case of two parties holding
different beliefs about the same fact, separated by a timer.** That is the shape
of the theme, and it does not change for the rest of the course — sessions,
routes, DNS answers and IPsec SAs are all the same pattern with bigger numbers.

---

## Artifact

### Windows

```powershell
Get-NetNeighbor -AddressFamily IPv4 | Format-Table IPAddress, LinkLayerAddress, State, InterfaceAlias
Get-NetNeighbor -AddressFamily IPv6 | Format-Table IPAddress, LinkLayerAddress, State, InterfaceAlias
```

*Expected:* a **State** column showing `Reachable`, `Stale`, `Permanent` and
possibly `Probe` or `Incomplete`. Note that Windows shows this for v4 as well —
the ARP cache is presented through the same NUD-shaped interface even though the
v4 protocol has no such state machine underneath. **The tool's model is not the
protocol's model**, and that distinction is worth carrying.

The lease side:

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object IPAddress, PrefixOrigin, SuffixOrigin
ipconfig /all
```

*Expected:* `ipconfig /all` prints "Lease Obtained" and "Lease Expires" for a DHCP
address. Compute the lease length from those two, then compute where T1 and T2
fall. That arithmetic is the timer, made concrete.

### Linux (WSL2)

```bash
ip neigh
ip -6 neigh
```

*Expected:* the state is printed as a bare word at the end of each line —
`REACHABLE`, `STALE`, `DELAY`, `PROBE`. This is the cleanest available view of
NUD, and it is worth watching change under traffic.

### FortiOS **[home PC]**

```
get system arp
diagnose ip arp list
diagnose ipv6 neighbor-cache list
execute dhcp lease-list
get system interface physical
```

*Expected:* `get system arp` gives address, age, MAC and interface — with **age**
being the visible timer. `execute dhcp lease-list` shows the FortiGate's own DHCP
leases with expiry, which is the identity record from the section above.

If `diagnose ipv6 neighbor-cache list` is not the command on your firmware
version, `diagnose ipv6 ?` will list what is — the subcommand names have moved
between major versions and I am not going to state a version I have not checked.

The DHCP relay configuration, which is the plan-execution mechanism made visible:

```
show system interface <internal-interface>
```

*Expected:* if relay is in use, `set dhcp-relay-service enable` and
`set dhcp-relay-ip` appear on the interface. The interface's own IP becomes the
`giaddr` the server uses to pick a scope — which means **changing an interface
address silently changes which DHCP pool that segment draws from.** That is a
non-obvious coupling and it is the kind of thing this course exists to make
obvious.

---

## Hands-on

About 50 minutes. Two sections change state and **both ship their undo first.**

### 1. Watch ARP happen from empty

Undo first — nothing here is destructive, but the cache flush needs admin and
briefly interrupts traffic to already-known neighbours while they are re-resolved.
There is no undo needed; the cache rebuilds itself from traffic. That is the
point of the exercise.

Start a capture in Wireshark with filter `arp`, then:

```powershell
Get-NetNeighbor -AddressFamily IPv4 | Format-Table IPAddress, LinkLayerAddress, State
Remove-NetNeighbor -AddressFamily IPv4 -Confirm:$false
ping -n 1 <your gateway>
Get-NetNeighbor -AddressFamily IPv4 | Format-Table IPAddress, LinkLayerAddress, State
```

*Expected:* the neighbour list empties, then repopulates on first use. In the
capture: one broadcast request, one unicast reply. Confirm both by looking at the
Ethernet destination MAC of each — `ff:ff:ff:ff:ff:ff` on the request, a specific
MAC on the reply.

Then leave it alone for a few minutes and re-run `Get-NetNeighbor`. *Expected:*
the state transitions from `Reachable` to `Stale` without any packet loss and
without anything breaking. Watching a cache age out while everything still works
is the intuition this block is for.

### 2. Watch ND happen, and count the difference

Same capture, filter `icmpv6.type == 135 || icmpv6.type == 136` (solicitation and
advertisement), then:

```powershell
Get-NetNeighbor -AddressFamily IPv6 | Format-Table IPAddress, LinkLayerAddress, State
ping -6 -n 1 <a link-local or global v6 neighbour, or your gateway's fe80:: address>
```

For a link-local destination you will need the zone index, e.g.
`ping -6 fe80::1%12` where 12 is the interface index from `Get-NetIPInterface`.

*Expected:* a neighbour solicitation sent to a `ff02::1:ff…` address, not to a
broadcast. Compare the last 24 bits of that multicast address with the last 24
bits of the target — they match. Seeing that match once makes solicited-node
multicast permanent knowledge.

If there is no v6 neighbour to reach, do the exercise against `ff02::1` (all
nodes) with:

```powershell
ping -6 ff02::1%<interface index>
```

*Expected:* replies from every v6-capable device on your link — including ones
that would swear they do not run IPv6. Count them. Record the count in the ledger
next to the address count from block 1a.

### 3. Watch DORA end to end — Hyper-V lab

Best done on a lab VM whose lease you can safely release. **Undo: `ipconfig
/renew` restores the address; if the VM loses connectivity, it regains it at the
next renew or on adapter restart.** Do not do this on the machine you are reading
this on if it is remote.

On the lab VM, with a capture running on the host or in the guest, filter `bootp`
or `dhcp`:

```powershell
ipconfig /all          # record current lease obtained/expires first
ipconfig /release
ipconfig /renew
```

*Expected:* four packets. Identify each of D, O, R and A, and for each one record
the source IP, destination IP, source MAC and destination MAC. The Discover's
source of `0.0.0.0` is worth a moment — a host with no address still has to put
*something* in the header.

Then in the Ack, expand the options and find: mask (1), router (3), DNS (6), lease
time (51). Every one of those is a decision from blocks 1a and 1b being handed to
a host that had no idea about any of it.

### 4. Find the second gateway nobody configured

Filter a capture for `icmpv6.type == 134` (router advertisement) and leave it for
a few minutes on your ordinary network.

*Expected:* either RAs from your legitimate router, or nothing, or — the
interesting case — RAs from something you did not expect. If any appear, note the
source `fe80::` address and its MAC, and look up the OUI.

Whatever the result, write it in the ledger. **"No RAs observed on this segment"
is a real finding and it is worth recording with a date**, because it is the
baseline that makes a future RA anomalous.

---

## Checkpoint

From memory, in [LEDGER.md](LEDGER.md).

1. An ARP request is broadcast and its reply is unicast. Give the reason for each,
   from what each party knows at that moment.
2. ARP has no IP header. Name two consequences of that fact — one about where ARP
   can go, one about what it doesn't need.
3. ND messages are sent with hop limit 255 and discarded if they arrive with
   anything else. Explain what that check proves and why it was necessary for ND
   but not for ARP.
4. Derive the solicited-node multicast address for `2001:db8:10:20::5eaf:c0de`,
   and say how many hosts on a 500-host segment process the resulting frame
   compared to an ARP request.
5. Why is the DHCP Request broadcast rather than unicast to the chosen server?
   What breaks if it is unicast?
6. A DHCP relay inserts `giaddr`. What is it for, and what silently changes if
   somebody re-addresses the relaying interface?
7. State the identity consequence of a lease timer in one sentence, then say what
   a short guest-wireless lease trades away and what it buys.
8. Name every cache introduced in block 1, its timer, and the specific symptom
   when it disagrees with reality. Five of them.

---

## Prediction

**A. Neighbour states, before you look.** Without running anything, predict:

- how many entries `Get-NetNeighbor -AddressFamily IPv4` will return,
- how many of them are `Reachable` versus `Stale`,
- the same two numbers for `-AddressFamily IPv6`,

then run both. *Expected:* far more v6 entries than v4, and mostly `Stale` in
both — because `Stale` is the resting state of anything not currently being
talked to, not a fault. If you predicted `Reachable` as the normal state, that is
the correction worth having, and it is the same misreading that makes people
chase healthy neighbour tables during outages.

**B. Cache repopulation.** Before running hands-on 1, predict exactly how many ARP
frames the capture will contain between the flush and the first successful ping,
and their direction. Then count them. If the count is higher than predicted,
work out what else on the machine sent traffic — that difference is a real
measurement of how chatty an idle Windows host is.

**C. [home PC] — the age column.** Bring this to the FortiGate:

```
get system arp
```

Predict, before running: the number of entries, and roughly what the largest
`age` value will be. Then run it twice, sixty seconds apart, and predict which
entries will have disappeared before checking. *Expected:* entries for devices
that are talking stay young; entries for devices that have gone quiet grow old and
eventually vanish. **Watching a specific row's age climb and then disappear is the
timer theme, observed rather than described.**

**D. [home PC] — leases against the plan.**

```
execute dhcp lease-list
```

Predict the pool range before running it, from the addressing plan alone. If your
prediction of the pool is right but the *lease count* surprises you, that gap is
worth noting — it is either devices you did not know were there or leases
outliving devices that have gone, and both are findings.

---

## End of block 1

Blocks 1a, 1b and 1c together should have closed the gap from "an address is a
number on an interface" to "an address is a position in a designed hierarchy,
delivered by a protocol, resolved by a cache, and true only until a timer says
otherwise."

Block 2 starts from a specific cost this block created. ARP broadcasts to every
host on the segment; DHCP Discover broadcasts to every host on the segment; every
one of those interrupts a CPU. **What does that cost, exactly, as the segment
grows — and what is the fix?** That is where VLANs come from, and they will be
introduced as the answer to a number you measure rather than as a feature.
