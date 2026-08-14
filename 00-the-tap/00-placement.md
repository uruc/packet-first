# Lesson 00 — Placement: what is upstream of the tap

> **What this answers:** what had already happened to this packet before the
> capture point saw it — and which of my conclusions does that invalidate?

---

## Borrowed on trust

This is the first lesson in the repo, so a few words are used before they are
earned. Each is taken on trust and lands where noted. If any of them feels
shaky, that is information — say so at the checkpoint rather than reading past.

- **Frame, packet** — a unit of data with headers wrapped around it. Track 01.
- **MAC address, VLAN tag** — layer-2 identifiers. Track 01.
- **NAT** — a box rewriting an address in transit. Track 02.
- **Router hop** — a box that forwards between segments. Track 03.

You need none of them in depth here. You need only to accept that each names a
*transformation* something can perform on a packet in flight.

## The claim

> A capture is not "what happened on the network." A capture is **what happened
> at one point, after everything upstream of that point had already acted on the
> packet.**

That is not a caution. It is the definition. There is no such thing as a capture
of "the network" — every capture is a point observation, and a point observation
on a path of transformations is only interpretable if you know which
transformations sit between the origin and your point.

## What "upstream" actually means

Here is the part that trips people, and it is worth getting exactly right:

**A tap has two upstreams, one per direction.** For traffic leaving your host,
upstream is the host's own stack — almost nothing has happened yet. For traffic
arriving at your host, upstream is everything from the sender to you — every
rewrite, every hop, every middlebox. The same capture file therefore contains
two populations of packets with completely different evidentiary value, and
reading it as one population is the most common way to get this wrong.

So "what is upstream of the tap?" is really two questions, and both must be
answered before the file is opened.

## The transformation classes

You do not need to know how any of these work yet. You need to know that they
exist, because each one destroys or rewrites something you might have been
planning to conclude from.

| Class | What it does to the packet | What you can no longer trust |
|---|---|---|
| **Address rewriting** | Replaces a source or destination address | Either address field, as an identifier of a machine |
| **Header replacement** | Swaps the layer-2 header at every router hop | The MAC address, as an identifier of anything but the last hop |
| **Encapsulation** | Wraps the packet in an outer header | The visible addresses and ports are the *tunnel's*, not the conversation's |
| **Tag insertion/removal** | Adds or strips a VLAN tag | The presence or absence of the tag in your file |
| **Dropping** | The packet does not arrive | Absence — which is not evidence of non-transmission |

The last row is the one that costs the most. A packet missing from your capture
was either never sent, sent and dropped upstream, sent and dropped by your
capture stack, or sent and delivered somewhere your tap cannot see. Those are
four completely different incidents and the capture alone cannot separate them.

## The worked case — a path with two rewrites

Take a Hyper-V topology with a firewall VM between an internal switch and the
host's own uplink. It contains a double rewrite:

```
internet ◀ WiFi ◀ WinNAT ◀ [LabWAN] ─ port1 ┐
                                            │ firewall VM
client VM ─ [Lab-Internal] ──────────── port2 ┘
```

