---
name: capacity-engineering
description: Finds the resource that reaches its limit first. Proves it with a load test that resembles real traffic. Plans for the load that was measured. Use this skill for capacity planning, a load test, a stress test, or a bottleneck hunt. Use it for a question about headroom, saturation, or throughput. Use it for utilization, queue depth, thread pool size, or connection pool contention. Use it for cache hit rate, precomputed content, garbage collector tuning, or session timeout. Use it for N+1 queries, table scans, or a launch traffic forecast. Use it for load balancing policy, subsetting, and server count. The content comes from Release It! (Nygard) and Site Reliability Engineering (Beyer and others).
---

# CAPACITY ENGINEERING

**ROLE:** You are the engineer who measures a running system. You do not guess its limit.

**CORE FUNCTION:** You receive a capacity claim. You return one named constraint, the
measurement that proves it, and the load that the system holds.

This skill is a **knowledge and gate skill**. It runs no workflow. `planner`,
`detail-planning`, `implement`, `verify`, and `code-review` consult it.

Nygard states the premise in one sentence. "In every system, exactly one constraint
determines the system's capacity." (Release It! Sec. 8.2, p. 163.)

## The five gates

Every output of this skill passes these five gates. Each gate gives a pass or a fail.

1. **Constraint gate.** Name exactly one constraint. Give the measurement that proves it.
2. **Evidence gate.** Give a load test with noise and think time, or a production measurement.
3. **Multiplier gate.** Multiply every per-unit cost by the production volume before you accept it.
4. **Safety limit gate.** Every pool, queue, cache, session, response, and call has a maximum and a timeout.
5. **Recompute gate.** The plan names its inputs and the trigger that regenerates it.

---

# 1. WHEN TO USE THIS SKILL

Use this skill when the work touches any row of Table B. Use `system-design` when no system
exists to measure. Use this skill when a number exists, or when a test can produce one.

## Table A — Which skill answers which question

| Skill | Question it answers | Time |
|---|---|---|
| `system-design` | How large will it be? | Before the code exists |
| `capacity-engineering` (this skill) | Which resource reaches its limit first? | After the code runs |
| `data-systems-design` | Does it stay correct under concurrency? | Any time |
| `self-healing-apis` | What does the system do when it is over the limit? | At the limit |
| `reverse-branching` | Which release caused the regression, and how do we undo it? | After a change |

## Table B — Activation triggers

| Trigger area | Words and artifacts that appear |
|---|---|
| A capacity question | capacity, headroom, saturation, "how many servers", "will it hold" |
| A test | load test, stress test, soak test, virtual users. Tool names are **Modern**: k6, JMeter, Gatling, Locust |
| A limit | thread pool, connection pool, `maxWait`, `max_connections`, semaphore, queue depth |
| A per-request cost | template render, serialization, response size, compression, N+1, table scan |
| Memory | cache size, eviction, hit rate, heap, garbage collection, out of memory |
| A session | session timeout, session store, cookie size, token size |
| Distribution | load balancer, round robin, subsetting, sticky session. **Modern**: autoscaling |
| A plan | forecast, quarterly allocation, resource request, launch traffic, peak season |
| A measurement | p99, utilization, CPU seconds, queue wait, knee, throughput curve |

If no row applies, this skill stays silent. Section 12 gives the rule.

---

# 2. THE FOUR NUMBERS AND THE ONE CONSTRAINT

Teams mix four different numbers. Separate them before you discuss any target.

## Table C — The four numbers

| Number | Definition | Unit | What it does not tell you |
|---|---|---|---|
| Performance | How fast the system processes one transaction (Sec. 8.1, p. 161) | Milliseconds per transaction | Nothing about throughput. Faster work outside the bottleneck adds no throughput (p. 162) |
| Throughput | The number of transactions in a time span (Sec. 8.1, p. 162) | Transactions per second | Nothing about the response time of one user |
| Capacity | The maximum throughput at an acceptable response time, for a stated workload (Sec. 8.1, p. 162) | Transactions per second, plus a workload and a bound | Nothing, unless both the workload and the bound are stated |
| Utilization | A following variable such as CPU, memory, or disk (Sec. 8.2, p. 164) | Percent | Nothing about capacity, unless it is the constraint |

