# Lesson 02 — Records that are not packets

> **What this answers:** when the evidence is a log line or a flow record rather
> than a capture, what did that form throw away — and how many records does one
> conversation become?

---

## Borrowed on trust

- **5-tuple** — source address, source port, destination address, destination
  port, protocol. The conventional key for "one conversation." Track 04.
- **TCP session, session close** — a connection with a defined beginning and
  end. Track 04.

## Three forms, one packet

Most of the evidence you are handed is not a capture. It is a **summary**,
computed by some device, at some moment, according to some schema. Three forms
dominate, and each is lossy in its own specific direction:

| Form | Keeps | Destroys | Emitted by |
|---|---|---|---|
| **Packet capture** | Everything the tap saw, up to snaplen | Nothing, but is tap-limited and expensive | A tap |
| **Flow record** | 5-tuple, counters, timestamps | All content, and packet boundaries | An exporter, on a **timer** |
| **Log line** | Whatever the device's schema has | Everything not in the schema | A device, at a **moment it chose** |

Lesson 00 asked *where* the evidence was taken. Lesson 01 asked what the
*instrument* did to it. This lesson asks the third question: **what function was
applied to the packets to produce this record, and what does that function
discard?**

The three questions together are the whole of Track 00, and every one of them is
answerable before you know anything about networking. That is why this track is
first.

## Flow records are timer output, not traffic output

This is the single most useful thing in the lesson, and it is almost never
taught.

A flow exporter watches packets and maintains a table of in-progress flows keyed
on the 5-tuple. It emits a record when one of two timers fires:

- **Inactive (idle) timeout** — no packets for N seconds, so the flow is
  considered over and the record is emitted. Typically short: tens of seconds.
- **Active timeout** — the flow is *still running*, but it has been in the table
  too long, so a record is emitted anyway and the counters reset. Typically
  long: minutes to tens of minutes. Defaults differ by vendor and are
  configurable; check the exporter you actually have rather than assuming.

The active timeout is the trap. It means:

> **One TCP connection becomes ⌈duration ÷ active_timeout⌉ flow records**, each
> with its own start time and its own byte count, and none of them individually
> representing the transfer.

Consequences, in the order they will bite you:

1. **A per-record byte threshold measures the timer, not the transfer.** A slow
   three-hour exfiltration split across six records never trips a "large upload"
   rule that a single fast upload would trip instantly. The detection is
   measuring the exporter's configuration.
2. **Records must be summed by 5-tuple before they mean anything** — and you
   need to know the active timeout to know whether summing is even correct,
   because two genuinely separate connections can share a 5-tuple over time.
3. **Absence is weaker than it looks.** Many exporters sample. Under sampling,
   "no record exists" means "no record was created," which is not the same as
   "no traffic occurred."

The pattern here is the repo's running theme, pointed at telemetry instead of at
protocols: *a cache with an expiry, whose expiry decides what the evidence looks
like.* Everything downstream of a flow exporter is shaped by two numbers that
have nothing to do with the network.

### And one flow format cannot represent IPv6 at all

The record format matters as much as the timers. NetFlow v5 has fixed-width
32-bit address fields — it is structurally incapable of carrying an IPv6 flow,
not merely unconfigured for it. The template-based formats that replaced it
(NetFlow v9, IPFIX) can.

So on a v5 exporter, IPv6 traffic does not produce short records or wrong
records. **It produces no records.** Your flow data will be silently and
completely IPv4-only, and nothing in the data says so — the absence looks
identical to an absence of traffic.

That is the third distinct way this track has now shown you an absence with four
possible causes, and it is worth noticing that the fix is the same every time:
establish what the instrument is *capable* of representing before concluding
anything from what it didn't.

## Log lines are emitted at a moment, and the moment is chosen

A device writes a log record when its own logic says to. That moment is a design
decision with consequences you will hit constantly:

**A device that logs at session close cannot have logged during the session.**
This is not a limitation, it is arithmetic — fields like duration and total
bytes are only knowable once the session ends. So a firewall that reports
duration and byte counts is telling you, implicitly, that the record was written
at the end.

Which means: **the log's timestamp is not the time the traffic started.** For a
long session, the log timestamp and the traffic's actual start can be an
arbitrary distance apart, bounded only by how long the session ran. Two sensors
observing one event will therefore disagree about when it happened, by exactly
the session duration, and the correct reconciliation is arithmetic rather than a
tolerance window.

There is a second, independent offset stacked on top: the time a record was
*generated* by the device and the time it was *received* by the collector are
different fields, and collectors do not always display the one you assume. If
your search window is built on receive time and your hypothesis is built on
event time, you can search the right window and find nothing.

And a third: log transport itself can be lossy. A device with a finite in-memory
buffer shipping logs to a collector that becomes unreachable does not queue
indefinitely — it discards. A gap in a log source is not proof of a gap in
traffic.

## Clock synchronisation, stated once

Every correlation claim you will make in Track 04 and every claim in the Track 08
capstone assumes the clocks on the boxes involved agree. That assumption is
usually true and occasionally catastrophically false, and it is invisible when
false — the records simply fail to line up and you conclude they are unrelated
events.

It gets one paragraph, here, and is then treated as a precondition to *check*
rather than a topic to study: **before correlating two sources, verify both
clocks and record their offset.** If you cannot, say so in the finding.

---

## Artifact

**In a flow record** — the start-time, end-time, and byte-count fields. Six
records for one connection, each internally consistent, collectively misleading
unless summed.

