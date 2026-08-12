# Lesson 01 — The instrument lies

> **What this answers:** what did the capture *stack* itself do to the frame,
> and what did it never hand over at all?

---

## Borrowed on trust

- **MTU** — the largest frame a link will carry. Track 03.
- **TCP segment, session** — a connection with state at both ends. Track 04.
- **Checksum** — a field a receiver uses to detect corruption. Track 04.
- **VLAN tag** — a 4-byte field marking a frame's segment. Track 01.

## Lesson 00 was only half the problem

Lesson 00 said: know what is upstream of the tap. That treats the tap as a
window — a clean pane you look through, where the only question is what side of
the glass you're standing on.

It is not a window. **A tap is a position inside a software stack**, and the
layers between that position and the wire are not passive. They add, remove,
combine, split, and silently discard. So there are two questions, not one:

1. What happened to this packet before it reached my tap? *(Lesson 00)*
2. What does my tap's own stack do to packets, and what does it never see?

The second question is the one nobody asks, and it produces the most confident
wrong answers in the business — because the artifacts it creates look like
network problems.

## Four ways the instrument lies

### 1. Traffic that never reaches the tap's stack

Some devices forward packets in hardware, on a path that does not traverse the
general-purpose CPU. A software sniffer running on that CPU is not positioned on
the forwarding path at all — it can only see what the CPU sees.

On FortiOS this is **offload**. A session's first packets go through the CPU
because a forwarding decision has to be made; once made, the remainder of the
session can be handled by the network processor directly. The consequence is
brutal and counter-intuitive:

> A live, high-volume transfer can produce a *silent* on-box sniffer. You will
> see the session set up, and then nothing.

The wrong conclusion is "the transfer stopped." The right conclusion is "this
session is offloaded," which is a statement about the *instrument*, not the
traffic. The confirmation is to disable offload for that policy temporarily and
watch the packets appear — which is also the proof that the traffic was there
the whole time.

Generalised: **on any device that forwards in hardware, a software capture sees
control-path traffic and not much else.** This is not a Fortinet quirk. It is
what hardware forwarding *is*.

### 2. Traffic transformed after the tap, or before it

On a host, a capture library sits **above the NIC driver**. Modern NICs do work
that used to be the OS's job, and that work happens below the tap:

- **Segmentation offload (TSO/LSO/GSO).** The OS hands the NIC one large buffer
  and the NIC chops it into MTU-sized frames. Your capture sees the buffer.
  Result: **frames far larger than the link's MTU, which never existed on the
  wire in that form.** A 32 KB "packet" on a 1500-byte link is not evidence of
  anything except that you captured above the segmenter.
- **Receive coalescing (LRO/RSC).** The same thing inbound — many small frames
  merged into one before the tap sees them. Your packet counts are wrong and
  your inter-packet timing is destroyed.
- **Checksum offload.** The NIC computes checksums on transmit. Above the NIC,
  the field is still a placeholder. Result: **outbound packets flagged as having
  bad checksums, inbound ones fine.** A capture where every outbound packet is
  "wrong" and every inbound one is right is describing offload, not corruption.

The asymmetry is the tell in all three cases: if a defect appears in exactly one
direction, suspect the instrument before the network.

### 3. Fields removed before the tap

On Windows, the driver model lifts the VLAN tag out of the frame and carries it
as separate metadata. Whether the capture library reconstructs it depends on the
library and its version.

Consequence: **a capture on a trunk that shows no VLAN tags is not evidence the
traffic was untagged.** It may be evidence that your capture path strips them.
The only safe move is to establish, once, on your own machine, whether tags
survive — and then remember the answer.

This is the smallest of the four and the easiest to be caught by, because
nothing looks wrong. The frames are all there. One field is just gone.

### 4. Absence with no reason attached

A NIC-level capture can show you that a packet is not present. It structurally
cannot tell you *why*, because a discard by a component further up the stack
happens somewhere the tap is not.

Windows' `pktmon` is built differently: it attaches at multiple components along
the networking stack and reports drops **with a reason** — filtered, MTU
mismatch, and so on. That difference is the entire reason to reach for it.

> Wireshark answers "what arrived." pktmon can answer "what was dropped and by
> which component." Those are different instruments, and choosing wrongly costs
> you the investigation.

## The capture's own self-report

Two numbers, both ignored, both decisive:

**Snaplen.** The number of bytes retained per packet. If it is shorter than the
packet, the packet is truncated — headers survive, payload does not. A capture
taken with a small snaplen will look complete and support no payload conclusion
whatsoever. Defaults differ by tool and by version; the only correct move is to
check the one you actually used.

**Drop counters.** The capture stack has a fixed-size buffer between the kernel
and the writing process. Under load it overflows and packets are discarded *by
the capture*, not by the network. Every capture tool reports this count.
A capture with a non-zero drop count is a sampled capture, and any claim of the
form "X never happened" is unsupported.

This is the closest thing this lesson has to a cache: a fixed-size buffer whose
overflow is silent in the data and visible only in a counter nobody reads.

---

## Artifact

**In Wireshark** — `[Packet size limited during capture]` (snaplen truncation),
a frame length exceeding the interface MTU (segmentation offload), and
`[incorrect, should be 0x…]` on checksums in exactly one direction (checksum
offload). Three visible strings, three statements about the instrument, zero
statements about the network.