**Exactly one constraint sets the capacity of a system** (Sec. 8.2, p. 163). Every other part
queues work or discards it. Any nonconstraint metric is useless for a projection (p. 163).

**Acceptable response time is a business judgment.** Nygard gives three examples (Sec. 8.1,
p. 162). An ecommerce retailer loses customers past two seconds. A financial exchange needs
milliseconds. A travel system allows 500 ms for a search and 30 seconds for a reservation.

Capacity is also a revenue measure (Sec. 8.6, p. 174). State what the limit costs in money.

---

# 3. FIND THE CONSTRAINT

Run these seven steps. They come from Nygard, Sec. 8.2, p. 163–164.

1. Consider the whole system first. Do not start inside one component.
2. Name the driving variables. A driving variable sits outside your control.
3. Measure the following variables that move with each driving variable. Target a correlation
   coefficient between 0.8 and 1.0 (Sec. 8.2, p. 164, footnote 2).
4. Decompose the system into layers. Repeat steps 2 and 3 inside each layer.
5. Find the knee. The knee is the load where the correlation stops (Sec. 8.2, p. 164).
6. Elevate the constraint. Add the resource, or reduce your use of the resource.
7. Repeat the procedure. An elevated constraint moves the constraint to another resource.

## Table D — Driving variables and following variables

| Kind | Definition | Examples | Can you set it? |
|---|---|---|---|
| Driving variable | A variable outside your control that creates demand (Sec. 8.2, p. 163) | Page requests per second, user demand, the clock, the calendar | No |
| Following variable | A variable that moves in response to a driving variable (Sec. 8.2, p. 164) | CPU usage, free memory, disk rate, page swap rate, network bandwidth | Only by a design change |

**Every directly measurable performance statistic is a following variable** (Sec. 8.2,
p. 164). One following variable drives another. Database disk activity drives application
server response time. That response time drives web server memory use (Sec. 8.5, p. 168).

## Table E — What the evidence must be

| Claim in the plan | Evidence that this skill accepts | Evidence that it rejects |
|---|---|---|
| "The database is the constraint" | The knee in a throughput curve, plus a correlation of 0.8 or more | A screenshot of one busy dashboard |
| "We have headroom" | The measured throughput at the knee, and the current peak against it | Average CPU below 50 percent |
| "This release did not change capacity" | The same load test against both builds | The release notes |
| "The cache helps" | The measured hit rate (Sec. 10.2, p. 208) | The presence of the cache |
| "It scales horizontally" | A measured second node, and the shared resource named | A shared-nothing claim with no test |

---

# 4. THE LOAD TEST THAT FINDS REAL FAULTS

The launch in Chapter 7 passed three months of polite tests. The site then failed thirty
minutes after launch (Sec. 7.1, p. 148, and Sec. 7.4, p. 155–157). Nobody scripted the traffic
that caused the failure.

## Table F — Which test to run

| Test | Question it answers | What it cannot prove | Source |
|---|---|---|---|
| Load test | What is the throughput at an acceptable response time? | Behavior above the limit | Release It! Sec. 7.3, p. 152 |
| Stress test | Where is the limit, and does the component degrade or fail? | Behavior at normal load | SRE Ch. 17, p. 228 |
| Soak test | Does a resource leak over hours? | Peak behavior | **Modern** |
| Canary | Does the new build change the variance under real traffic? | The size of the fault before release | SRE Ch. 17, p. 228 |
| Production probe | Is the deployed configuration equivalent to the tested one? | Any capacity number | SRE Ch. 17, p. 243 |

SRE states the reason for the stress test. Components do not degrade gracefully past a point,
and instead fail catastrophically (Ch. 17, p. 228). `reverse-branching` owns canary progression.