**In a FortiGate traffic log** — the `duration`, `sentbyte` and `rcvdbyte`
fields, and the record's own timestamp. Their coexistence is the proof that the
record was written at session close: `log_timestamp − duration ≈ actual start`.
That subtraction is the correct join key against an endpoint record, and it is
the single most useful piece of arithmetic in this track.

**In Defender's `DeviceNetworkEvents`** — `Timestamp` is when the *endpoint*
recorded the connection, which is at its start. Joining that against a firewall
log directly will fail on any session that ran for more than the tolerance you
allowed.

**In FortiAnalyzer** — the displayed Date/Time column is `itime`, the time FAZ
*received* the log, not `eventtime`, the time the FortiGate generated it. A
search window built on one and a hypothesis built on the other can miss the
event entirely. Establish which column you are filtering on, once, for every
source you use — this is not a FortiAnalyzer quirk, it is what every collector
does, and FAZ merely names both fields honestly.

## Caches and timers introduced

This lesson is almost entirely timers. Four:

| Timer | Where it lives | What breaks when it expires |
|---|---|---|
| **Flow inactive timeout** | The exporter | A flow is declared over while the connection is still open |
| **Flow active timeout** | The exporter | One conversation is cut into N records, breaking any per-record threshold |
| **Log emit trigger** | The logging device | The record's timestamp is the session's *end*, not its start |
| **Clock offset** | Every box, independently | Two records of one event fail to correlate and look like two events |

The first two determine what the evidence *is*. The second two determine whether
two pieces of evidence can be joined. None of them are network behaviour; all of
them shape every conclusion you draw from network telemetry.

---

## Hands-on

About forty minutes. Steps 1–3 and 5 run on `lx0r`. Step 4 needs a FortiGate and
therefore the **home PC**.

**Nothing in steps 1–3 or 5 changes configuration** — these are read-only
analysis and a status query. No undo is required. Step 4 is also read-only.

### 1. Capture one conversation and measure it properly

Capture a single sustained download of at least a minute. Then, in WSL2:

```bash
tshark -r download.pcap -q -z io,stat,0
tshark -r download.pcap -q -z conv,tcp
```

*Expected:* the first gives total packets and bytes across the file. The second
gives one row per TCP conversation with packet counts, byte counts in each
direction, a start time and a duration.

Write down, for your one conversation: **start time, duration, bytes each
direction.** These are the ground truth every later step is compared against.

### 2. Watch the summary discard things

Look at what the `conv,tcp` row does *not* contain: no packet boundaries, no
timing between packets, no content, no flags. Then ask the question that
matters — *name one thing you concluded from the capture in step 1 that this row
cannot support.* Write it down.

### 3. Split one flow into many — the active timeout, demonstrated

If a flow exporter is available in WSL2, run your pcap through it twice with
different active timeouts:

```bash
softflowd -r download.pcap -n 127.0.0.1:9995 -t maxlife=30
```

> **Check the flag names first** with `softflowd -h`. Timer names and syntax vary
> by version, and `maxlife` is the option that corresponds to the active timeout
> described above. If the tool is not installed or the flags differ, do not
> guess — the fallback below teaches the same thing.

*Expected:* a conversation longer than the active timeout produces multiple
records rather than one; shortening the timeout produces more of them.

**Fallback with no exporter, same lesson:** use tshark to slice your own capture
into fixed windows and total each one.

```bash
tshark -r download.pcap -q -z io,stat,30
```

*Expected:* the transfer's bytes distributed across several 30-second buckets.
Now answer: **if a detection alerted on any single bucket exceeding a threshold,
what size transfer would evade it, and how would the attacker choose the rate?**
That question is the whole point of the step.

### 4. The session-close arithmetic — **home PC, FortiGate**

Run a transfer through the FortiGate that lasts at least two minutes. Then find
it in the traffic log — GUI or CLI.

*Expected:* one record, timestamped at or after the transfer's **end**, carrying
a duration and byte counters.

Now do the arithmetic: subtract `duration` from the log timestamp and compare it
against the actual start time you noted in step 1. Then compare the log's byte
counters against tshark's, in each direction.

They will not match exactly. **Before looking for a reason, write down three
hypotheses for why counters measured at two different points on a path would
differ.** You are not equipped to resolve them yet — that is Tracks 03 and 04 —
but forming them now is what makes those tracks land.

### 5. Establish your clock offset

```powershell
w32tm /query /status
w32tm /stripchart /computer:time.windows.com /samples:5 /dataonly
```

*Expected:* a source, a last-sync time, and a per-sample offset in seconds.
Record the offset in `PROGRESS.md`. When a correlation fails later, this is the
first number to re-check rather than the last.

---

## Checkpoint

Answer from memory, in [PROGRESS.md](../PROGRESS.md), without scrolling back.

1. A three-hour connection appears as six flow records. Explain the mechanism,
   and say what would have to be true for it to appear as three instead.
2. A "large upload" detection thresholds bytes per flow record and never fires
   on a known slow exfiltration. Explain the failure without using the word
   "threshold" — what is the detection actually measuring?
3. A firewall log line carries a `duration` field. What does that fact alone
   tell you about when the record was written, and why does it follow
   necessarily rather than by convention?
4. An endpoint says 14:03 and a firewall says 14:41 for the same connection.
   Give the arithmetic that reconciles them, and name a second, independent
   offset that could also be involved.
5. Rank packet capture, flow records and firewall logs by how much they
   preserve — then give one question each of the other two can answer that the
   richest one cannot.

Question 5 has no clean answer and is meant not to. The ranking is easy; the
second half is the lesson.
