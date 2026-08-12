# Block 1b — The plan

> **What this answers:** You can compute any subnet. Nothing has told you how big
> to make one, or where to put it. How is an addressing plan actually designed,
> and what does a good one buy that a working one doesn't?

One sitting. Runs on `lx0r` — this is a design block, mostly paper. FortiGate
sections flagged **[home PC]**.

Depends on: block 1a (the mask, and that the mask generates a connected route).

**Three forward references, declared up front** rather than hand-waved:

- **Summarisation in the route table** is block 3. Here you need only one fact,
  taken on trust: *a router can hold one entry covering many subnets, if and only
  if those subnets share leading bits and are all reachable the same way.*
- **NAT** is block 6. Here it appears only as the consequence of a decision you
  are making in this block.
- **VLANs** are block 2. Here, treat "VLAN" as a synonym for "one broadcast
  domain, one subnet" — block 2 explains how one wire carries several.

---

## The problem 1a created

The mask lets you carve address space to any size you like. That is a capability,
not a design. Nothing so far says whether a floor of user PCs should be a /24 or
three /26s, or whether branch 14's servers should be numbered near branch 13's.

And here is the trap: **every one of those choices works.** Traffic flows either
way. A bad plan is not a plan that fails, it is a plan that is indistinguishable
from a good one on day one and costs you for a decade afterwards. So you cannot
evaluate a plan by testing it. You have to evaluate it against what it makes
possible later.

## Where the space comes from

Before design, constraints.

### RFC 1918 and the exhaustion story

IPv4 is 32 bits — 4.3 billion addresses, minus large reserved ranges. That was
never going to be enough for a device-per-person world, and by the early 1990s
the trajectory was obvious. Two responses landed together and they are related:

- **RFC 1918 private space** — three ranges declared non-routable on the public
  internet, reusable by everyone simultaneously:

  | Range | Prefix | Size |
  |---|---|---|
  | `10.0.0.0` – `10.255.255.255` | `10.0.0.0/8` | 16.7 M |
  | `172.16.0.0` – `172.31.255.255` | `172.16.0.0/12` | 1 M |
  | `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | 65 K |

- **NAT**, the thing that makes private space usable for reaching the internet.

The pairing is the point. Private space alone would just be an unreachable
island; NAT alone would have nothing to translate from. **Together they let one
public address front thousands of private ones**, and that is what deferred
exhaustion for thirty years.

You inherit two things from that history. First, ample private space — 16.7
million addresses in `10.0.0.0/8`, which is why almost every enterprise uses it
and why you can afford to waste space for legibility. Second, a permanent
translation layer sitting between your addressing and the internet's, with all
the consequences block 6 covers.

**Note the shape of this**: a scarcity problem was solved by adding a translation
layer, and the translation layer became a permanent architectural feature with
its own costs. That pattern recurs. It is the single most useful lens for
answering "why was it done this way" about anything in a network built between
1995 and now.

### What a plan is optimising for

Not address efficiency. Say that plainly, because it is the instinct 1a's
arithmetic tends to install:

> **In RFC 1918 space, address efficiency is worth almost nothing. Structure is
> worth a great deal.**

You have 16.7 million addresses in `10/8`. Burning a /24 on a segment with nine
devices costs you nothing measurable. Numbering that segment somewhere that
doesn't summarise costs you a route-table entry at every router forever, a
firewall rule that can't be written as one line, and a detection rule that has to
enumerate instead of match a prefix.

The four things a plan is actually optimising:

1. **Summarisation.** Can a whole site be described by one prefix? If yes, one
   route, one firewall object, one rule. If no, one of each *per subnet*, forever.
2. **Legibility.** Can a human read `10.14.30.55` and say what it is without a
   lookup? An address that self-describes turns every log line into a partially
   pre-enriched record.
3. **Failure and broadcast domain size.** How many devices go down together, and
   how much broadcast noise does each device eat? (Block 2 gives the real
   numbers; for now: smaller is safer, and bigger is easier to manage.)
4. **Growth.** Can site 15 be added without renumbering site 14, and can a
   segment double without moving?

Numbers 1 and 2 are what separate a good plan from a working one, and they pull
in the same direction. Number 3 pulls against them mildly. Number 4 is where
plans usually die, and it dies for a reason worth naming: growth was planned as
*more space at the end* rather than as *space left free in the middle*.

## The method

### Step 1 — allocate on bit boundaries, always

The rule underneath everything else:

> **A block that summarises is a block whose members share leading bits and whose
> boundaries fall on a power of two.**

`10.14.0.0/16` summarises. `10.14.0.0` through `10.14.199.255` does not — there is
no prefix that expresses it, so it can never be one route or one address object.
Ranges that are not powers of two are not addressable as a single concept, ever.
That is not a convention; it is what a prefix mathematically is.

So allocate in halves. Splitting a /16 gives two /17s, four /18s, eight /19s. Any
allocation that isn't one of those cannot be summarised, and the space it saved
was worth nothing anyway.

### Step 2 — build a hierarchy that matches the topology

Summarisation only works if **everything inside a prefix is reachable the same
way**. So the address hierarchy must mirror the *physical and routing* hierarchy,
not the org chart.

Three levels is usually right:

```
    10.  14.  30.  55
    │    │    │    └── host
    │    │    └────── function within the site   (VLAN / segment)
    │    └─────────── site
    └──────────────── enterprise