## Table G — What a realistic load test must contain

| Element | The rule | Source |
|---|---|---|
| Session mix | Model grazers, searchers, and buyers separately | Sec. 7.3, p. 152 |
| Page depth | A checkout session reaches twelve pages, a scan session reaches seven | Sec. 7.3, p. 152 |
| Conversion share | Send about 4 percent of virtual users through checkout | Sec. 7.3, p. 152 |
| Think time | Use the measured delay between clicks, not zero | Sec. 7.3, p. 152 |
| Cookieless clients | Include clients that never return a cookie | Sec. 7.4, p. 156 |
| Repeated URL floods | Include one client that requests the same URL many times per second | Sec. 7.4, p. 156 |
| Dead links | Include requests for URLs that return 404 | Sec. 7.4, p. 156 |
| Harshness | Make the mix somewhat harsher than the real traffic | Sec. 7.3, p. 152 |
| Independence | Send the next request on a schedule, not after the previous response | **Modern** |
| Data volume | Run against production-sized data, scrubbed | Sec. 9.7, p. 194–195 |
| Topology | Match the production topology, not only the configuration values | Sec. 14.1, p. 241 |

**Verify the load generators themselves.** Nygard reports a hacked generator farm and a capped
network subnet. Both produced a false ceiling (Sec. 7.3, p. 152–155).

**A test that stops at the target proves nothing about failure.** Continue one run until a
component fails. Record the knee, the first component to fail, and the recovery time. Nygard's
site needed nearly an hour to serve pages again after full saturation (Sec. 7.6, p. 158).

---

# 5. THE MULTIPLIER PASS

Multiply every per-request and per-server cost by the production volume, before anyone accepts
the design. Find every multiplier effect, because multipliers dominate the cost (Sec. 8.6, p. 174).

## Table H — The three myths, and the refutation

| Myth | The refutation | The number |
|---|---|---|
| "CPU is cheap" (Sec. 8.5, p. 167) | Every cycle consumes clock time, and clock time is latency | 250 ms per transaction across 1 million transactions per day is 69.4 hours of compute time per day, which needs four more servers at an 80 percent load factor (p. 167–168) |
| "Storage is cheap" (Sec. 8.5, p. 169) | Storage is a service, not a device | Mirroring costs 100 percent overhead, RAID 5 costs 20 percent, and managed storage is charged at up to $7 per gigabyte (p. 171) |
| "Bandwidth is cheap" (Sec. 8.5, p. 171) | A burstable connection charges for every megabit minute above the committed rate | 1,024 bytes of junk on 1 million pages per day is 1,024,000,000 excess bytes per day (p. 173) |
| **Modern** "The cloud is elastic" | An autoscaling group has a maximum, a start delay, and a quota | State the maximum instance count, the delay before a new instance serves traffic, and the account quota |
| **Modern** "Egress is small" | Egress is billed per gigabyte, and it obeys the same multiplier | Multiply the response size by the request count per day before you accept a payload |

Run the pass in four steps.

1. List every per-unit cost that the change adds: bytes, milliseconds, connections, rows, calls.
2. Name the multiplier: requests per day, instances, machines, or the retention period.
3. Compute the product. Record it in the Capacity Record.
4. Accept the cost, or remove the work from the request path.

**Do the most work when nobody waits for it** (Sec. 8.6, p. 174).

---

# 6. ELEVATE THE CONSTRAINT: THE CAPACITY PATTERNS

**A pattern applied to a nonconstraint gives no capacity** (Sec. 8.2, p. 163). Select the
pattern from the constraint that Section 3 named.

## Table I — Pattern selection

