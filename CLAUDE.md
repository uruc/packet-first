# CLAUDE.md — network-study

A deliberate, foundations-up rebuild of networking knowledge, aimed at operating
the Fortinet stack (FortiGate / FortiManager / FortiAnalyzer, SD-WAN and SASE)
with no soft spots. Read this before touching anything in here.

## What this repo is

Study material, written as it is worked through. Not a reference dump, not a
cert cram. Eight tracks, each built only on what came before, each gated by a
checkpoint the user has to answer cold before moving on.

**This is not a beginner course.** The owner is a practising network security
engineer — strong on Fortinet, Windows/AD, Splunk, ATT&CK. The gap being closed
is not vocabulary, it is *mechanism*: the exact ordered decision a box makes,
and where in a pipeline each thing happens. Pitch accordingly:

- Move fast where the ground is already solid. Do not belabour subnetting.
- Stop hard where it is not. Depth beats coverage, every time.
- Explain the mechanism first, vendor syntax second. Concept, then FortiOS.
- An expert relearning foundations still deserves adult prose. No baby talk,
  no "imagine a post office."

## The rules that make this work

**No forward references.** A lesson may only use concepts established in
earlier lessons of the same or an earlier track. If an explanation needs
something not yet covered, either move the lesson or say plainly "this is
Lesson N, we take it on trust until then." Never hand-wave.

**No fabricated output.** Never invent command output, routing tables, capture
contents or device responses. If something has not actually been run, write
what it is *expected* to return and label it as expected. This is a study repo —
invented output teaches a wrong model, which is worse than no model.

**Checkpoints gate progress.** Every lesson ends with questions. They are
answered from memory, in `PROGRESS.md`, before the next lesson is written or
read. A soft answer means the lesson repeats — that is a success condition of
the design, not a failure.

**Hands-on is the point.** Every lesson carries exercises. They run on gear the
owner has: `lx0r` (this laptop — Hyper-V, WSL2, Wireshark), the home PC
(Hyper-V AD lab, Docker/WSL2 `lab-splunk`, the physical FortiGates), or `VEGAS`
(FMG `10.99.99.10` / FAZ `10.99.99.20`). If a step needs hardware not on the
current machine, say so and hand it over — do not pretend it can run here.

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
  NN-lesson.md     one lesson: mechanism → hands-on → checkpoint
labs/              longer multi-step builds referenced by lessons
PROGRESS.md        the ledger — lessons done, checkpoints passed, answers
```

Lesson files follow one shape: **what it answers → mechanism → hands-on →
checkpoint.** Keep it.

## Commit policy

Commit after each lesson is written or each checkpoint is recorded. Message
form: `Track NN: <lesson> — <what changed>`. **Do not push without asking** —
no remote is configured yet.

## Cross-repo

Validated FortiOS behaviour that turns into production or lab work belongs in
`fortinet-sdwan-lab/`, not here. This repo holds the understanding; that one
holds the artifacts. Link across rather than duplicating.
