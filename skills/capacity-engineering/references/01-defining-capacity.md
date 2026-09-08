# Defining Capacity

**Sources:** *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard
(Pragmatic Bookshelf, 2007). Ch. 8 "Introducing Capacity", Sec. 8.1 to Sec. 8.6, p. 161–174.
Sec. 7.1 and Sec. 7.4, p. 147–157 supply the launch story that Sec. 8.3 cites. Sec. 9.1,
p. 176–179 supplies the pool example. Every bare `Sec. x.y, p. n` names *Release It!*.

This file fixes the words. A capacity discussion fails when two people say "performance" and
mean two different numbers. Nygard defines the terms in Sec. 8.1, p. 161–162, and he states
that architecture and development need more precision than marketing does.

**The one sentence that governs the whole file: exactly one constraint sets the capacity of a
system. Every other number is useless for a capacity projection (Sec. 8.2, p. 163).**

---

## 1. Five words, five meanings

| Term | Nygard's definition | Unit | Page |
|---|---|---|---|
| Performance | How fast the system processes a single transaction | Milliseconds per transaction | Sec. 8.1, p. 161 |
| Throughput | The number of transactions in a given time span | Transactions per second | Sec. 8.1, p. 162 |
| Scalability | Sense 1, how throughput changes under varying loads. Sense 2, the modes of scaling that a system supports | A curve, or a mode | Sec. 8.1, p. 162 |
| Capacity | The maximum throughput at an acceptable response time, for a given workload | Transactions per second, plus a workload and a bound | Sec. 8.1, p. 162 |
| Utilization | CPU use, free memory, and I/O rate. Nygard treats these as following variables | Percent, or a rate | Sec. 8.2, p. 164 |

Nygard uses the word scalability in sense 2, which is the addition of capacity to the system
(Sec. 8.1, p. 162). This file uses it the same way. Section 8 gives the two modes.

**Nygard does not define utilization in Sec. 8.1.** He lists CPU use, free memory, I/O rates,
page-swapping rates, and network bandwidth as following variables (Sec. 8.2, p. 164).

Performance has a large effect on throughput, but the customer does not buy performance. The
customer is interested in throughput or in capacity (Sec. 8.1, p. 161). The end user is
interested only in the performance of their own transaction, because the user cannot see the
servers (Sec. 8.1, p. 161–162). Nygard states the user's test: "when the response time exceeds
their expectation, the system is down" (Sec. 8.1, p. 162).

---

## 2. What each number cannot tell you

| Number | What it does not tell you |
|---|---|
| Performance | Nothing about throughput. Faster work outside the bottleneck adds none (Sec. 8.1, p. 162) |
| Throughput | Nothing about the response time of one user (Sec. 8.1, p. 161–162) |
| Scalability, sense 1 | Nothing about the ceiling, until the curve reaches the knee (Sec. 8.2, p. 164) |
| Capacity | Nothing, unless the workload and the response-time bound are both stated (Sec. 8.1, p. 162) |
| Utilization | Nothing about capacity, unless that resource is the constraint (Sec. 8.2, p. 163) |

**Optimizing the performance of a part that is not the bottleneck does not increase
throughput** (Sec. 8.1, p. 162). This is the same rule as Section 4, stated for performance.

---

## 3. Capacity is not a single number

The definition carries two variables. The first is the workload. The second is a human judgment
about an acceptable response time (Sec. 8.1, p. 162).

**There is no single fixed number that you can regard as your capacity** (Sec. 8.1, p. 162).
The workload changes when users want different services around the holidays, and the capacity
then changes with it (Sec. 8.1, p. 162).

### Acceptable response time is a business judgment

| System | The stated bound | Page |
|---|---|---|
| Ecommerce retailer | A response time longer than two seconds makes customers leave | Sec. 8.1, p. 162 |
| Financial exchange | On the order of milliseconds | Sec. 8.1, p. 162 |
| Travel reservation | Five hundred milliseconds for an availability search, thirty seconds to confirm a reservation | Sec. 8.1, p. 162 |

**A capacity number with no workload and no response-time bound is not a capacity number.**
It cannot be tested and it cannot be refuted. This is catalog entry C-03 in
`capacity-defect-catalog.md`.