| Pattern | The signal that selects it | The price | The measurement that confirms it |
|---|---|---|---|
| Pool connections (Sec. 10.1, p. 206) | Connection setup appears in the transaction time. Setup takes 400 to 500 ms | An undersized pool creates contention. An oversized pool stresses the database | The wait time for a connection, and the high-water mark |
| Use caching carefully (Sec. 10.2, p. 208) | The same object is read often and changes rarely | Memory that requests need. A stale read. A flush storm | The hit rate per cached class, and the maximum memory in use |
| Precompute content (Sec. 10.3, p. 210) | The content changes far less often than the page is served | A publish step, and a cold start after a restart | The ratio of renders per day to changes per day |
| Tune the garbage collector (Sec. 10.4, p. 214) | The collector uses about 10 percent of runtime at production volume | The tuning is release-specific and you must repeat it | Collector time as a percentage, target 2 percent or less |

## Table J — Caching carefully means five decisions

| Decision | The rule | Source |
|---|---|---|
| Maximum size | The maximum memory of all application caches is configurable | Sec. 10.2, p. 208 |
| Hit rate | Measure the hit rate. A low hit rate can be slower than no cache | Sec. 10.2, p. 208 |
| What not to cache | Do not cache a trivial object, or an object used once in the life of a server | Sec. 10.2, p. 208 |
| Invalidation | Every cache removes an item when its source data changes | Sec. 10.2, p. 209 |
| Flush rate | Limit how often a flush can start. Frequent flushes create self-denial | Sec. 10.2, p. 209 |

Point-to-point invalidation works at ten or twelve application servers. At hundreds of servers
it stops working. Use a queue or multicast, and prevent every server from reloading the item
at the same moment (Sec. 10.2, p. 209). `data-systems-design` owns invalidation correctness.

---

# 7. SAFETY LIMITS

Nygard states the general rule. Place safety limits on everything, and protect the
request-handling threads (Sec. 8.6, p. 174).

## Table K — Safety limits, with the book's numbers

| Limit | The rule | Source |
|---|---|---|
| Pool size against thread count | Make the resource pool size equal to the number of request threads | Sec. 9.1, p. 176 |
| Pool wait | Replace an unbounded wait with a bounded wait, and handle the failure in application code | Sec. 9.1, p. 178 |
| Pool total across the fleet | 20 machines by 5 instances by 50 connections is 5,000 database connections, which is 5 GB of database server memory | Sec. 9.1, p. 177 |
| Pool total under failover | One node of the cluster must serve every query and every connection | Sec. 9.1, p. 179 |
| Contention at regular peak | There is no contention at regular peak, which is a typical day outside the peak season | Sec. 9.1, p. 179 |
| Session timeout | Set it to one standard deviation past the average think time | Sec. 9.4, p. 185 |
| Session timeout, measured | About 10 minutes for retail, 5 for a media gateway, up to 20 for travel | Sec. 9.4, p. 185 |
| Cookie size | A cookie carries an identifier, which is less than about 100 bytes | Sec. 9.10, p. 202 |
| Garbage collector | An untuned application uses about 10 percent of runtime. Reduce it to 2 percent or less | Sec. 10.4, p. 214 |
| Active-request limit per backend | 100 is a reasonable limit for most backends | SRE Ch. 20, p. 281 |
| Drain interval before exit | 10 to 150 seconds, by client complexity | SRE Ch. 20, p. 283 |
| Subset size | 20 to 100 backend tasks per client | SRE Ch. 20, p. 284 |
| Minimum backend count | At least three tasks, because two means a 50 percent capacity loss per machine | SRE Ch. 20, p. 278 |

`self-healing-apis` states the limit that a task must hold above its provisioned rate. This
skill does not restate it. Read
`self-healing-apis/references/05-overload-and-load-shedding.md`, Section 6.

**A timeout with no handled failure path is not a safety limit.** The pool returns null or
throws when the bounded wait expires. The application code must handle that result (p. 178).

**Every pool needs three runtime metrics** (Sec. 9.1, p. 179). Record how often callers block,
the high-water mark since start, and the count created and destroyed. Poll them on a schedule.

---

# 8. THE PLAN THAT RECOMPUTES

Encode the requirement, not the allocation. SRE states the motto in one line. "Specify the
requirements, not the implementation." (SRE Ch. 18, p. 253.)

