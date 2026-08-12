# Track 00 — Seeing the network

**What this track answers:** how do you look at a network and trust what you
see?

This track exists because every hands-on exercise in every later track depends
on it, and because the most common way to learn a wrong thing is to draw a
correct conclusion from a capture taken in the wrong place.

## Why it's Track 00 and not Track 08

Observation isn't the last step of learning networking, it's the instrument you
learn with. Putting it at the end would mean doing every earlier exercise
half-blind.

It's numbered 00 rather than 01 because it isn't conceptual scaffolding — it
builds no model of how networks work. It's a toolkit. Run Lessons 00 and 01
before Track 01's hands-on; Lesson 02 pairs naturally with Track 05, and can
wait.

| # | Lesson | Answers |
|---|---|---|
| 00 | Where you stand determines what you can conclude | Capture placement, and the claims a capture point does and does not license |
| 01 | The toolkit, one question at a time | The same question asked on FortiOS, Windows and Linux — and the invariant that appears |
| 02 | Diagnostic method | Forming a hypothesis that can be killed, and killing it cheaply |

## The idea Lesson 00 is built around

A capture is not "what happened on the network." A capture is **what happened
at one point, after everything upstream of that point had already acted on the
packet.**

The worked example is already in hand — the Hyper-V lab topology, where a
FortiGate SNATs and then WinNAT SNATs again:

```
internet ◀ WiFi ◀ WinNAT ◀ [Default Switch] ─ port1 ┐
                                                    │ FortiGate-VM
host ── vEthernet (LabInternal) ─ [LabInternal] ─ port2 ┘
```

Capture on `vEthernet (LabInternal)` and on `Wi-Fi` for the same ping and you
get two different source addresses for one packet. Neither capture is wrong.
Either one, read alone, licenses a false conclusion.

That is the whole lesson, and it generalises to every troubleshooting session
that follows: **before reading a capture, state what is upstream of the capture
point.** If you can't, you can't safely conclude anything from it.

## The cross-platform table Lesson 01 builds on

The same question, asked on three boxes. Running these side by side is what
makes the *algorithm* visible underneath the vendor syntax.

| Question | FortiOS | Windows | Linux |
|---|---|---|---|
| What routes exist? | `get router info routing-table all` | `Get-NetRoute` | `ip route show` |
| Where would *this* go? | `diagnose ip route lookup` | `Find-NetRoute -RemoteIPAddress x` | `ip route get x` |
| Who are my neighbours? | `get system arp` | `Get-NetNeighbor` | `ip neigh` |
| Show me the packets | `diagnose sniffer packet` | `pktmon`, Wireshark | `tcpdump` |

The middle row is the underused one. It answers "what would you actually do"
without sending a packet, and it is the fastest way to check a prediction
against reality — which is the loop this whole repo runs on.