```

Which reads as: enterprise `10/8`, site 14, function 30, host 55.

If you need a region tier — because regions aggregate at a regional hub, and a
hub is a routing boundary — take it out of the site bits:

```
    10.  R R S S S S S S  .  F F F F F F F F  .  host
         └── region ──┘      └── function ─┘
              └ site ┘
```

The test for whether a tier deserves bits is not organisational, it is: **is there
a device where all traffic for this tier converges?** If yes, that device can
summarise, and the tier is real. If no, the tier is documentation wearing a
prefix, and it will not save you a single route.

### Step 3 — make the function octet mean the same thing everywhere

This is the highest-value, lowest-effort decision in the whole block.

Fix a meaning for the third octet and hold it across every site:

| Third octet | Function |
|---|---|
| `0` | Infrastructure — switch and firewall management |
| `10` | User endpoints |
| `20` | Voice |
| `30` | Servers |
| `40` | Printers, scanners, building systems |
| `50` | Wireless guest |
| `60` | Point-to-sale / branch-specific |
| `250+` | Point-to-point links, loopbacks |

Now `10.14.30.55` and `10.22.30.9` are both servers, at sites 14 and 22, and
anybody can see it. But the operational payoff is much larger than readability:

- A firewall rule "all user endpoints may reach all servers" becomes matchable on
  a consistent pattern rather than an enumeration of 40 unrelated subnets.
- A detection rule "no workstation should ever talk to a branch switch's
  management address" becomes one expression.
- A new site inherits the whole rule set by construction. Nobody edits policy to
  add site 23.

That last point is the one to take to work. **A consistent plan means adding a
site is an addressing task, not a policy task.** An inconsistent plan means every
new site touches every rule set, forever, and every touch is a change window and
a chance to get it wrong.

### Step 4 — leave the growth in the middle, not at the end

The universal mistake: allocate sites 1, 2, 3, 4 consecutively, plan to grow by
appending 5, 6, 7. Then site 2 needs to double, and there is nothing next to it,
so it gets a second non-adjacent block — and site 2 is no longer one prefix. Its
summary is gone, and it never comes back without renumbering.

Fix: **allocate sparsely on the bit you expect to need.** If sites are /20s and
you expect some to double, allocate them at /19 spacing and use the lower half.
The upper half is not wasted — it is *reserved adjacency*, and adjacency is the
only thing that makes growth free.

Same idea inside a site: if user endpoints get `x.x.10.0/24` and might need a
second, don't put voice at `x.x.11.0`. Space the functions so each can grow into
a shorter prefix in place.

You are not saving addresses. **You are preserving the ability to shorten a
prefix later,** which is the only form of growth that doesn't cost a renumber.

### Step 5 — write down the point-to-point and loopback space separately

Router-to-router links and loopbacks are numerically tiny and topologically
everywhere. Give them their own block at the top of the site's range, or a
dedicated range enterprise-wide.

Two reasons. They summarise separately (they are infrastructure, and it is often
correct to route them differently from user space). And they are the addresses
that appear in traceroutes, BGP peerings, IPsec tunnel endpoints and OSPF router
IDs — so having them instantly recognisable as infrastructure means you know what
you are looking at the moment you see one. Blocks 3, 7 and 8 all cash this.

## A worked plan

A bank: three regions, up to 60 branches, a data centre, a few hundred segments.

```
10.0.0.0/8                       enterprise

  10.0.0.0/16                    shared services / data centre
    10.0.0.0/24                    infrastructure management
    10.0.10.0/24                   DC user / jump hosts
    10.0.30.0/22                   DC servers  (grown from /24, in place)
    10.0.250.0/24                  loopbacks   (each a /32 within it)
    10.0.251.0/24                  point-to-point links (as /31s)

  10.16.0.0/12                   region A          ← one route at the region hub
    10.16.0.0/20                   branch A-01     ← one route at the branch
      10.16.0.0/24                   infrastructure
      10.16.10.0/24                  user
      10.16.20.0/24                  voice
      10.16.30.0/24                  servers
      10.16.40.0/24                  building systems
      10.16.50.0/24                  guest wireless
    10.17.0.0/20                   branch A-02
    10.18.0.0/20                   branch A-03
    ...

  10.32.0.0/12                   region B
  10.48.0.0/12                   region C
  10.64.0.0/10                   RESERVED — do not allocate