## Table L — The four levels of intent

| Level | The statement | Degrees of freedom | Use it when |
|---|---|---|---|
| 1 | "50 cores in clusters X, Y, and Z" | None | A hard placement rule exists |
| 2 | "A 50-core footprint in any 3 clusters in region YYY" | Location | The region is fixed |
| 3 | "Meet the demand in each region with N+2 redundancy" | Location and size | **Default.** SRE reports the best result here (Ch. 18, p. 254) |
| 4 | "Run the service at 5 nines of reliability" | Location, size, and redundancy | A service with a measured performance model (Ch. 18, p. 254) |

The plan needs four inputs before it can regenerate itself. The inputs are performance data,
the demand forecast, the resource supply, and the resource pricing (SRE Ch. 18, p. 257–258).

**Performance metrics are the glue between dependencies** (SRE Ch. 18, p. 255). A load test
produces them. This is where Section 4 feeds Section 8.

**Record the prioritization before the shortfall arrives.** State which requirements you
sacrifice first. Intent-based planning makes that decision open and consistent (Ch. 18, p. 256).

**A plan that one person rebuilds by hand every quarter is toil** (SRE Ch. 5, p. 75). It is
manual, repetitive, automatable, and it scales with the service.

---

# 9. LOAD DISTRIBUTION AND UTILIZATION SIGNALS

## Table M — Distribution method

| Method | What it gives | What it costs | Source |
|---|---|---|---|
| DNS records | The first layer, before the client sends a request | No health data, cache effects, a 512-byte reply limit, and a TTL floor on propagation | SRE Ch. 19, p. 272–274 |
| Virtual IP address | One address across many machines, and invisible maintenance | Connection state, or consistent hashing under pressure | SRE Ch. 19, p. 274–276 |
| Reverse proxy | Distribution from one address to many, and caching | Squid and Apache do not track origin health | Release It! Sec. 13.3, p. 234–235 |
| Hardware load balancer | Health checks, layer 4 to 7 rules, and site failover | Five or six figures of cost, and an SSL decision | Release It! Sec. 13.3, p. 235–237 |
| Subsetting | A bounded connection count between clients and backends | A selection algorithm that must spread load evenly | SRE Ch. 20, p. 283–289 |

**DNS round robin is not a load balancer.** DNS holds no health data and keeps sending addresses
for dead servers. A long-lived Java caller caches the first address forever (Sec. 13.3, p. 233).

## Table N — Balancing policy

| Policy | Spread it achieves | Failure cause | Source |
|---|---|---|---|
| Simple round robin | Up to 2 times the CPU from the least to the most loaded task | Small subsets, variable query cost, machine diversity, antagonistic neighbors | SRE Ch. 20, p. 290–293 |
| Least-loaded round robin | About 2 times, which is little better | A fast-failing task attracts traffic. This is sinkholing | SRE Ch. 20, p. 293–295 |
| Weighted round robin | The best spread of the three | Backends must report queries, errors, and utilization in every response | SRE Ch. 20, p. 296 |

The fix for sinkholing is one policy change. Count recent errors as if they were active
requests (SRE Ch. 20, p. 295).

## Table O — Utilization signals

| Signal | What it means | Threshold | Source |
|---|---|---|---|
| Task CPU utilization | The current CPU rate divided by the reserved CPUs | Read it to place the constraint. `self-healing-apis` owns the reject threshold | SRE Ch. 21, p. 303 |
| Wasted capacity | The CPU gap between the most loaded task and every other task, summed | 1,000 reserved CPUs can yield only about 700 usable | SRE Ch. 20, p. 281 |
| Pool wait time | How long a thread waits for a pooled resource | Zero contention at regular peak | Release It! Sec. 9.1, p. 179 |
| Pool high-water mark | The most resources taken since start | Below the configured maximum | Release It! Sec. 9.1, p. 179 |
| Cache hit rate | Hits divided by lookups per cached class | A low rate means the cache buys nothing | Release It! Sec. 10.2, p. 208 |
| Collector time | The percentage of runtime in garbage collection | 2 percent or less | Release It! Sec. 10.4, p. 214 |
| Session count | Live sessions in memory | Watched at all times during the launch throttle | Release It! Sec. 7.6, p. 158 |
| Latency percentiles | p50, p95, p99, p999 | The mean hides the tail | SRE Ch. 4, p. 68 |

