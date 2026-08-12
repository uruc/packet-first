# network-study

Networking fundamentals rebuilt from the ground up, aimed at one outcome: a
packet capture, a firewall log, a proxy decision, or a detection that fired
makes sense **at the mechanism level** rather than by pattern-match.

Started 2026-08-12. Re-aimed 2026-08-12 after an adversarial review of the
original track map.

## Why this exists

The gap is not vocabulary and not configuration. It is that every artifact a
security engineer reads — a pcap, a traffic log, a proxy verdict, a
`DeviceNetworkEvents` row — is a **lossy projection of a packet, taken at one
point, after everything upstream had already acted on it.** Read one without
knowing what made it and you are pattern-matching. Read one knowing exactly
which stage wrote which field, and which timer could make it lie, and you are
doing the job.

Fortinet is where this fluency gets spent. It is not the reason for it. When a
question is "what does FortiOS do here," the answer belongs in
`fortinet-sdwan-lab/`; this repo answers "what is the mechanism, and where does
it surface in evidence."

## The through-line

The tracks build the packet, then the things that rewrite it, then the machine
that observes and records it — and end at the artifact you started with.

```
00  the tap            what the capture point already did to the packet
01–04  the packet      link → address → path → session
05–06  the rewrites    names, encryption, encapsulation
07  the machine        the ordered decision that emits a log line
08  the capstone       four artifacts of one incident, reconciled
```

Track 00 is deliberately small and deliberately first: it teaches nothing about
how networks work, only about how instruments lie. Track 08 is where the
premise above stops being a claim you are asked to accept and becomes a thing
you are graded on.

## The tracks

| # | Track | Why it exists |
|---|---|---|
| 00 | [The tap](00-the-tap/) | Every exercise in every later track is read through an instrument; an instrument you don't understand teaches wrong models confidently |
| 01 | [The link, and what it leaks](01-the-link/) | L2 is where identity is strongest and shortest-lived, and where half of "impossible" results come from |
| 02 | Addresses, identity, and attribution | An alert is an IP and a timestamp; turning that into a machine and a person is the most common thing you do and the least understood |
| 03 | Paths, and why your sensor saw half | Traffic reaches sensors by accident of routing; asymmetry and placement decide what detection is even possible |
| 04 | Sessions | A firewall log line is a session record, not a packet record — you cannot read one without knowing what built it |
| 05 | Names, and what resolves without asking | The richest telemetry that exists, the most abused covert channel, and the one most easily made invisible |
| 06 | Opaque traffic | Most traffic is encrypted or encapsulated; the skill is knowing exactly what you still know, and what breaks when you decide to look |
| 07 | The pipeline and the log it writes | The ordered decision where everything above converges into an allow/deny and a set of log fields |
| 08 | Four artifacts, one incident | Reading one artifact well is a skill; reconciling four that disagree is the job |

Tracks 00 and 01 have directories; the rest are map, not written content.

## What you can do after each

This is the contract. If a track finishes and these aren't true, the track
failed — not the reader.

**00 — The tap.** State where in the stack a given capture was taken and what
that position already did to the frame: why the FortiOS sniffer goes silent on
an offloaded session, why Wireshark shows a 32 kB "packet" on a 1500-byte link,
why a trunk capture has no VLAN tag, why pktmon can name a drop reason that a
NIC capture structurally cannot. Say how many records one flow becomes and why a
per-record byte threshold misses slow exfil.

**01 — The link.** Explain why a firewall log carries a router's MAC and not the
host's. Decide from a capture alone whether the sender was on your segment.
State what a VLAN guarantees and what it doesn't. Explain, mechanically, why
mitm6 and Responder work and what evidence each leaves.

**02 — Identity.** Resolve an alert IP + timestamp to a machine with a stated
confidence, and name the specific log sources that would raise it. Explain why
one laptop appears under six IPv6 addresses in a week and what that does to
every IP-keyed detection. Explain why a FortiGate log says `user="jdoe"` for a
session jdoe did not make.

**03 — Paths.** Predict which sensors a given flow will and will not cross
*before* deploying one. Diagnose "we only see one direction" as routing
asymmetry rather than an attack. Read a TTL to infer hop count and origin OS.
Explain why "we talked to 1.1.1.1" names a service and not a machine.

**04 — Sessions.** Read a firewall log line and reconstruct the packets behind
it, including which byte counter went which way. Tell a port scan, a refused
connection, a silent timeout, and a policy deny apart from their artifacts
alone. Explain why the nightly backup dies at the one-hour mark with no RST from
either end — and why a host would have sent one.