**In the capture file's own metadata** — Wireshark's *Statistics → Capture File
Properties* shows the snaplen and, for live captures, the dropped count. This is
the field to read *first* and essentially nobody does.

**In pktmon output** — a per-component drop reason. This is the one artifact in
the track that answers "why is it missing" rather than "what is here."

**On FortiOS** — the absence of sniffer output during a session you can prove is
active. The artifact is the silence, and the silence has a meaning.

## Caches and timers introduced

**The kernel capture buffer.** Fixed size, not time-based, but it fails the same
way every cache in this repo fails: it is a finite store between a fast producer
and a slower consumer, it discards silently under pressure, and the loss is
recorded somewhere other than the data. When the drop counter is non-zero, your
capture and reality have diverged and nothing in the file says so.

**Session offload state** on a forwarding device is state that is established
after the first packets and persists for the session — which is why the sniffer
sees the beginning of a conversation and not the middle.

---

## Hands-on

About forty-five minutes. Steps 1–4 run on `lx0r`. Step 5 needs the physical
FortiGates and therefore the **home PC** — flag it and carry it out there.

### 1. Read the instrument's spec sheet

```powershell
Get-NetAdapterAdvancedProperty -Name "Wi-Fi" | Format-Table DisplayName,DisplayValue
Get-NetAdapterChecksumOffload -Name "Wi-Fi"
Get-NetAdapterLso -Name "Wi-Fi"
```

*Expected:* a list of offload features with enabled/disabled states. Names vary
by vendor — look for large send, receive segment coalescing, and checksum
entries. Anything enabled here happens **below your tap**.

### 2. Toggle segmentation offload — undo first

> **Undo, before you break it.** Write this down before running the next
> command. Re-enable with:
>
> ```powershell
> Enable-NetAdapterLso -Name "Wi-Fi"
> ```
>
> Toggling an offload setting resets the adapter. **Expect the link to drop for
> a few seconds.** Do not run this on a machine you are connected to remotely,
> and do not run it mid-transfer. If the adapter does not come back, disable and
> re-enable it in Network Connections.

Capture a sustained download, then:

```powershell
Disable-NetAdapterLso -Name "Wi-Fi"
```

Capture the same download again, then re-enable immediately using the command
above.

*Expected:* in the first capture, outbound frames substantially larger than the
link MTU. In the second, no frame exceeding it. Same network, same transfer, two
different pictures — produced entirely by a setting on your own machine.

### 3. Make the checksum artifact appear

In Wireshark: *Edit → Preferences → Protocols → TCP*, enable checksum
validation. Capture, then filter:

```
tcp.checksum.status == "Bad"
```

*Expected:* if checksum offload is enabled, matches in the **outbound**
direction only. Then turn validation back off — it is off by default precisely
because this artifact generates false alarms.

Ask yourself the diagnostic question before moving on: *what would it mean if
inbound packets also failed?*

### 4. Get a drop reason out of pktmon

```powershell
pktmon start --capture --pkt-size 128 --file C:\Temp\drops.etl
# reproduce the traffic
pktmon stop
pktmon format C:\Temp\drops.etl -o C:\Temp\drops.txt
```

> **Note:** `pktmon` syntax has changed across Windows builds. Run
> `pktmon start --help` first and adapt. If a flag above is rejected, that is a
> version difference, not an error in the lesson.

*Expected:* a text report of packets with the component each traversed, and for
dropped packets, a reason. Compare that against what a plain Wireshark capture
of the same event tells you: Wireshark shows the absence, pktmon should show the
cause.

### 5. FortiOS offload — **home PC, physical FortiGate**

Check the model first, since offload capability depends on it:

```
get hardware status
```

Then, during a sustained transfer through the unit:

```
diagnose sniffer packet <interface> 'host <ip>' 4
```

*Expected on a unit with hardware offload:* the session's opening packets
appear, then output slows dramatically or stops entirely, despite the transfer
continuing. That silence is the lesson.

To confirm the cause rather than assume it, inspect the session:

```
diagnose sys session filter dst <ip>
diagnose sys session list
```

*Expected:* session flags indicating the session is offloaded. Terminology
varies by FortiOS version and platform; read what is actually printed rather
than looking for a specific string.

> **Optional, and undo first.** To prove the traffic was always there, offload
> can be disabled per policy. **Undo:**
>
> ```
> config firewall policy
>   edit <id>
>     set auto-asic-offload enable
>   next
> end
> ```
>
> Only then set it to `disable`, re-run the sniffer, and **restore it
> immediately.** This changes how the box forwards production-path traffic and
> costs throughput while disabled. Do it on a lab policy, not a policy carrying
> anything you care about.

---

## Checkpoint

Answer from memory, in [PROGRESS.md](../PROGRESS.md), without scrolling back.

1. A capture on a 1500-byte link shows a 40 KB frame. What happened, where
   exactly did it happen relative to your tap, and what does it tell you about
   the wire?
2. Every outbound packet in your capture has a bad checksum; every inbound one
   is fine. Give the explanation, and give the reasoning that gets you there
   without checking any settings.
3. An on-box sniffer goes silent during a transfer you can prove is running.
   What is the mechanism, and why does it produce silence rather than errors?
4. What can `pktmon` tell you that a NIC-level capture structurally cannot, and
   why is it a property of *where it attaches* rather than of the tool's
   features?
5. Two numbers decide whether a capture supports the claim "X never happened."
   Name them and say what each one being wrong would hide.