**Saturation is one of the four golden signals** (SRE Ch. 6, p. 88). A utilization target is
essential, because many systems degrade before 100 percent utilization (SRE Ch. 6, p. 89).

**Do not model capacity in queries per second.** Measure capacity in available resources, such
as CPU cores and memory. The cost of one request moves (SRE Ch. 21, p. 297–298). Catalog entry
C-36 carries the signature and the fix.

This skill stops at the utilization signal. `self-healing-apis` owns the response past the
threshold: throttling, criticality, retry budgets, the Circuit Breaker (Sec. 5.2, p. 115), and
load shedding. It also owns the executor load average and the memory-pressure signal, because
both drive a reject decision rather than a capacity number.

---

# 10. THE CAPACITY DEFECT SCAN (MANDATORY)

Run `references/capacity-defect-catalog.md` against the design, the difference, or the
configuration file. The catalog holds 47 named defects in nine groups, with the prefix `C-`.
Each entry gives a signature, a consequence, and a fix.

Severity: 🔴 causes data loss, corruption, or an unrecoverable state, and it blocks the merge.
🟡 causes an outage or a wrong result under load. 🔵 is a risk to operation or maintenance.

Report in this format. List only the codes that apply.

```md
### Capacity Defect Scan
| Code | Present | Evidence | Required fix |
|---|---|---|---|
| C-10 Pool smaller than the thread count | Yes | `pool.xml:14` has 8 connections and 50 threads | Set the pool to 50, then check C-14 |
| C-28 Cache with no maximum size | No | `CacheConfig` sets 256 MB | — |
```

**"Not applicable" is a valid result.** Write one line and state the reason. "Not applicable,
this change adds no query and no per-request work" is better than an invented finding.

The catalog also carries an index by symptom. Use it when you start from a production complaint
instead of a difference. `code-review` cites `C-` codes. `verify` runs the scan on a phase.

---

# 11. THE CAPACITY RECORD

This is the output document of this skill. `planner` and `engineer-workflow` write it into
`plan.md` under `## Capacity`. Every line is a commitment that `verify` can check later.

```md
## Capacity Record: [Component]

### Workload — value, measured or estimated, source
- Requests per second at peak: []
- Sessions at peak: []
- Transaction mix: []
- Think time, mean and standard deviation: []
- Data volume today, and growth per day: []
- Noise share: bots, dead links, repeat clients: []

### Response-time bound
- [operation]: [bound] at p99, because [reason]

### The constraint
- Constraint: [the one resource]
- Following variable at its ceiling: [metric]. Correlation: [0.8 to 1.0]
- Throughput at the knee: [number]. Current peak: [percentage of the knee]
- Next constraint after this one is elevated: [resource]

### Evidence
- Test type: [load / stress / soak / production measurement]. Run and date: []
- Mix, think time, and noise: [what the script contained]
- Arrival model: [fixed rate, independent of response time]
- Data volume and source: [production-sized and scrubbed / generated]
- Topology against production: [what differs, and why the result still holds]

### Multipliers — per-unit cost, multiplier, total per day, accepted or not
- Bytes per response, by requests per day: []
- Milliseconds per transaction, by transactions per day: []
- Connections per instance, by instances and machines: []
- Bytes per row, by rows per day and the retention period: []

### Safety limits — the value, and the file that holds it
- Request threads, and connection pool size: []
- Pool wait timeout, and the behavior on timeout: []
- Cache maximum memory, and flush rate limit: []
- Session timeout, and maximum response size: []
- Queue maximum depth, and behavior when full: []
- Call timeout per integration point: []

### Utilization signals — where recorded, and the alert threshold
- Constraint metric: []
- Pool wait time, and high-water mark: []
- Cache hit rate, and collector time percentage: []
- p99 latency per operation: []

### Intent
- Level (1 to 4 from Table L): []
- The requirement, stated without an allocation: []
- Inputs that regenerate this plan: [performance data, forecast, supply, pricing]
- Prioritization under a shortfall: [what is sacrificed, in order]

### Elevation plan
- [action]: gain [], cost [], start it at [x] percent of the knee

### Recompute trigger
This record is invalid when any of these change: [release, traffic pattern, forecast, supply,
pricing, data volume]. Monitor capacity continuously, because each release can affect
scalability and performance (Sec. 8.6, p. 174).

### Capacity defect scan
[The table from Section 10]
```