### Capacity is also a revenue measure

Nygard states that capacity measures how much revenue the system can generate in a given
period (Sec. 8.6, p. 174). A design choice that reduces capacity reduces the top-line revenue
of the company, and the company then pays more capital and operational expense (Sec. 8.6,
p. 174).

---

## 4. Exactly one constraint sets capacity

Nygard states it directly: "In every system, exactly one constraint determines the system's
capacity" (Sec. 8.2, p. 163). The constraint is the limiting factor that reaches its ceiling
first. He cites the Theory of Constraints for the idea (Sec. 8.2, p. 163, footnote 1).

**Once the system reaches the constraint, every other part either queues work or drops it on
the floor** (Sec. 8.2, p. 163).

### The book's two examples

The first example is a database server. An Oracle server with the multithreaded server option
processes as many simultaneous requests as it has daemon processes. With fifty processes, the
fifty-first request waits its turn, and the workers above it idle (Sec. 8.2, p. 163).

The second example inverts the picture. Application server RAM is the constraint, because each
user session consumes RAM. When the RAM is gone, a new session makes the application server
page. The database server becomes idle, because it now receives less work (Sec. 8.2, p. 163).

**Any nonconstraint metric is useless for projecting or increasing capacity** (Sec. 8.2,
p. 163). Nygard repeats the rule in the summary: improving a nonconstraint metric will not
improve capacity (Sec. 8.6, p. 174).

He also states the positive half. **Once you find the constraint, you can predict a capacity
improvement from a change to that constraint** (Sec. 8.2, p. 163). The prediction holds until
something else becomes the constraint.

### The fallacy that this rule kills

Nygard names the reasoning that people apply instead. We handle 10,000 users at 50 percent CPU
use, so we handle 20,000 users (Sec. 8.2, p. 163). The projection is linear and the effects are
not. He adds the image that decides the case. Reading the CPU of the web server tells you
nothing while smoke trickles out of the database (Sec. 8.2, p. 163). This is catalog entry
C-02.

**No simple formula produces an all-encompassing capacity number** (Sec. 8.2, p. 163). Nygard
requires systems thinking instead, which he takes from Peter Senge. It is the ability to think
in dynamic variables, in change over time, and in interrelated connections (Sec. 8.2, p. 163).

---

## 5. Driving variables and following variables

| Kind | Definition | Examples | Can you set it? |
|---|---|---|---|
| Driving variable | A variable outside your control that creates demand (Sec. 8.2, p. 163–164) | Page requests per second, user demand, the clock, the calendar | No |
| Following variable | A variable that moves in response to a driving variable (Sec. 8.2, p. 164) | CPU use, free memory, I/O rate, page-swapping rate, network bandwidth | Only by a design change |

**Every directly measurable performance statistic is a following variable** (Sec. 8.2, p. 164).
That is why a dashboard alone cannot name a constraint. A dashboard shows following variables.

Two more facts change how you read a correlation. A single following variable can correlate
with more than one driving variable. A following variable in one relationship is a driving
variable in another. Database I/O drives the response time of the application server, and that
response time drives the memory use of the web server (Sec. 8.2, p. 164).

**The constraint of the system is a limit in one of the following variables** (Sec. 8.2,
p. 164). Target a correlation coefficient between 0.8 and 1.0 when you relate a following
variable to a driving variable (Sec. 8.2, p. 164, footnote 2).

---

## 6. The procedure that finds the constraint

Nygard gives this procedure in Sec. 8.2, p. 163–164. Each step names its own signal.

1. **Consider the system as a whole first.** Signal: a capacity question about the product,
   not about one component.
2. **Find the driving variables.** Search for the things outside your control, such as user
   demand, the clock, and the calendar. Signal: you cannot set the value of the variable.
3. **Determine the following variables that correlate with each driving variable.** Use load
   testing, stress testing, observation of production, and data analysis. Signal: a
   correlation coefficient of 0.8 or more.
4. **Decompose the whole system into layers or subsystems, and repeat steps 2 and 3.** Signal:
   a variable changes role between two layers.