```

Read what this bought:

- **The region hub advertises one prefix** — `10.16.0.0/12` — instead of sixty
  branch routes. Block 3 explains the mechanism; the design consequence is
  visible now.
- **A branch is one object.** "All of branch A-02" is `10.17.0.0/20`. One firewall
  address object, one rule, one detection filter.
- **Function is readable at a glance**, at every site.
- **Branches are spaced a whole /16 apart** even though each uses a /20. Fifteen
  sixteenths of each branch's allocation is unused, deliberately. Any branch can
  grow to a /16 without moving, and the region still summarises to a /12.
- **A quarter of the enterprise space is reserved and untouched**, because the
  acquisition, the cloud footprint or the segmentation project you cannot predict
  will need contiguous space, and the only way to have contiguous space in ten
  years is to not spend it now.

Is that wasteful? Enormously. It uses maybe 0.5% of the addresses it reserves.
And it costs nothing, because the space is free and the structure is not.

## Where plans get broken

Four ways, all of which you will meet:

1. **Merger and acquisition.** Two organisations both used `10.0.0.0/16` for their
   data centre. The addresses collide, both are in use, and neither can renumber
   quickly. The fix is NAT between them — block 6 — and it is permanent, because
   the temporary fix always is. This is *the* argument for using an unusual,
   sparse slice of `10/8` rather than starting at `10.0`.
2. **VPN peers and third parties.** Your partner uses `192.168.1.0/24`. So does
   half the world, including possibly one of your own branches. Same fix, same
   permanence.
3. **Cloud.** A VPC gets a prefix that has to not collide with anything on-prem,
   present or future — the reason for the reserved block above.
4. **Organic growth by whoever was on call.** A segment gets carved out of
   whatever was free at 3 a.m. It works. It is not summarisable, and now the site
   is two routes instead of one and every rule that referenced "the site" is
   quietly wrong. This is the common one, and it is why the plan has to be written
   down somewhere that the person at 3 a.m. will actually look.

## The IPv6 plan, alongside

Same method. One entire step deleted.

You are assigned a **/48** (typical site or enterprise allocation) or a **/56**
(typical smaller allocation) by your ISP or RIR. Every subnet is a /64 — 1a
settled that. So:

```
    2001:db8:abcd::/48        your allocation
    └──── 48 bits ────┘└─16─┘└──── 64 bits ────┘
                       subnet      interface ID
                        bits
```

**All your design freedom is those 16 subnet bits.** 65,536 subnets, and you never
size any of them.

Which means step 1 (bit boundaries) applies, step 2 (hierarchy matching topology)
applies, step 3 (consistent function encoding) applies, step 4 (sparse
allocation) applies — and **the host-count sizing that dominates v4 planning
simply does not exist.**

A hex-legible scheme, using the nibbles of the 16 subnet bits:

```
    2001:db8:abcd:  R S S F  ::/64
                    │ └┬┘ └── function  (0=infra, 1=user, 3=server, 5=guest)
                    │  └───── site
                    └──────── region
```

So `2001:db8:abcd:1023::/64` is region 1, site 02, function 3 — servers.

And the trick worth stealing: **encode the v4 third octet into the v6 subnet
bits**, so `10.16.30.0/24` and `2001:db8:abcd:1630::/64` are the same segment.
Now one mental model covers both stacks and dual-stack troubleshooting stops
requiring two lookups.

The honest summary of v6 planning: it is *easier* than v4, and the reason people
find it harder is that it removes the part they practised most. Sizing was never
the design. It was the tax.

## Caches and timers introduced by this block

**None.** A plan is a document.

Worth noticing, though, that the plan determines the *shape* of several caches you
have not met yet: how many route-table entries exist (block 3), how many firewall
address objects and therefore how long policy evaluation takes (block 6), and how
many prefixes a dynamic protocol has to carry and re-advertise (block 3 and 8). A
plan that summarises makes every one of those tables smaller. **Design decisions
made on paper here show up as table sizes in memory there.**

---

## Artifact

### FortiOS address objects **[home PC]**

The plan's payoff is visible as object count. A summarisable site is one object:

```
config firewall address
    edit "site-A02"
        set subnet 10.17.0.0 255.255.240.0
    next
    edit "site-A02-users"
        set subnet 10.17.10.0 255.255.255.0
    next
