# Track 00 — The tap

**What this track answers:** where was this capture taken, and what had already
happened to the frame by the time the instrument saw it?

This track builds no model of how networks work. It is about how instruments
lie. It exists first because every hands-on exercise in every later track is
read through one of these instruments, and the most efficient way to learn a
wrong thing is to draw a correct-looking conclusion from an instrument you did
not understand.

It has no prerequisites, and Track 01's first exercise already needs it.

## The idea the track is built around

> A capture is not "what happened on the network." A capture is **what happened
> at one point in one stack, after everything upstream of that point had already
> acted on the packet — including the capture stack itself.**

Two halves to that. The first is *placement*: what is upstream of the tap. The
worked example is already in hand — the Hyper-V lab on `lx0r`, where a FortiGate
SNATs and then the Windows host's NAT SNATs again:

```
internet ◀ WiFi ◀ WinNAT ◀ [LabWAN] ─ port1 ┐
                                            │ FortiGate-VM
client VM ─ [Lab-Internal] ──────────── port2 ┘
```

Capture on the client, on the host's `LabWAN` vNIC, and on `Wi-Fi` for one
outbound flow, and the same conversation carries **three different source
addresses**. No capture is wrong. Any one of them, read alone, licenses a false
conclusion.

The diagram above is a **shape, not an inventory** — the smallest topology that
produces the problem. Lesson 00 asks you to draw your own path before capturing
anything, and works with any two taps that have something between them.

> The client is a separate VM rather than the Windows host, and that is not
> incidental. A host that is *both* the client and the firewall's upstream router
> has one routing table serving both roles, and any route pointing into the lab
> loops the traffic straight back. Build notes and the failure in full are in
> `fortinet-sdwan-lab/study-lab/`.

The second half is *the instrument itself*, and it is the part most people never
learn. A capture is taken at a specific position in a specific stack, and the
layers on either side of that position have already transformed the frame — or
never handed it over at all.

## Lessons

| # | Lesson | Answers |
|---|---|---|
| 00 | [Placement](00-placement.md) | What is upstream of the tap, and which claims a capture point does and does not license |
| 01 | [The instrument lies](01-the-instrument-lies.md) | Why the packets you see never crossed a wire in that form, and why the ones you don't see still did |
| 02 | [Records that are not packets](02-records-not-packets.md) | What each telemetry form preserves, what it destroys, and how many records one conversation becomes |

The three questions stack: *where was this taken* (00), *what did the instrument
do to it* (01), *what function was applied to produce this record* (02). All
three are answerable before you know anything about networking, which is why
they come first.

Lesson 01 is the one with no substitute. It covers, concretely:

- **Hardware offload.** Traffic offloaded to an NP/ASIC does not traverse the
  CPU path, so the on-box sniffer never sees it. A silent sniffer during a live
  transfer is a *conclusion about offload*, not a conclusion about traffic.
- **TSO / LRO / checksum offload.** A local capture sits above the NIC, so it
  shows segments far larger than the MTU that never existed on the wire, and
  checksums the NIC had not computed yet.
- **VLAN tag stripping.** The tag is lifted into per-packet metadata by the
  driver before the capture library sees the frame — a trunk capture that shows
  no 802.1Q header is not evidence the traffic was untagged.
- **Tap position is a choice, not a constant.** A capture at the NIC and a
  capture at a stack component are different instruments; only some of them can
  tell you *why* a packet was dropped rather than merely that it is absent.
- **Snaplen and drop counters.** The two numbers that say whether the capture
  itself is complete, and which nobody reads.

Lesson 02 carries the repo's timer theme into telemetry: an exporter's active
timeout splits one long connection into several records, which is why a
per-record byte threshold never trips on slow exfiltration. It also names
**clock synchronisation** once, as the precondition every correlation claim in
Track 04 and Track 08 quietly assumes.

## Reference — the same question on three boxes

Not a lesson. A table to keep open during every later track's hands-on, so an
exercise written for one box can be run on whichever box is in front of you. The
value is the *invariant*: three vendors, three syntaxes, one algorithm.

| Question | FortiOS | Windows | Linux |
|---|---|---|---|
| What routes exist? | `get router info routing-table all` | `Get-NetRoute` | `ip route show` |
| Where would *this* go? | `diagnose ip route lookup` | `Find-NetRoute -RemoteIPAddress x` | `ip route get x` |
| Who are my neighbours? | `get system arp` | `Get-NetNeighbor` | `ip neigh` |
| Show me the packets | `diagnose sniffer packet` | `pktmon`, Wireshark | `tcpdump` |

The middle row is the underused one. It answers "what would you actually do"
without sending a packet, which makes it the cheapest way to check a prediction
against reality — the loop this whole repo runs on.