5. **Find the point where the correlation breaks.** Run the correlation analysis with variable
   windows. Signal: demand continues to rise and the servicing of that demand decreases.
6. **Elevate the constraint.** Increase the resource that the constraining variable needs, or
   decrease your use of that resource (Sec. 8.2, p. 164).
7. **Repeat the procedure.** An improved constraint moves the constraint to another resource
   (Sec. 8.2, p. 163).

### The knee

The break in step 5 is the knee in a load-testing chart. **A rapid decrease at the knee means
that the system already reached a constraint** (Sec. 8.2, p. 164). Nygard uses the same word
for the point where request threads exceed pooled resources (Sec. 9.1, p. 176).

`04-load-testing.md` gives the test that produces this curve. `02-capacity-antipatterns.md`
gives the designs that move the knee to the left.

---

## 7. Interrelations

**An effect in one layer surfaces as a cause in another layer** (Sec. 8.3, p. 165).

The sequence is short. Demand on one layer exceeds the capacity of that layer. The performance
of that layer degrades. The layer responds slowly, or it does not respond. Nygard states the
consequence: "Slow response is actually worse than no response" (Sec. 8.3, p. 165). The
slowdown in one layer can then trigger a cascading failure in another layer (Sec. 8.3, p. 165).

**This is why a capacity question is hard to separate from a stability question** (Sec. 8.3,
p. 165). Nygard names the retail launch of Chapter 7 as the example, where a severe capacity
problem led directly to a stability problem (Sec. 8.3, p. 165). This skill finds and states the
limit. `self-healing-apis/SKILL.md` owns what the system does above the limit.

---

## 8. Scalability: the two modes

| Mode | What it is | What it gives | What it costs |
|---|---|---|---|
| Horizontal, "getting wide" | Add servers behind a load balancer or a virtual IP address (Sec. 8.4, p. 165) | Nearly linear growth with a shared-nothing design. Doubling the servers should almost double the capacity | The added load can overwhelm another service (Sec. 8.4, p. 165) |
| Vertical, "getting big" | Add CPUs and RAM to the existing servers (Sec. 8.4, p. 165–166) | More power per machine, with no change to the software | A higher initial cost, and a forklift upgrade once the chassis is full (Sec. 8.4, p. 166) |

**Perfect horizontal scaling needs a server that runs without knowing anything about any other
server** (Sec. 8.4, p. 165). Nygard calls this a shared-nothing architecture. A cluster
architecture also scales horizontally, at somewhat less than linear benefit, because cluster
management has an overhead. Web servers and Ruby on Rails servers are perfectly horizontally
scalable, and J2EE application servers scale through clustering (Sec. 8.4, p. 165).

**A database server is the counter-example.** It becomes unwieldy when you try to cluster three
or more redundant servers. Nygard prefers a strong pair with failover (Sec. 8.4, p. 165). A
horizontal architecture also lets you spend incrementally, instead of committing capital to a
few large chassis at the start (Sec. 8.4, p. 166).

`system-design/references/03-scaling-ladder.md` gives the order of the steps. This file
gives the measured number that names the step that is due.

---

## 9. The myths about capacity

Nygard states that some false beliefs are harmless and that some cost a company millions of
dollars (Sec. 8.5, p. 166). He refutes three.

### Myth 1 — "CPU is cheap" (Sec. 8.5, p. 167–169)

**The claim.** A CPU costs less than half a day of programmer time today, against several years
of a programmer's salary in the 1960s (Sec. 8.5, p. 167).

**The refutation.** The silicon is cheap and the cycles are not. Every CPU cycle consumes clock
time, and clock time is latency (Sec. 8.5, p. 167). Nygard then gives four costs.

1. **The direct cost.** An extra 250 milliseconds per transaction, across one million
   transactions per day, is an extra 69.4 hours of compute time per day. At an 80 percent load
   factor per server, you need four more servers (Sec. 8.5, p. 167–168).
2. **The nonlinear cost.** Most application servers take request-handling threads from a pool.
   A request that holds its thread longer raises the probability that the next request queues
   instead of executing at once (Sec. 8.5, p. 168).
