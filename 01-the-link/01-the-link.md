# Lesson 01 — The link

> **What this answers:** What is a link, and what does a switch actually do?

---

## Start with one wire

Original Ethernet was literally that: a single coaxial cable with machines
tapped into it. Every machine's transmission reached every other machine,
because there was nothing in between to stop it.

That is the primitive. Everything since — switches, VLANs, Hyper-V virtual
switches — is an *optimisation* on top of that idea, not a replacement for it.

Hold onto that, because it explains the thing people get wrong later:
**a switch does not change the logical model.** A switched network still
behaves, semantically, like one shared wire. It just stops wasting capacity.

## What a MAC address actually is

Not an identity. **A filter.**

Every NIC on a shared medium receives every frame — physically, electrically,
it has no choice. What it does next is compare the destination MAC field to its
own. Match, or broadcast? Pass it up to the OS. Otherwise? Discard it,
silently, in hardware.

That is the entire mechanism. And it explains three things at once:

- **Promiscuous mode** is just "stop discarding." It is not a special power; it
  is the *absence* of the normal filter. That is why packet capture works.
- **MAC addresses are not security.** The filter is voluntary and lives on the
  receiving device.
- **A switch is an optimisation, not a redesign.** It reduces how many machines
  have to run that filter for a given frame. The semantics are unchanged.

## What a switch actually does — three rules

The complete algorithm. There is no fourth rule.

1. **Learn from the source.** A frame arrives on port 5 with source MAC
   `AA:BB:…` → record "`AA:BB:…` is reachable via port 5." A switch learns from
   *sources*, never from destinations.
2. **Forward on the destination.** Look up the destination MAC. Known? Send it
   out that one port only.
3. **Flood if unknown.** Destination not in the table? Send it out every port
   except the one it arrived on. Same for broadcast (`FF:FF:FF:FF:FF:FF`) and,
   with caveats, multicast.

That **learn-from-source / forward-on-destination** asymmetry is the whole
design, and it is elegant: a switch populates its table purely as a side effect
of carrying traffic. Nobody configures it. It needs no protocol.

Two consequences worth internalising now, because they come back later:

- **Flooding is the default, not the failure case.** An empty table is not
  broken — a switch with no knowledge behaves exactly like the original shared
  wire. It degrades gracefully to the primitive. Everything still works, just
  less efficiently and less privately.
- **Entries age out.** The table is a cache, not a database. Silent devices get
  forgotten and have to be flooded to again.

## The broadcast domain

Define it precisely, because Lesson 02 depends on it:

> **A broadcast domain is the set of devices that receive each other's
> broadcast frames — the set reachable without any forwarding decision above
> layer 2.**

Also called a segment, a LAN, a VLAN, a subnet's link. Same thing, different
context.

Here is the sentence to carry out of this lesson:

> **Inside a broadcast domain, there is no routing. Delivery is direct.**

A machine reaching another machine on its own link does not consult a routing
table, does not need a gateway, and does not require a router to exist anywhere
in the world. It puts the destination's MAC on a frame and the switch delivers
it. Done.

Which means routing is **not** a fundamental necessity of networking. Routing
exists only because we chose to break the world into many broadcast domains.
Lesson 02 is about why we made that choice.

---

## Hands-on

Roughly twenty minutes. Runs entirely on `lx0r` except where noted.

### 1. Look at a real forwarding table

On the Hyper-V switch, the port-to-MAC mapping is visible from the host:

```powershell
Get-VMNetworkAdapter -All | Format-Table VMName,Name,SwitchName,MacAddress,Status
```

*Expected:* one row per VM port, each with a MAC Hyper-V assigned. Note that
this is the switch's **authoritative** view (ports register their MAC), not a
learned table — a difference from physical switches worth remembering.

On a physical FortiGate operating a hardware switch interface — **home PC, not
this laptop**:

```
get system arp
diagnose netlink brctl name host <switch-name>
```

*Expected:* MAC-to-port mappings that nobody configured. They exist purely
because traffic flowed. That is rule 1 in action.

### 2. Watch your own NIC's filter

Capture on the Wi-Fi adapter in Wireshark. Note whether promiscuous mode is
enabled, then capture once with it off and once with it on.

The difference in what you see **is** the MAC filter, made visible. On a
switched network the difference will be smaller than you might expect — think
about why before reading Lesson 02.

### 3. Find a broadcast

In any capture, filter:

```
eth.dst == ff:ff:ff:ff:ff:ff
```

Look at what is actually using it — ARP, DHCP, mDNS, NetBIOS, LLDP. For each
one ask: *why does this protocol need to shout at everyone rather than address
someone specific?*

You will answer that properly in Lesson 04. Form an opinion now and write it in
`PROGRESS.md` under open questions — it is more useful to be wrong on record
than vague.

---

## Checkpoint

Answer from memory, in [PROGRESS.md](../PROGRESS.md), without scrolling back.
Any answer that comes out fuzzy means this lesson repeats.

1. A switch has an empty MAC table. Traffic still flows correctly. Explain
   why — and what is worse about it.
2. Why does a switch learn from source addresses rather than destination
   addresses? What would break if it tried the reverse?
3. Two machines on the same switch, same broadcast domain. Every router on
   earth is unplugged. Can they talk? Why?
4. A frame arrives with a destination MAC belonging to no device on the
   segment. Trace what happens.
5. In one sentence: why is a MAC address unsuitable for identifying a machine's
   *location* in a large network?

Question 5 is the setup for Lesson 02. Answer it from what you know now; it
gets tested rather than assumed.
