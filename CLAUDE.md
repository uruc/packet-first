# CLAUDE.md — packet-first

A foundations-up rebuild of networking knowledge for a **security engineer**,
aimed at one outcome: a packet capture, a firewall log, a proxy decision, or a
detection that fired makes sense at the mechanism level rather than by
pattern-match. Read this before touching anything in here.

## What this repo is

Study material, written as it is worked through. Not a reference dump, not a
cert cram. Nine tracks, each built only on what came before, each gated by a
checkpoint the user has to answer cold before moving on.

**This is not a beginner course, and it is not a network-engineering course.**
The owner is a practising security engineer — detection engineering in Defender
XDR and Splunk, plus network security architecture on a Fortinet stack. They are
strong on Fortinet, Windows/AD, Splunk and ATT&CK. They are not becoming a
network engineer and material that only a network engineer needs is dead weight,
not a harmless extra.

The gap being closed is *mechanism*: the exact ordered decision a box makes,
which stage of which pipeline wrote a given field, and which timer could make
that field lie. Pitch accordingly:

- **Do not assume the fundamentals are already solid.** The owner operates this
  stack professionally, but operational fluency and mechanism are different
  things, and which parts are genuinely solid is *unverified*. Job title is not
  evidence. Establish the mechanism properly the first time it is needed —
  including addressing and subnetting — and let the checkpoint, not an
  assumption about the reader, decide whether it can be moved through quickly.
- Move fast only where a passed checkpoint has shown the ground holds.
- Stop hard where it does not. Depth beats coverage, every time.
- Explain the mechanism first, the artifact second, vendor syntax third.
- An expert relearning foundations still deserves adult prose. No baby talk,
  no "imagine a post office."

**Fortinet is where the fluency gets spent, not the reason for it.** FortiOS
appears as the worked instance of a general mechanism and as the source of log
lines to read. It is never the subject.

## The rules that make this work

**Every lesson lands on an artifact.** A lesson that explains a mechanism
perfectly and never shows where it surfaces in evidence — the log field, the
capture bytes, the schema column, the thing the owner would actually be handed —
has missed the point of the repo. Name the artifact explicitly. This is the rule
that keeps the material aimed at the goal; it outranks completeness of coverage.

**No forward references.** A lesson may only use concepts established in
earlier lessons of the same or an earlier track. If an explanation needs
something not yet covered, either move the lesson or say plainly "this is
Lesson N, we take it on trust until then." Never hand-wave.

**No fabricated output.** Never invent command output, routing tables, capture
contents, log lines or device responses. If something has not actually been run,
write what it is *expected* to return and label it as expected. This is a study
repo — invented output teaches a wrong model, which is worse than no model. It
is worse still here than in most repos, because the whole subject is reading
evidence.

**Checkpoints gate progress.** Every lesson ends with questions. They are
answered from memory, in `PROGRESS.md`, before the next lesson is written or
read. A soft answer means the lesson repeats — that is a success condition of
the design, not a failure.

**IPv6 rides along, it does not get deferred.** Any lesson touching addressing
or address resolution covers v6 next to v4 in the same breath. Do not write an
"and now IPv6" section at the end of a lesson, and do not promise a v6 track
later. ND-vs-ARP comparisons are encouraged — they explain both. The v4/v6
destination-selection rules matter as much as the addressing does: "we don't run
IPv6" is a description of a blind spot, not a defence.

**Name the timers — including the ones in the logging pipeline.** Nearly every
mechanism in networking is a cache with an expiry, and most confusing failures
are a timer disagreeing with reality. That extends past protocols into the
telemetry chain: flow-exporter active and inactive timeouts, identity-mapping
expiry, log receive-time versus event time. Each lesson states which caches it
introduced and what happens when they age out. This accumulates into a theme; do
not teach it as a standalone lesson.

**Hands-on is the point.** Every lesson carries exercises. They run on gear the
owner has: `lx0r` (this laptop — Hyper-V, WSL2, Wireshark, `pktmon`), the home
PC (Hyper-V AD lab, Docker/WSL2 `lab-splunk`, the physical FortiGates), or
`VEGAS` (FMG `10.99.99.10` / FAZ `10.99.99.20`). If a step needs hardware not on
the current machine, say so and hand it over — do not pretend it can run here.

**The user runs the commands.** Hand over the command to run, with the expected
result and the undo step. Do not execute lab config for them; the doing is the
learning. Writing and editing files in this repo is fair game.

**Every destructive exercise ships its undo first.** Anything that can drop
connectivity or change host routing states the rollback command *before* the
command that breaks it.

## Layout

```
NN-track-name/
  README.md        what this track answers, lesson list, status
  NN-lesson.md     one lesson: mechanism → artifact → hands-on → checkpoint
labs/              longer multi-step builds referenced by lessons
PROGRESS.md        the ledger — lessons done, checkpoints passed, answers
```

Lesson files follow one shape: **what it answers → mechanism → artifact →
hands-on → checkpoint.** Keep it.

## The track map

Ordering principle: every artifact you read is a lossy projection of a packet,
taken at one point, after everything upstream already acted on it. The tracks
build the packet, then the things that rewrite it, then the machine that
observes and records it.

| # | Track |
|---|---|
| 00 | The tap — what the capture point already did to the packet |
| 01 | The link, and what it leaks |
| 02 | Addresses, identity, and attribution |
| 03 | Paths, and why your sensor saw half the conversation |
| 04 | Sessions — the unit every firewall log is written in |
| 05 | Names, and everything that resolves without asking |
| 06 | Opaque traffic — what a middlebox still knows |
| 07 | The pipeline and the log line it writes |
| 08 | Four artifacts, one incident (capstone) |

Track 00 teaches nothing about how networks work, only about how instruments
lie — it has no prerequisites and is needed from Track 01's first exercise.
Track 08 is where the ordering principle stops being a claim and becomes a
graded exercise. See `README.md` for the per-track capability claims; those
claims are the contract and a track that does not deliver them has failed.

**Out of scope, on purpose:** SD-WAN and SASE design, HA configuration,
FortiManager/FortiAnalyzer operations, spanning tree, OSPF/BGP beyond what a
path decision needs, VRFs, VXLAN fabric design, vendor feature tours, and
malware-analysis topics wearing a networking hat (DGA algorithm reversal). If a
lesson starts drifting into any of these, that is a signal it has left the goal.

## Commit policy

Commit after each lesson is written or each checkpoint is recorded. Message
form: `Track NN: <lesson> — <what changed>`, or `crash-course: <what changed>`
for the parallel lane.

**Do not push without asking.** The remote is
`github.com/uruc/packet-first` and it is **public** — so the no-secrets rule is
not a hygiene preference here, it is the actual threat model. Lab addresses
(`10.99.99.x`) and machine names are already published and are fine. Anything
naming the employer, a production address, a real policy or a real log line is
not, and does not go in this repo at all.

**Local directory is still `network-study/`.** The repo was renamed on GitHub;
the folder on disk was deliberately not, because the sibling repos' CLAUDE.md
refers to it by path.

## Cross-repo

Validated FortiOS behaviour that turns into production or lab work belongs in
`fortinet-sdwan-lab/`. Telemetry gaps and detection logic discovered here belong
in `blue-team-hunting/`. This repo holds the understanding; those hold the
artifacts. Link across rather than duplicating.