3. **The cost in the layer above.** A web server that waits for the application server holds
   idle resources. It holds at least two sockets, memory for the request state, and memory for
   a partly buffered response. The CPU use of the application server directly drives the memory
   use of the web server (Sec. 8.5, p. 168).
4. **The breaking CPU.** Only so many chips fit in one machine. In a four-way box, the fifth
   CPU forces a new chassis. That chassis needs its own RAM, disk, network cards, cooling, and
   rack space (Sec. 8.5, p. 168). For the entry-level servers of Figure 8.3, the third CPU
   costs about 1.2 times the second CPU (Sec. 8.5, p. 168–169).

### Myth 2 — "Storage is cheap" (Sec. 8.5, p. 169–171)

**The claim.** Storage cost about a dollar per megabyte ten years earlier, and less than fifty
cents per gigabyte in January 2007 (Sec. 8.5, p. 169, footnote 3).

**The refutation.** Nygard states it in one line: "Storage is a service, not a device"
(Sec. 8.5, p. 170). Storage is the managed system of drives, interconnects, allocation,
redundancy, and backups (Sec. 8.5, p. 170).

- **Every server needs its own space.** It holds the operating system, the applications, the
  local configuration or data, the log files, and the temporary working space (Sec. 8.5,
  p. 170).
- **The server multiplier.** One gigabyte across twenty servers is 20 GB, not 1 GB (Sec. 8.5,
  p. 171).
- **The redundancy multiplier.** RAID 1 mirroring costs 100 percent overhead, which doubles the
  disk count. RAID 5 costs 20 percent (Sec. 8.5, p. 171).
- **The backup multiplier.** An extra gigabyte across twenty servers can push the backup past
  its window. That forces more tape drives in parallel, and more tapes (Sec. 8.5, p. 171).
- **The chargeback.** A gigabyte on local storage costs less than one dollar. In an enterprise,
  managed storage can be charged back at up to $7 per gigabyte (Sec. 8.5, p. 171).

### Myth 3 — "Bandwidth is cheap" (Sec. 8.5, p. 171–173)

**The claim.** Nygard states that people say this one less often and assume it just as often
(Sec. 8.5, p. 171). An OC3 connection costs $7,500 to $12,000 per month. A serious business
runs at least a pair, preferably from two carriers. A load-balanced pair gives a theoretical
maximum of 310 Mb per second for $15,000 to $24,000 per month (Sec. 8.5, p. 171).

- **The contract shape matters.** A dedicated connection gives the same bandwidth every second
  at a flat rate. A burstable connection charges a lower flat rate for the committed bandwidth,
  and charges per megabit minute above it (Sec. 8.5, p. 171, p. 173). A megabit minute is the
  excess transfer rate in megabits times the number of minutes at that level (Sec. 8.5, p. 173,
  footnote 4).
- **A faster client costs you more.** TCP/IP handshaking guarantees that a client pulls data as
  fast as it can receive it. A dial-up user connects at 44 Kbps and pulls 38 to 39 Kbps after
  the PPP overhead. Some cable modem customers receive 6 Mbps. You could serve thirteen times
  as many dial-up users as cable modem users (Sec. 8.5, p. 173).
- **The junk multiplier.** Suppose each page carries 1,024 bytes of junk. At one million pages
  per day, that is 1,024,000,000 excess bytes per day. Nygard notes that most pages carry far
  more than 1,024 unnecessary bytes (Sec. 8.5, p. 173).

### Two myths that the book predates — **Modern**

Neither book states these two rows. They follow the arithmetic of the three myths above.

| Myth | The refutation | What to state instead |
|---|---|---|
| "The cloud is elastic" | An autoscaling group has a maximum, a start delay, and an account quota | The maximum instance count, the delay before a new instance serves traffic, and the quota |
| "Egress is small" | Egress is billed per gigabyte, and it obeys the same multiplier | The response size times the request count per day, before anyone accepts the payload |

**The rule behind all five myths: always search for the multiplier effects, because they
dominate your costs (Sec. 8.6, p. 174).**

---

## 10. War stories

**Four times the demand on two-thirds of the hardware (Sec. 8.1, p. 161).** Nygard reports a
system that improved over eighteen months. It then handled four times the demand on two-thirds
of the original hardware. The improvement came entirely from software design changes. A
capacity number is a property of the design, and not only of the purchase order.