**Short form.** For a change class that does not need the full record, produce the defect scan
plus this one line.

```md
Capacity gate: constraint is [resource], measured at [number] [unit], current peak [number]. No new multiplier.
```

---

# 12. PROPORTIONALITY

Rigor scales with blast radius. A full Capacity Record for a copy change is a failure mode.

## Table P — Required output per change class

| Change class | Required output |
|---|---|
| A change with no new call, no new query, and no new per-request work | Nothing from this skill |
| A new endpoint or job over an existing schema | The defect scan only |
| A new pool, cache, session field, template, or background job | Defect scan, plus the multiplier pass |
| A change to a request path that a user waits on | Defect scan, multiplier pass, and the constraint statement |
| A launch, a peak season, a 10-fold growth target, or a new service | The full Capacity Record |
| An incident where the system reached a limit | The full Capacity Record, plus the knee measurement |

**This skill stays silent when no row of Table B applies.** Write `Capacity gate: not
applicable — [reason].` A gate that fires on every change teaches people to ignore it.

**Simplicity remains a goal.** One measured constraint, one stated limit, and one recorded
signal beat a forecast model that nobody regenerates.

---

# 13. LANGUAGE DISCIPLINE

**A capacity claim without a number is not a claim.** These phrases are forbidden in a design,
a review, or a plan. Each one hides a missing measurement.

| Forbidden phrase | Replace it with |
|---|---|
| "It will scale" | The constraint, its ceiling, and the measurement |
| "CPU is only at 40 percent" | The constraint metric, which may not be CPU (Sec. 8.2, p. 163) |
| "We tested it" | The mix, the think time, the noise, the data volume, and the topology |
| "Add a cache" | The size limit, the invalidation rule, the flush limit, and the expected hit rate |
| "Add more servers" | The shared resource that does not divide, and the measured gain per node |
| "The cloud is elastic" — **Modern** | The instance maximum, the delay before a new instance serves traffic, and the account quota |
| "Concurrent users" | Sessions, requests per second, or virtual users (Sec. 7.3, p. 153) |
| "It is fast enough" | The percentile, the operation, and the bound |
| "We can double the load" | The measured throughput at the knee, and the current peak |
| "Storage is cheap" | Bytes per record, records per day, the retention period, and the mirror overhead |
| "The queue absorbs it" | The maximum depth, the behavior when full, and the drain rate |

---

# 14. GLOSSARY

This skill uses one word for one meaning. Use these words in your output.

| Word | Meaning in this skill |
|---|---|
| **capacity** | The maximum throughput at an acceptable response time, for a stated workload |
| **performance** | The time to process one transaction. **throughput** is transactions per second |
| **utilization** | The used fraction of a reserved resource |
| **constraint** | The one resource that sets capacity |
| **driving variable** | A variable outside your control that creates demand |
| **following variable** | A variable that moves in response to a driving variable |
| **the knee** | The load where the correlation between the two variables stops |
| **elevate** | To add the constrained resource, or to use less of it |
| **headroom** | The gap between the current peak and the throughput at the knee |
| **noise** | Traffic that no person sends: bots, scrapers, dead links, cookieless clients |
| **think time** | The delay between one user request and the next |
| **regular peak** | A typical day outside the peak season |
| **multiplier** | The count that turns a per-unit cost into a daily cost |
| **measure** | To obtain a number from a running system. An **estimate** is not a measurement |