> **This is a shape, not an inventory.** It is drawn this way because it is the
> smallest topology that produces the problem. Do not assume it matches what is
> on your machine — the hands-on below asks you to draw *your own* path first,
> and if you don't currently have a firewall VM in line, the same lesson works
> with a single rewrite (the host's own NAT to the internet) or with any two
> taps that have anything at all between them.

Traffic from the client to the internet crosses two address rewrites: the
firewall VM's, then WinNAT's. There are three places to tap, and each licenses a
different set of claims:

- **The client's own NIC** — before both rewrites. The source address here is
  the client's real address. This tap can tell you *who originated the traffic*
  and cannot tell you *what the internet saw*.
- **The host's `LabWAN` vNIC** — between the two rewrites. Neither the original
  source nor the final one.
- **`Wi-Fi`** — after both. This tap can tell you what left the machine and
  cannot tell you who inside originated it.

Capture the same flow at the first and last and you get two different source
addresses for one packet. **Neither capture is wrong.** Either one, read alone,
licenses a false conclusion — and the false conclusion looks exactly like a true
one, because a capture never announces what it is missing.

Note what the middle tap is *not*: the client is a separate VM, not the Windows
host. If the host were the client, it would be both the origin of the traffic
and the firewall's upstream router — one routing table serving two roles — and
any route sending its traffic into the firewall would loop it straight back. The
lesson here is about placement, and that is a placement failure in the topology
itself rather than in the tap.

## The same tap, different epistemics per stack

This is where IPv4 and IPv6 stop being interchangeable, and it matters from the
first capture you take.

The reason the source address is untrustworthy above is address rewriting. IPv6
was designed without the address shortage that motivates it, and a typical IPv6
path performs **no address rewriting at all** — the source address in an IPv6
packet at the far end of the path is usually the source address the originating
host actually assigned itself.

So a single capture file can contain both:

- IPv4 packets whose source address tells you which *NAT device* they last
  crossed, and
- IPv6 packets whose source address tells you which *host* sent them.

Two protocols, one tap, and the identical field means something different in
each. That is not a footnote about v6 — it is the clearest available proof that
"what does this field mean" is a question about the *path*, not about the field.

The complication that cuts the other way — a host holding several IPv6 addresses
at once and rotating them — is Track 02. Here, only the path property matters.

## The rule this produces

Before opening a capture file, write down, in this order:

1. **Where is the tap**, named as an interface on a named machine.
2. **What is upstream of it, outbound.** Usually short.
3. **What is upstream of it, inbound.** Usually long, and often partly unknown.
4. **Which fields are therefore untrustworthy**, per direction.

If you cannot complete step 3 even approximately, you can still read the
capture — you just cannot make claims that depend on the inbound path. Say which
those are, out loud, before you start. The failure mode is never "I didn't know";
it is "I forgot that I didn't know."

---

## Artifact

Where this shows up in something you would actually be handed:

**In a capture** — the source address field. Same packet, two taps, two values.

**In a FortiGate traffic log** — a single event carries `srcip` *and* the
translated pair `transip` / `transport`, because the device knows it performed a
rewrite and records both sides of it. (Confirm the exact field names against
your FortiOS version's log reference — they have moved between releases, and
this repo does not assert what it has not checked on your box.) That pairing
exists precisely because one address field cannot answer both "who sent it" and
"what did the far end see." A capture gives you one of those two values and does
not tell you which one you got.

**In an endpoint record** — Defender's `DeviceNetworkEvents` carries `LocalIP`,
the endpoint's own pre-rewrite source, because the endpoint sits upstream of
every rewrite on the path. That is the same fact from the other end: the tap
position determines the field's meaning, and the endpoint is simply a tap at
position zero.

The general form: **when two sources disagree about an address, the first
hypothesis is not that one is wrong. It is that they are taps at different
points, and both are right about different questions.**

## Caches and timers introduced

**None.** This lesson introduces no cache and no expiry, and that is worth
noticing rather than skipping — it is the only lesson in the track that doesn't.
Placement is a static property of where you stood. Lessons 01 and 02 both
introduce state that can be stale, and the contrast is the point.

---

## Hands-on

About thirty minutes. Runs entirely on `lx0r`.

**Nothing in this lesson changes configuration.** Packet capture is read-only;
there is no undo step because there is nothing to undo. That will not be true in
Lesson 01, which is why it is stated explicitly here.

### 1. Draw it before you capture it

Before starting Wireshark, write the path out in `PROGRESS.md` under open
questions — tap point, outbound upstream, inbound upstream, untrusted fields.
Do this from what you know about your own lab, not from the diagram above.

This is the exercise. The captures below only check it.

### 2. Enumerate your actual tap points

```powershell
Get-NetAdapter | Format-Table Name,InterfaceDescription,Status,LinkSpeed
Get-VMSwitch | Format-Table Name,SwitchType,NetAdapterInterfaceDescription
```

*Expected:* the physical Wi-Fi adapter, plus a `vEthernet (...)` adapter for
each Hyper-V switch the host itself has a presence on. Note that a Hyper-V
switch with no host adapter is a segment you **cannot tap from the host at
all** — an invisible tap point is worth knowing about before you need it.

### 3. Predict, then check

Pick one destination and one ping. Before running anything, write down the
source address you expect to see at the host's `LabWAN` vNIC and at `Wi-Fi`.

Then capture on both simultaneously — two Wireshark instances, or select both
interfaces in one capture — and generate the traffic **from the client**, not
from the host. Traffic the host originates has not crossed the firewall and will
not show the first rewrite.

*Expected:* the same exchange appears in both, with different source addresses,
because a rewrite sits between them. The `Wi-Fi` view should show the address
your Wi-Fi adapter holds on your home network; the `LabWAN` view should show the
firewall's own WAN-side address — **not** the client's, because the firewall has
already translated it. The client's real address is only visible at a third tap,
on the client itself.

The number that matters is not the addresses. It is **whether your written
prediction matched.** A wrong prediction here is the most valuable outcome
available — record it.

### 4. Find the two populations

In one capture file, apply a filter that separates traffic your host sent from
traffic it received. In Wireshark:

```
eth.src == <your adapter's MAC>
```

Then invert it. Look at how differently you would have to reason about each
half. Ask, for the inbound half only: *how many devices could have modified this
before I saw it, and do I know what any of them are?*

### 5. Both stacks, one file

```
ip
```
then
```
ipv6
```

*Expected:* on a normal home network, both are present — mDNS, ND, and often
real traffic. Look at the source addresses in each set and state, for each,
whether it identifies a **host** or a **rewriting device**.

If you see no IPv6 at all, that is a finding about your capture point or your
network, not about the world. Note which you think it is.

---

## Checkpoint

Answer from memory, in [PROGRESS.md](../PROGRESS.md), without scrolling back.
Any answer that comes out fuzzy means this lesson repeats.

1. Why does a single capture file contain two populations of packets with
   different evidentiary value, and what determines which population a given
   packet belongs to?
2. A packet you expected is not in the capture. You are asked to determine which
   of the possible causes it was, using only that capture. Explain why you
   cannot, and say what additional evidence would separate them.
3. Two capture points on the same path show different source addresses for the
   same packet. Which one is wrong, and why is that the wrong question?
4. In the same capture file, the source address of an IPv4 packet and the source
   address of an IPv6 packet do not answer the same question. Explain the
   difference and what causes it.
5. You are handed a capture file with no note saying where it was taken. State
   precisely what you can still conclude from it and what you cannot.

Question 5 is the one this whole track exists for. Answer it now, badly if
necessary; you will answer it again at the end of Track 08 and the difference is
the measurement.