**The launch that Sec. 8.3 cites (Sec. 8.3, p. 165, with Sec. 7.1 and Sec. 7.4,
p. 147–157).** A retail site reached 10,000 active sessions at 9:05 a.m. and more than 50,000
at 9:10. It reached 250,000 at 9:30, and then it crashed. The constraint was sessions, which
consumed RAM, CPU, and replication bandwidth. Nygard uses this launch as the case where a
severe capacity problem led directly to a stability problem. `04-load-testing.md` gives the
test that did not find it.

**The 73rd CPU (Sec. 8.5, p. 169).** The chassis of a Sun Fire 6900 cost more than $200,000. A
minimal chassis for a Sun E25K cost more than a million dollars, and the E25K held 72
processors. A million dollars pays for a large amount of profiling and optimization, so prove
that you need the 73rd CPU before you commit to it.

**One gigabyte that became forty (Sec. 8.5, p. 171).** A team adds one gigabyte of data per
server. Twenty servers make that 20 GB. Data center servers boot from mirrored volumes, and
RAID 1 doubles the disk count, so the 20 GB becomes 40 GB. The same gigabyte can also push the
nightly backup past its window. One decision, three multipliers.

---

## 11. The Chapter 8 checklist

Nygard closes the chapter with seven rules (Sec. 8.6, p. 174). Each rule below is a paraphrase
and a gate that this skill checks.

1. **Search for the multiplier effects.** They dominate the cost. Section 9 gives the
   arithmetic.
2. **Understand the effect that one layer has on another.** Section 7 gives the mechanism.
3. **Improving a nonconstraint metric does not improve capacity.** Section 4 gives the reason.
4. **Do the most work when nobody waits for it.** `03-capacity-patterns.md` gives the pattern.
5. **Place a safety limit on everything.** Nygard names a timeout, a maximum memory size, and a
   maximum connection count. `02-capacity-antipatterns.md` gives the values.
6. **Protect the request-handling threads.** A blocked thread holds memory, CPU slices, and
   pooled resources (Sec. 9.1, p. 178–179).
7. **Monitor capacity continuously.** Each release can change scalability and performance.

---

## 12. How to state a capacity claim

| Forbidden phrase | State this instead |
|---|---|
| "It will scale" | The constraint, its ceiling, and the measurement that found it (Sec. 8.2, p. 163) |
| "CPU is only at 40 percent" | The constraint metric, which may not be CPU (Sec. 8.2, p. 163) |
| "We can double the load" | The measured throughput at the knee, and the current peak (Sec. 8.2, p. 164) |
| "It is fast enough" | The operation, the percentile, and the bound (Sec. 8.1, p. 162) |
| "Storage is cheap" | Bytes per record, records per day, the retention period, and the mirror overhead (Sec. 8.5, p. 170–171) |
| "Add more servers" | The shared resource that does not divide, and the measured gain per node (Sec. 8.4, p. 165) |

**A capacity claim without a number is not a claim.** Write the claim in this shape.

```
Constraint: [resource]. Driving variable: [variable]. Correlation: [0.8 to 1.0].
Capacity: [number] [unit], at workload [mix], within [bound] for [operation].
Current peak: [number] [unit], measured by [test or production date].
```

---

## 13. Where to continue

| Question | File |
|---|---|
| Which design decisions move the knee to the left? | `02-capacity-antipatterns.md` |
| How do I elevate the constraint? | `03-capacity-patterns.md` |
| How do I produce the curve and the knee? | `04-load-testing.md` |
| How do I write a plan that recomputes? | `05-intent-based-capacity-planning.md` |
| How do I spread the load, and which signal do I read? | `06-load-balancing-and-utilization.md` |
| Which named defect is this? | `capacity-defect-catalog.md` |
| Which rung of the scaling ladder is due? | `system-design/references/03-scaling-ladder.md` |
| What does the system do above the limit? | `self-healing-apis/SKILL.md` |
| Does it stay correct under concurrency? | `data-systems-design/references/hazard-catalog.md` |

The gates that use this file are in `../SKILL.md`, Section 2 and Section 3.