end
```

An unsummarisable site is an address *group* of however many pieces it got carved
into, and every rule referencing it drags that group along.

```
show firewall address
show firewall addrgrp
diagnose firewall iprope list
```

*Expected:* `show firewall addrgrp` reveals the shape of your real plan more
honestly than any documentation, because a group with seven members is a site
that did not summarise. **Count the members of your largest group and you have
measured how well your addressing plan is holding up.** That is a real,
runnable audit and it takes one command.

### FortiOS routing **[home PC]**

```
get router info routing-table all
get router info routing-table summary
```

*Expected:* a route count. Compare it to your number of segments. A well-summarised
network shows far fewer routes than segments at any router above the access layer;
a flat one shows roughly one route per segment everywhere. Block 3 explains what
produced each entry — for now the *count* alone is the plan's report card.

---

## Hands-on

About 45 minutes. Paper and `lx0r`; the FortiGate parts are read-only audits to
carry to the home PC.

### 1. Design a plan from scratch

Requirements:

- One data centre, one HQ, up to 24 branches.
- Each branch: user, voice, server, printer, guest wireless, and infrastructure
  management segments. Largest branch has 400 users; smallest has 12.
- HQ has 2,000 users across several floors.
- A cloud footprint is coming and its size is unknown.
- One branch will be acquired-and-merged into within two years.

Produce:

- The enterprise prefix and its top-level split.
- A per-site allocation size, with the reasoning for its size — including how much
  is deliberately left free and what event you left it free *for*.
- The function encoding table.
- The point-to-point and loopback ranges.
- The parallel IPv6 plan from a `/48`.
- **One paragraph on what you would have to renumber if a merger arrived.**

That last item is the actual exercise. A plan that answers it with "nothing" has
reserved correctly.

### 2. Audit the plan you actually have

Take the real addressing at work — or the lab's, if the real one isn't to hand on
`lx0r`. For each site or zone:

- Is it expressible as **one prefix**? Yes or no.
- If no, how many prefixes, and what event caused the split?
- Does the function octet mean the same thing at every site?
- What is the largest firewall address group, and is it large because the plan
  didn't summarise?

Write the count of "no" answers in the ledger. That number is the honest
measurement of the plan, and it will explain more about why certain rule sets
look the way they do than any documentation will.

### 3. Test summarisation by hand

Given these six subnets, find the single shortest prefix that covers all of them
and **nothing else**:

```
10.24.0.0/24    10.24.1.0/24    10.24.2.0/24
10.24.3.0/24    10.24.4.0/24    10.24.5.0/24
```

Then answer the real question: *can these be summarised into one route without
including anything not in the list?* Work it in binary. The answer is instructive
and it is not the one people expect — this is exactly the situation that produces
a "why do we have three routes for that site" conversation.

Then redo it with the last two removed, and note how much the answer changes for a
tiny change in the input. **That sensitivity is why step 4 exists.**

---

## Checkpoint

From memory, in [LEDGER.md](LEDGER.md).

1. Address efficiency is nearly worthless in RFC 1918 space, but bit-boundary
   alignment is critical. Those sound contradictory. Explain why they are not.
2. What is the test for whether an organisational tier deserves bits in the
   address plan? Give the test, and give an example of a tier that would fail it.
3. Consistent function encoding across sites buys something specific in firewall
   policy. State it, and state what it means for the work of adding a new site.
4. Why is growth planned as reserved adjacency rather than as space at the end?
   Describe the exact failure that appending causes.
5. An IPv6 plan deletes one entire step of the v4 method. Name it, name what made
   it deletable, and say why v6 planning is genuinely easier despite feeling
   harder.
6. Name two events that force NAT between internal networks regardless of how
   good your plan was, and name the single design choice that most reduces the
   chance of the first one.

---

## Prediction

**A. The summarisation question, before you compute it.** For hands-on 3, write
down your answer to "can those six /24s be one route?" *before* working it in
binary. Then work it. Most people say yes on instinct because the numbers look
contiguous; contiguity is not sufficient, and finding out which of those two you
believed is the point of the exercise.

**B. [home PC] — route count against segment count.** Before running anything,
predict two numbers:

- how many connected subnets the FortiGate has,
- how many total routes are in its table.

Then:

```
get router info routing-table connected
get router info routing-table all
get router info routing-table summary
```

*Expected:* more total routes than connected ones, with the gap made of static,
default and dynamic entries. Predict the **gap** as well as the totals — if the
gap is much larger than you expected, that is either a plan that isn't
summarising or a dynamic protocol carrying more than you thought, and block 3
will tell you which. Note down which one you'd bet on.

**C. [home PC] — the group audit.** Predict the member count of the largest
address group before running `show firewall addrgrp`. This is the most direct
measurement available of whether your real plan summarises, and predicting it
first is the difference between measuring and confirming.

If B's gap surprises you, note it and carry it forward — block 3 is written to
answer exactly that surprise, and arriving with a specific number to explain is
better than arriving with a topic.
