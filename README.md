# network-study

Rebuilding networking from the ground up, on purpose, so that operating the
Fortinet stack rests on mechanism rather than pattern-matching.

Started 2026-08-12.

## Why this exists

Most working knowledge of networking is *operational*: configure the thing,
traffic flows, move on. That carries you a long way and then stops dead the
moment you meet a box that isn't a router, or a case where the tiebreakers
matter, or a packet that vanishes with a clean config.

The fix isn't more features. It's the decision procedure underneath — the exact
ordered algorithm a box runs on every packet, and where in its pipeline each
thing happens. That's what this repo rebuilds, from the link layer up.

The end state is concrete: **be able to predict what the FortiGate will do
before running the command, and explain why when it doesn't.**

## The rule of the repo

No forward references, and no advancing on a soft checkpoint. Each lesson uses
only what earlier lessons established. Each ends in questions answered from
memory in [PROGRESS.md](PROGRESS.md). A failed checkpoint means the lesson
repeats — that's the design working.

## The tracks

| # | Track | The question it answers |
|---|---|---|
| 00 | [Seeing the network](00-seeing-the-network/) | How do you look at a network and trust what you see? Capture placement, the cross-platform toolkit, diagnostic method |
| 01 | [Foundations](01-foundations/) | What is a link, why did we break the network into pieces, and how does a box decide whether a destination is local? |
| 02 | Switching & segmentation | VLANs, trunking, STP, ARP as a protocol, and the L2 attack surface |
| 03 | Routing | Route tables, RIB vs FIB, selection and tiebreakers, ECMP, OSPF, BGP, VRFs, policy routing |
| 04 | Transport & services | TCP/UDP behaviour, MTU/MSS/fragmentation, DNS, DHCP — the protocols firewalls break |
| 05 | FortiOS packet flow | The pipeline: ingress → DoS → session lookup → route → policy → NAT → UTM → egress. **The hinge of the whole repo.** |
| 06 | NAT & tunnels | NAT types and placement, IPsec/IKEv2, SSL VPN, TLS inspection, certificates |
| 07 | SD-WAN & SASE | Performance SLA, rule evaluation order, ADVPN, overlay design, SASE edge |
| 08 | Availability & operations | HA (FGCP/FGSP), FortiManager/FortiAnalyzer, logging pipelines, diagnostic methodology |

Track 00 is a toolkit, not a model — it builds no theory, it makes the later
exercises trustworthy. Tracks 01–04 are vendor-neutral mechanism. Track 05 is
where it converges: the FortiOS pipeline is only learnable once you know what
each stage is *doing*. Tracks 06–08 are elaboration on that pipeline.

## Two things that are deliberately not tracks

**IPv6** is covered *alongside* v4 in the lessons where addressing and address
resolution happen, not bolted on at the end. Partly because a v6 track at
position 09 is a track nobody reaches — and partly because comparing ND against
ARP is the clearest available explanation of what ARP is *for*: the same
problem, solved twice, the second time by people who had seen the first
attempt.

**"Everything is a cache with a timer"** is a theme, not a lesson. MAC table
aging, ARP cache lifetime, session TTL, route holddown, SLA probe intervals,
DNS TTL — the same shape recurring at every layer, each with its own failure
mode when the timer and reality disagree. Each lesson names its timers; the
pattern accumulates rather than being taught once.

## Status

Track 01 in progress. See [PROGRESS.md](PROGRESS.md) for the ledger.

Later tracks are listed above as a map, not a promise of written content —
lessons get written as they're reached, so that pacing follows the checkpoints
rather than a plan made before the work started.

## How a lesson is shaped

```
What it answers   one question, stated plainly
Mechanism         how it actually works, vendor-neutral
Hands-on          commands to run, expected result, undo step
Checkpoint        questions answered cold, recorded in PROGRESS.md
```

## Gear

Exercises run on what's already here — nothing needs buying.

| Box | Available for |
|---|---|
| `lx0r` (this laptop) | Hyper-V vSwitches, WSL2 Linux, Wireshark, `pktmon` |
| Home PC | Hyper-V AD lab, Docker/WSL2 `lab-splunk`, **the physical FortiGates** |
| `VEGAS` | FortiManager `10.99.99.10`, FortiAnalyzer `10.99.99.20` |

Anything needing the FortiGates or the AD lab gets flagged in the lesson and
carried out on the home PC.

## Relationship to the other repos

This repo holds *understanding*. `fortinet-sdwan-lab/` holds the build
artifacts and the production rollout work; `purple-team-lab/` and
`blue-team-hunting/` hold detection work. When something learned here turns
into a config or a detection, it moves there and gets linked — not copied.