**05 — Names.** Trace a hostname to the exact resolver that answered it and say
whether that answer was loggable anywhere. Explain why the endpoint and the
firewall disagree about which domain a connection was for. Say which names never
reach a resolver at all, and what asked for them instead. Judge whether a given
DNS detection is evadable, and how.

**06 — Opaque traffic.** Given a TLS flow you cannot decrypt, enumerate what you
still know and what detection remains possible. Recover an SNI from a QUIC
capture and explain why that is possible when the payload is not. State what
changes in every log when inspection is switched on, and which applications
break and why. Explain why a `proto=50` log line has no ports and what that
costs every port-keyed rule you own.

**07 — The pipeline.** Predict what the FortiGate does with a packet before
running it. Map every field in a traffic log to the pipeline stage that wrote
it. Explain why a policy that "should match" doesn't. Explain why one flow
appears twice in the logs with two session IDs.

**08 — Capstone.** Take one incident's endpoint record, firewall log, resolver
log and pcap, and build a single defensible timeline — naming, at each seam, the
mechanism that made the join fail and the confidence that survives it.

## Two things carried through every lesson, not taught as one

**IPv6 rides alongside v4**, in the same breath, in every lesson that touches
addressing or resolution. Not a section at the end and not a track at position
09 that nobody reaches. ND against ARP is the clearest available explanation of
what ARP is *for*; RFC 6724's selection rules are the clearest explanation of
why "we don't run IPv6" describes a blind spot rather than a defence.

**Everything is a cache with a timer.** MAC aging, ARP/ND lifetime, DNS TTL,
firewall session TTL, DHCP lease, FSSO logon expiry, flow-exporter active and
inactive timeouts, FortiAnalyzer receive-time lag. Each lesson names the caches
it introduced and what breaks when the timer and reality disagree. This includes
timers in the **logging pipeline**, not just in protocols — several of the worst
correlation failures are two records of one event that a timer pulled apart.

## Deliberately not here

SD-WAN and SASE design, firewall HA configuration, FortiManager/FortiAnalyzer
operations, spanning tree, OSPF and BGP beyond what a path decision needs, VRFs,
VXLAN fabric design, and general vendor feature coverage. All of it is network
engineering. The SD-WAN work has its own repo.

Two consequences of those topics survive, because they are log-reading facts
rather than design topics: HA session pickup (Track 07) and SLA-driven egress
selection making your own source IP unstable (Track 03).

## The rule of the repo

No forward references, and no advancing on a soft checkpoint. Each lesson uses
only what earlier lessons established. Each ends in questions answered from
memory in [PROGRESS.md](PROGRESS.md). A failed checkpoint means the lesson
repeats — that's the design working.

## How a lesson is shaped

```
What it answers   one question, stated plainly
Mechanism         how it actually works, vendor-neutral
Artifact          where this surfaces in evidence — the log field,
                  the capture bytes, the schema column
Hands-on          commands to run, expected result, undo step
Checkpoint        questions answered cold, recorded in PROGRESS.md
```

The **Artifact** section is the one that keeps this repo pointed at its goal. A
lesson that explains a mechanism perfectly and never shows where it appears in
something you'd actually be handed has taught networking, not this.

## Gear

Exercises run on what's already here — nothing needs buying.

| Box | Available for |
|---|---|
| `lx0r` (this laptop) | Hyper-V vSwitches, WSL2 Linux, Wireshark, `pktmon` |
| Home PC | Hyper-V AD lab, Docker/WSL2 `lab-splunk`, **the physical FortiGates** |
| `VEGAS` | FortiManager `10.99.99.10`, FortiAnalyzer `10.99.99.20` |

Anything needing the FortiGates or the AD lab gets flagged in the lesson and
carried out on the home PC.

## Status

Track 00 and Track 01 exist as directories; one lesson is written (the old
Track 01 Lesson 01, "The link"), unworked. The map above is a map, not a promise
of written content — lessons get written as they are reached, so that pacing
follows the checkpoints rather than a plan made before the work started.

See [PROGRESS.md](PROGRESS.md) for the ledger.

## Relationship to the other repos

This repo holds *understanding*. `fortinet-sdwan-lab/` holds build artifacts and
the production rollout; `purple-team-lab/` and `blue-team-hunting/` hold
detection work. When something learned here becomes a config it moves to the
first; when it becomes a detection or a telemetry-gap finding it moves to the
last. Linked, not copied.