---

# 15. REFERENCE INDEX AND RELATED SKILLS

| Reference | Title | Sources |
|---|---|---|
| `references/01-defining-capacity.md` | Defining Capacity | Release It! Ch. 8, p. 161–174 |
| `references/02-capacity-antipatterns.md` | The Capacity Antipatterns | Release It! Ch. 9, p. 175–203 |
| `references/03-capacity-patterns.md` | The Capacity Patterns | Release It! Ch. 10, p. 204–217 |
| `references/04-load-testing.md` | A Load Test That Finds Real Faults | Release It! Ch. 7, p. 147–160 |
| `references/05-intent-based-capacity-planning.md` | Intent-Based Capacity Planning | SRE Ch. 18, p. 248–269 |
| `references/06-load-balancing-and-utilization.md` | Load Balancing and Utilization | SRE Ch. 19, p. 270–277, Ch. 20, p. 278–296, Ch. 21, p. 303–304 |
| `references/capacity-defect-catalog.md` | Capacity Defect Catalog | Cross-cutting, prefix `C-` |

## Who owns the overlapping material

| Skill | Overlap | What this skill does instead |
|---|---|---|
| `engineer-workflow` | The gate order in the pipeline | Supplies the gate content. Add a Capacity gate after the architecture gate |
| `planner` | The estimation gate in `plan.md` | Replaces an estimate with a measurement. The record supersedes the estimate row |
| `system-design` | Capacity formulas, the scaling ladder, component selection | Does not restate the ladder. Supplies the number that names the rung that is due |
| `detail-planning` | The per-phase specification | Supplies the safety limits. Each limit becomes a spec line with a file and a value |
| `implement` | Timeouts, bounded queues, jittered retries, bounded result sets | States the numbers. `implement` enforces them in code |
| `verify` | Checking a phase against its specification | Supplies the defect scan and the Capacity Record as the checklist |
| `code-review` | Reviewing a difference | Supplies the `C-` codes for its review modes |
| `data-systems-design` | Cache correctness, invalidation, partitioning, hot keys | Owns the physical limit, the pool size, and the multiplier arithmetic |
| `systems-programming` | File descriptor limits, `EMFILE`, the cost of `fsync` | Owns the pool and thread counts above the kernel |
| `security-engineering` | Denial of service, abuse limits, the scraper as a threat | Owns noise as a capacity cost, not as an attack |
| `reverse-branching` | Canary progression, revert, the expiry of a mitigation | Owns the per-release capacity measurement that triggers the revert |
| `self-healing-apis` | Throttling, criticality, retry budgets, Circuit Breaker, shedding, the per-task reject signal | Owns the balancing policy, subsetting, and the CPU utilization reading that names the constraint |
| `slo-engineering` | The response-time objective and the error budget | Consumes the objective as the response-time bound in the record |

**Two boundaries that reviewers confuse.** `data-systems-design` asks whether the design stays
correct. This skill asks which resource reaches its limit first. A correct cache with no size
limit passes the hazard scan and still causes an outage. `self-healing-apis` asks what the
system does over the limit. This skill finds that limit and states it.

---

**Attribution:** the definitions, the thresholds, and the page numbers in this skill and in its
references come from two books. The first is *Release It! Design and Deploy Production-Ready
Software* by Michael T. Nygard (Pragmatic Bookshelf, 2007). It supplies the capacity
definitions, the constraint procedure, the antipatterns, the capacity patterns, and the
load-test rules. The second is *Site Reliability Engineering: How Google Runs Production
Systems* by Beyer, Jones, Petoff, and Murphy (O'Reilly, 2016). It supplies intent-based
capacity planning, load balancing, subsetting, the utilization signals, and the stress test.
Page numbers refer to the PDF pages of the extracted editions. Content marked **Modern**
postdates both books.
