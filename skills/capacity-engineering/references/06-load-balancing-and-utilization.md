# Load Balancing and Utilization

**Sources:** *Site Reliability Engineering: How Google Runs Production Systems* — Beyer, Jones,
Petoff, Murphy (O'Reilly, 2016). Ch. 19 "Load Balancing at the Frontend", p. 270–277. Ch. 20
"Load Balancing in the Datacenter", p. 278–296. Ch. 21 "Handling Overload", p. 297–298 and
p. 303–304. Supporting citations: SRE Ch. 6, p. 88, SRE Ch. 5, p. 75, and *Release It!* —
Michael T. Nygard (Pragmatic Bookshelf, 2007), Sec. 13.3, p. 233.

This file answers one question. How does a request reach a machine that can still serve it?

Bad distribution destroys capacity that you already paid for. You can reserve 1,000 CPUs in a
datacenter and use no more than about 700 of them (SRE Ch. 20, p. 281). SRE states that load
balancing has no magic bullet (Ch. 21, p. 312). **This file stops where the system rejects a
request.** Read `self-healing-apis/references/05-overload-and-load-shedding.md` for the
reject path. That file owns client throttling, criticality, retry budgets, and the Circuit Breaker.

---

## 1. Decide the level before you decide the method

Optimal distribution has no single answer. It depends on three factors (SRE Ch. 19, p. 271).
**Inside one datacenter, treat every machine as equally distant from the user** (Ch. 19, p. 271).

| Factor | The two choices | What it changes |
|---|---|---|
| Hierarchical level | Global, or local to one datacenter | Whether distance matters at all |
| Technical level | Hardware, or software | Cost, and the layer that holds state |
| Nature of the traffic | Latency-bound, or throughput-bound | The link that the request should take |

**War story — search against video upload (SRE Ch. 19, p. 271).** Two request classes in the same
company need opposite routing. A search request is latency-bound, so it goes to the nearest
available datacenter as measured in round-trip time. A video upload is throughput-bound, because
the user accepts a slow transfer but wants it to succeed on the first attempt. Google routes the
upload through a different link, possibly one that is underused. Classify the traffic first.

---

## 2. Frontend distribution with DNS

DNS is the first layer of load balancing. It acts before the client sends an HTTP request (SRE
Ch. 19, p. 272). The reply carries several A or AAAA records, and the client selects one. **DNS
gives you no control over the client.** The records attract about the same traffic each, whatever
the location or the capacity behind them (Ch. 19, p. 272). SRV records could carry weights and
priorities. HTTP has not adopted SRV records (Ch. 19, p. 272).

| Constraint | The number or the rule | Source |
|---|---|---|
| Reply size | Every reply should fit in the 512-byte limit of RFC 1035 | SRE Ch. 19, p. 274 |
| Propagation floor | A low TTL is required, because you cannot flush a resolver cache | SRE Ch. 19, p. 273–274 |
| TTL compliance | Not every resolver respects the TTL that you set | SRE Ch. 19, p. 274 |
| Caching effect | One authoritative reply can reach one user or several thousand | SRE Ch. 19, p. 273 |
| Client identity | The nameserver sees the resolver address, not the user address | SRE Ch. 19, p. 273 |

**The 512-byte limit bounds the address count in one reply.** SRE states that the count is almost
certainly smaller than the server count (Ch. 19, p. 274). DNS selects a site. Another layer selects
a machine. **"Best location" is not "closest location"** (Ch. 19, p. 274). The selected datacenter
must hold enough capacity for the users who receive that reply. Connect the nameserver to the
systems that track traffic, capacity, and infrastructure state.

**Two mitigations for the resolver problem** (SRE Ch. 19, p. 273). Keep an updated list of resolvers
with the approximate user count behind each one, and estimate the geographic spread of those users.
The EDNS0 extension carries the client subnet in the query. EDNS0 is not yet the official standard.

**War story — the national ISP with one nameserver site (SRE Ch. 19, p. 273).** A large
national ISP runs the nameservers for its whole network from one datacenter. It holds network
interconnects in each metropolitan area. Its resolvers receive the address that suits that one
datacenter, although better network paths exist for every user. The resolver location is a poor
proxy for the user location. This case is the argument for EDNS0.

**DNS round robin is not a load balancer.** DNS holds no health data and keeps returning addresses
for dead servers (Release It! Sec. 13.3, p. 233). A long-lived Java caller caches the first address
forever, which defeats the distribution (Sec. 13.3, p. 233). **Signal to add a layer below DNS:** a
dead server keeps receiving connections. See C-32 in `capacity-defect-catalog.md`.

---

## 3. Frontend distribution with a virtual address

A virtual IP address (VIP) belongs to no single interface, and many devices share it (SRE Ch. 19,
p. 274). **A VIP hides the fleet and makes maintenance invisible** (Ch. 19, p. 274–275). You can
upgrade a machine or add machines without the user knowing. **Do not select the least loaded
backend per packet.** This logic breaks quickly for a stateful protocol. Such a protocol must
use the same backend for the whole request (Ch. 19, p. 275).

| Mapping method | What it gives | What it costs | Source |
|---|---|---|---|
| Connection ID, `id(packet) mod N` | No state on the balancer | Removing one backend remaps almost every packet | SRE Ch. 19, p. 275 |
| Consistent hashing | A stable map when backends join or leave | More work per packet | SRE Ch. 19, p. 275 |
| NAT | Simple forwarding | An entry per connection, so no stateless fallback exists | SRE Ch. 19, p. 276 |
| Direct Server Response | Bandwidth savings, and no state on the balancer | Every balancer and backend needs one broadcast domain | SRE Ch. 19, p. 276 |
| GRE encapsulation | The balancer and the backend can be far apart | 24 bytes of overhead, and possible fragmentation | SRE Ch. 19, p. 276–277 |

**Use simple connection tracking normally, and change to consistent hashing under pressure** (SRE
Ch. 19, p. 276). A denial of service attack is the example that the book gives. **Signal that GRE
overhead hurts you:** fragmented packets inside the datacenter. The remedy is a larger MTU there
(Ch. 19, p. 277).

**War story — Google outgrew layer 2 forwarding (SRE Ch. 19, p. 276).** Direct Server Response
rewrites the destination MAC address, so the backend replies straight to the sender. It saves
bandwidth when requests are small and replies are large, and it needs no state on the balancer. It
also requires every balancer and every backend to sit in one broadcast domain, and that requirement
does not scale. Google changed to GRE encapsulation, which accepts 24 bytes of overhead for reach.

---

## 4. Datacenter distribution: the ideal case

**The ideal case: at every moment the least loaded and the most loaded backend task consume exactly
the same CPU** (SRE Ch. 20, p. 279). Real systems never reach it. **Wasted capacity is the sum of
the CPU gap between the most loaded task and every other task** (Ch. 20, p. 281). Wasted means
reserved but unused.

| Fact | The number | Source |
|---|---|---|
| Minimum backend processes per service | At least 3 | SRE Ch. 20, p. 278 |
| Reason for that minimum | Fewer processes lose 50 percent or more of capacity per machine failure | SRE Ch. 20, p. 278 |
| Service size | Typically 100 to 1,000 processes, and more than 10,000 for the largest | SRE Ch. 20, p. 278 |
| Usable share under poor balancing | About 700 CPUs from 1,000 reserved | SRE Ch. 20, p. 281 |

**Traffic reaches a datacenter only until the most loaded task reaches its limit** (SRE Ch. 20,
p. 279). The most loaded task sets the capacity of the whole site, not the average task. **Signal
that the spread costs you money:** the site refuses traffic while most tasks stay idle.

---

## 5. Identify a bad task: flow control and lame duck state

A client must know which backends can serve. SRE gives three client-visible states (Ch. 20, p. 282).

| State | What it means | What the client does |
|---|---|---|
| Healthy | The task initialized correctly and processes requests | Send requests |
| Refusing connections | The task is unresponsive, starting, stopping, or abnormal | Send nothing |
| Lame duck | The task listens and can serve, and asks clients to stop sending | Drain, then send elsewhere |

### Flow control

**Flow control counts active requests per connection and treats the backend as unhealthy at a
configured limit** (SRE Ch. 20, p. 281). For most backends, 100 is a reasonable limit. It also acts
as a simple form of load balancing. **Do not use an active-request cap as your only health signal.**
It does not solve the underlying problem. That problem is knowing whether a task is truly
unhealthy or only slow (Ch. 20, p. 282).

**War story — the flow control limit that hid every backend (SRE Ch. 20, p. 281–282).** The
default limit is 100 active requests. Backends that served very long-lived requests never
returned fast enough. Clients marked each backend unhealthy in turn. Google reports that the default limit
backfired and made all backend tasks unreachable, with requests blocked in the clients until they
timed out. Raising the limit removes the symptom and not the cause.

### Lame duck state

**A backend in lame duck state still serves, and explicitly asks its clients to send elsewhere**
(SRE Ch. 20, p. 282). The task broadcasts the change to every active client. Inactive clients learn
it from periodic UDP health checks. Propagation takes 1 or 2 round trips in the typical case.

The clean shutdown procedure has five steps (SRE Ch. 20, p. 282–283).

1. The job scheduler sends `SIGTERM` to the backend task.
2. The `SIGTERM` handler calls the RPC API that enters lame duck state.
3. Requests that started before the change execute normally.
4. The count of active requests falls to zero as responses return.
5. The task exits cleanly, or the scheduler kills it, after a configured interval.

**Set the drain interval large enough for every typical request to finish.** SRE gives a rule
of thumb. It is 10 seconds to 150 seconds, depending on client complexity (Ch. 20, p. 283).
See C-46 in `capacity-defect-catalog.md`. The exact value
is a local decision that your latency distribution sets. **Accept connections during a long
initialization, then signal readiness explicitly** (Ch. 20, p. 283).

**Prewarm a restarted task inside lame duck state** (SRE Ch. 20, p. 293). A restarted task needs
significantly more resources for a few minutes. Prewarming triggers the dynamic optimizations of a
runtime such as Java. **Signal to add lame duck state:** a deploy serves errors to the requests that
happened to be active.

---

## 6. Limit the connection pool with subsetting

**Subsetting limits the pool of backend tasks that one client task uses** (SRE Ch. 20, p. 283). One
client must not connect to very many backends. One backend must not receive connections from very
many clients. **An unbounded connection pool spends memory and CPU for little gain** (Ch. 20,
p. 284). Health checking can cost more than serving when many client tasks each send a low rate of
requests (SRE Ch. 21, p. 310). See C-35 in `capacity-defect-catalog.md`.

| Decision | The rule | Source |
|---|---|---|
| Subset size | Typically 20 to 100 backend tasks. The right size depends on the service | SRE Ch. 20, p. 284 |
| Use a larger size when | The client count is much smaller than the backend count, or clients are bursty with large fan-out | SRE Ch. 20, p. 284 |
| Uniformity requirement | If subsetting overloads one backend by 10 percent, overprovision the whole set by 10 percent | SRE Ch. 20, p. 284 |
| Resize requirement | The algorithm must absorb count changes with little connection churn | SRE Ch. 20, p. 285 |

SRE defines toil as manual, repetitive work that scales with the service (Ch. 5, p. 75). A subset
size that one engineer retunes by hand for every new service meets that definition.

| Algorithm | How it selects | Spread it achieves | Source |
|---|---|---|---|
| Random subsetting | Each client shuffles the list once and takes the head of it | 50 to 150 percent of average at a 10 percent subset size | SRE Ch. 20, p. 285–287 |
| Deterministic subsetting | Clients form rounds, and each round shuffles with its own seed | Every backend receives the same connection count in the example | SRE Ch. 20, p. 287–289 |

**War story — the random subsetting simulation (SRE Ch. 20, p. 285–287).** Google simulated 300
clients against 300 backends. At a 30 percent subset size, each client used 90 backends. The least
loaded backend held 57 connections against an average of 90, and the most loaded held 109. At a 10
percent subset size the range widened to 15 and 45 connections, which is 50 percent and 150 percent
of the average. Even spread would demand subsets of about 75 percent, which the book calls
impractical. Deterministic subsetting gives every backend the same count (Ch. 20, p. 289).

The deterministic algorithm has five steps (SRE Ch. 20, p. 287–289).

1. Divide the backend count by the wanted subset size. Call the result the subset count.
2. Group the client tasks into rounds. Each round holds that many consecutive client tasks.
3. Seed the shuffle with the round number, so one round shares a list and other rounds do not.
4. Shuffle the backend list with that seed.
5. Take the client index modulo the subset count, and return that slice of the list.

**Rule 1. Shuffle the backend list.** Without a shuffle, a client receives consecutive backend
tasks. A gradual push that updates tasks in order can then remove a whole subset at once (SRE
Ch. 20, p. 288).

**Rule 2. Use a different seed for each round.** With one shared seed, the load of a failed backend
spreads only inside its own subset. When N backends in a subset are down, their load lands on the
remaining tasks of that subset, and the effect compounds (SRE Ch. 20, p. 288).

**Result to expect:** in most cases the client count per backend differs by at most 1 (SRE Ch. 20,
p. 289). **Signal to change from random to deterministic subsetting:** connection counts per backend
differ by more than the overprovisioning that you can afford.

---

## 7. Load balancing policies

**A policy is the mechanism that a client task uses to select a backend inside its subset** (SRE
Ch. 20, p. 290). The client decides in real time, from partial and possibly stale state.

| Policy | Spread it achieves | Failure cause | Source |
|---|---|---|---|
| Simple round robin | Up to 2 times the CPU from the least to the most loaded task | Small subsets, variable query cost, machine diversity, antagonistic neighbors | SRE Ch. 20, p. 290–293 |
| Least-loaded round robin | About 2 times, which is little better | A task that fails fast attracts traffic. This is sinkholing | SRE Ch. 20, p. 293–295 |
| Weighted round robin | The best spread of the three | Backends must report queries, errors, and utilization in every response | SRE Ch. 20, p. 296 |

### Simple round robin, and its four failure causes

Each client sends requests in rotation to every reachable backend in its subset that is not in lame
duck state. SRE reports that this was their most common approach for many years (Ch. 20, p. 290).

| Cause | Mechanism | The number the book gives |
|---|---|---|
| Small subsetting | Clients issue requests at different rates, so busy clients load their backends more | SRE Ch. 20, p. 290–291 |
| Varying query costs | Services let the most expensive request consume far more than the cheapest | 100, 1,000, or 10,000 times (Ch. 20, p. 291) |
| Machine diversity | CPUs differ, so one request is different work on different machines | 2 CPU units on a slow machine equal 0.8 on a fast one (Ch. 20, p. 292) |
| Unpredictable factors | Performance differs for reasons that static accounting cannot capture | Up to 20 percent from antagonistic neighbors (Ch. 20, p. 293) |

**War story — the Java backend with a 15 millisecond average (SRE Ch. 20, p. 291–292).** Queries
against one Java backend consumed about 15 milliseconds of CPU on average. Some required up to 10
seconds. When one task receives a 10-second query, its load spikes for seconds. A task in that state
can exhaust its memory, or stop responding entirely because of memory thrashing. Even in the normal
case, other requests suffer from the competition for resources.

**Cap the work per request at the interface** (SRE Ch. 20, p. 291). The book changes "return every
email that user XYZ received in the last day" into "return the most recent 100 emails or fewer". The
fix has a stated price. Every client changes, and a client that concatenates pages naively produces
an inconsistent view that repeats or skips messages. See C-33 in `capacity-defect-catalog.md`.

**War story — machine diversity and the birth of GCU (SRE Ch. 20, p. 292).** Datacenters hold
CPUs of different performance. The same request is then a different amount of work on
different machines.
The remedy is to scale the CPU reservation by machine type. That remedy took significant effort,
because the job scheduler had to learn resource equivalence from average machine performance sampled
across services. Google created Google Compute Units, a virtual unit for a CPU rate.

**War story — antagonistic neighbors (SRE Ch. 20, p. 293).** Unrelated processes that other teams
run have degraded a process by up to 20 percent. The competition is for shared resources such as
memory-cache space and bandwidth. The latency of a backend's own outgoing requests grows because of
network competition. Its active-request count therefore grows, which can trigger more garbage
collection. Static accounting cannot capture this, so the policy must adapt.

### Least-loaded round robin

The procedure has five steps (SRE Ch. 20, p. 293–295).

1. Each client keeps a table of active-request counts for every backend in its subset.
2. Filter the subset to the tasks that hold the minimum count.
3. Apply round robin across that filtered set.
4. Increment the count on dispatch, and decrement it on completion.
5. Count recent errors as if they were active requests.

**Step 2 is required.** Without the filter, the policy may fail to spread requests well enough, and
some available backend tasks then stay unused (SRE Ch. 20, p. 293–295). **The active-request count
is a weak proxy for load** (Ch. 20, p. 295). Requests that wait on I/O make a fast machine look as
loaded as a slow one. Each client also sees only its own requests.

**War story — sinkholing, learned the hard way (SRE Ch. 20, p. 295).** A seriously unhealthy task
can serve 100 percent errors at very low latency. Returning an unhealthy answer is often much faster
than processing a request. Clients read the low active-request count as availability and send that
task a very large amount of traffic. The remedy is one policy change. Count recent errors as if they
were active requests. See C-34 in `capacity-defect-catalog.md`.

### Weighted round robin

**Utilization here is typically CPU usage** (SRE Ch. 20, p. 296). This policy replaces the client's
inference with the backend's own measurement. The procedure has five steps (Ch. 20, p. 296).

1. Each client keeps a capability score for every backend in its subset.
2. Each backend reports its current queries per second, errors per second, and utilization.
3. The backend includes that report in every response, health-check responses included.
4. The client adjusts the score from the successful requests served and their utilization cost.
5. A failed request applies a penalty that affects later decisions.

**War story — the weighted round robin change (SRE Ch. 20, p. 296, Figure 20-6).** Google
plotted CPU rates for a random sample of backend tasks. It plotted them around the moment
their clients changed from least-loaded to weighted round robin. The spread from the least loaded to the most loaded task
decreased drastically. This is the empirical case for backend-reported utilization.

---

## 8. What the spread costs, and which signal this file owns

**`self-healing-apis` owns the per-task utilization signal.** Read
`self-healing-apis/references/05-overload-and-load-shedding.md`, Section 1 and Section 6.
That file holds the CPU utilization signal, the executor load average, memory pressure, the
criticality thresholds, and the per-task invariant above the provisioned rate. Each of those
drives a reject decision. This file does not restate them.

**This file owns one number: what bad spread costs you.**

| Signal | What it means | Threshold | Source |
|---|---|---|---|
| Wasted capacity | The CPU gap between the most loaded task and every other task, summed | 1,000 reserved CPUs can yield about 700 usable | SRE Ch. 20, p. 281 |

**Saturation is one of the four golden signals** (SRE Ch. 6, p. 88). Latency, traffic, errors,
and saturation are the four.

**Do not model capacity in queries per second.** Do not model it in a static feature of a
request either. SRE states that a moving target makes a poor metric (Ch. 21, p. 298). Measure
capacity in available resources instead, such as CPU cores and memory (SRE Ch. 21, p. 298).
C-36 in `capacity-defect-catalog.md` holds the signature and the fix.

**No universal utilization threshold appears in these chapters.** The threshold is a local
decision. Derive it from your own load test. Read `04-load-testing.md` for the test that
produces the number, and `01-defining-capacity.md` for the definition of the constraint.

---

## 9. Where this file stops, and when it does not apply

This file answers which backend receives a request. Read
`self-healing-apis/references/05-overload-and-load-shedding.md` when every backend rejects.
Read `05-intent-based-capacity-planning.md` for the machine count of the next quarter. Read
`01-defining-capacity.md` for the constraint, `04-load-testing.md` for the test that measures it,
`03-capacity-patterns.md` for the pattern that elevates it, and `capacity-defect-catalog.md` for the
named defects. `self-healing-apis` owns the Circuit Breaker and the retry budget. `slo-engineering`
owns the response-time objective and the error budget.

State "not applicable" and stop, in these cases.

- The service runs as one process, and no client selects between backends.
- The change touches no routing rule, no connection pool, and no health check.
- The traffic comes from one internal caller at a known, fixed rate.
- A managed load balancer owns the policy, and the policy is not configurable.

**Do not add subsetting to a service with three backend tasks.** Subsetting bounds a connection
count that is already small. Add lame duck state instead, because it pays at every deploy.

**Do not report a threshold that this file does not give.** SRE gives 100 active requests as a
reasonable flow control limit. It gives 10 to 150 seconds for the lame duck drain. It gives a
subset size of 20 to 100 tasks, and at least 3 backend processes per service. It gives no
universal utilization threshold. That number is a local decision.
