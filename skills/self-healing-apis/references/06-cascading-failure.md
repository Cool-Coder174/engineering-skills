# Cascading Failure

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 22 "Addressing Cascading Failures", pp. 313–343. *Release It!* — Michael T. Nygard
(Pragmatic Bookshelf, 2007), Sec. 4.2 "Chain Reactions", pp. 61–64, and Sec. 4.3 "Cascading
Failures", pp. 65–67. Supporting pages: Release It! Sec. 4.5, pp. 81–87. Sec. 4.8,
pp. 96–99. Sec. 4.9, pp. 100–101. Sec. 5.3, pp. 119–123.

Read this file when a failure crossed a layer boundary. Read it when one sick dependency made
a healthy caller fail.

**Two neighbors own part of this ground.** `capacity-engineering` owns the load test that
finds the breaking point and the utilization signals that trigger a shed.
`production-troubleshooting` owns the method that names the cause during the incident. This
file states how a failure grows across a boundary, and how you stop the growth.

**A cascading failure grows over time because of positive feedback.** (SRE Ch. 22, p. 313.)
One part of the system fails. Its load moves to the parts that survive. That move raises the
failure probability of the survivors. Nygard states the stake plainly. He names cascading
failures the number-one crack accelerator, and their prevention the key to resilience.
(Release It! Sec. 4.3, p. 65.)

---

## 1. Two directions, two names

Nygard separates the sideways failure from the upward failure. Keep the two names apart. The
defense differs for each one.

| Property | Chain Reaction | Cascading Failure |
|---|---|---|
| Definition | A defect in a horizontally scaled layer kills one node after another | A crack in one layer triggers a crack in a calling layer |
| Direction | Sideways, inside one layer | Upward, across a layer boundary |
| Usual trigger | A resource leak, or a crash that load causes | A drained resource pool, a blocked thread, or an unbounded retry |
| Amplifier | The load balancer moves the dead node's share to the survivors | The caller holds threads while it waits for an answer |
| The only true fix | Repair the underlying defect | Repair the caller, not only the callee |
| Defense that contains it | Bulkheads, which split one reaction into two | Timeouts and Circuit Breaker in the caller |
| Source | Sec. 4.2, pp. 61–62 | Sec. 4.3, p. 65 |

**A chain reaction in one layer can produce a cascading failure in the calling layer.**
(Release It! Sec. 4.2, p. 62.) The two failures often appear in the same incident. Name both.

**The load arithmetic decides how fast a chain reaction accelerates.** Eight servers each
carry 12.5% of the load. One server dies. Each survivor now carries about 14.3%. The absolute
increase is 1.8 points. The increase in that server's own load is about 15%. In the two-node
case the survivor's load doubles. (Release It! Sec. 4.2, p. 61.) Report both numbers. The
relative number predicts the next death.

**Nygard names four transmission paths across the gap.** Threads block on a call that never
returns. A resource pool drains because no call returns. A caller retries with no limit. A
caller closes a connection on every exception. (Release It! Sec. 4.3, pp. 65–67.)

**A slow answer is the quiet form of this failure.** Slow responses move upward from layer to
layer as a gradual cascading failure. (Release It! Sec. 4.9, p. 100.) Read
`02-stability-antipatterns.md` for the full antipattern set.

---

## 2. Cause one — server overload

**Overload is the most common cause of a cascading failure.** (SRE Ch. 22, p. 314.)

The mechanism is a failover with no headroom. A frontend in cluster A serves 1,000 queries
per second. Cluster B fails. Cluster A now receives 1,200 queries per second. Cluster A
cannot serve that rate. It exhausts a resource, and its successful rate falls **below** the
original 1,000. (SRE Ch. 22, p. 315.) The loss is larger than the share you lost. You do not
lose cluster B's traffic only. You lose part of cluster A as well.

**A local overload can become a service-wide failure in a couple of minutes.** The load
balancer and the task scheduler act that fast. (SRE Ch. 22, p. 316.)

**Reduced demand does not by itself stop a cascade.** A service healthy at 10,000 queries per
second can start a cascade at 11,000. A reduction to 9,000 will almost certainly not stop the
crashes, because capacity is now reduced too. (SRE Ch. 22, p. 320.)

**Compute the recovery rate against the healthy fraction.** If 10% of servers are healthy, the
rate must fall to about 1,000 queries per second. (SRE Ch. 22, p. 320.) This arithmetic is the
reason the book later says to admit only 1% of traffic. For quotas, criticality, throttling,
and the shed decision, read `05-overload-and-load-shedding.md`.

---

## 3. Cause two — resource exhaustion

Resource exhaustion produces higher latency, a higher error rate, or a lower-quality answer.
Those effects are intended. Something must give way as load passes capacity. (SRE Ch. 22,
p. 316.)

| Resource | Signature you can measure | Secondary effect the book names | Source |
|---|---|---|---|
| CPU | More in-flight requests, long queues, missed deadlines | Threads starve. Cache use per core falls. The watchdog kills the server. | pp. 317–318 |
| Memory | Rising heap use and a rising garbage-collection rate | The container manager evicts the task. The cache hit rate falls. | p. 318 |
| Threads | Threads wait on a lock and make no progress | Health checks fail. In extreme cases the process exhausts the process identifiers. | pp. 317–319 |
| File descriptors | New connections fail to initialize | Health checks fail. | p. 319 |
| Connection pool | Threads wait for a connection that no thread returns | Every caller of that pool blocks. | Release It! Sec. 4.3, pp. 65–66 |

**Thread starvation is a symptom of load, not proof of a broken instance.** A failed health
check under overload often means the server is busy. (SRE Ch. 22, pp. 317, 319.) See entry
`I-27` in `integration-fault-catalog.md`.

**The internal watchdog can crash a server that only needed time.** A watchdog thread wakes on
a period and kills the server when it sees no completed work. (SRE Ch. 22, p. 317, and
footnote 108, p. 343.) A missed deadline also wastes the work twice. The server answers after
the client stopped waiting, and the client may then retry. (SRE Ch. 22, pp. 317–318.)

**The "GC death spiral" is the named memory cycle.** Less CPU is available, so requests run
slower. Slower requests hold more memory. More memory raises the garbage-collection rate.
That lowers available CPU again. (SRE Ch. 22, p. 318.)

**Resource exhaustion modes feed one another.** An overloaded service shows secondary symptoms
that resemble the root cause. (SRE Ch. 22, p. 319.) Do not treat the loudest symptom as the
cause. Read `04-fault-localization.md`.

**Safe resource pools always limit the time a thread can wait for a resource.**
(Release It! Sec. 4.3, p. 66.) See entry `I-08` in `integration-fault-catalog.md`.

---

## 4. Cause three — an unavailable dependency

**Crashes snowball into a crash loop.** A few servers crash under overload. Their load moves
to the rest, which then crash too. A restarted server meets a very high request rate and
fails almost at once. (SRE Ch. 22, pp. 319–320.)

**A server can reduce capacity without crashing.** The "lame duck" state makes a server appear
unhealthy to the load balancing layer. The effect on capacity resembles a crash. (SRE Ch. 22,
p. 320.)

**A load balancing policy that avoids error-serving backends can start the snowball.** It
removes those backends from the available capacity, so the load on the rest rises.
(SRE Ch. 22, p. 320.)

**Health check eviction does not save a homogeneous layer.** In Nygard's search-engine case
the health checks worked and the layer still died. Every node held the same defect.
(Release It! Sec. 4.2, p. 63.)

**Failure in a remote system becomes your problem when your code is not defensive.**
(Release It! Sec. 4.1, p. 60.) Read `01-integration-points.md` and `03-stability-patterns.md`.

---

## 5. Slow startup and cold caching

A process answers slower right after it starts. Initialization must finish. A runtime may
still need to compile code and load classes. Caches are empty. (SRE Ch. 22, pp. 331–332.)

**With a warm cache only a few requests miss. With an empty cache every request is
expensive.** (SRE Ch. 22, p. 332.) Three events produce a cold cache. The team starts a new
cluster. The team returns a cluster to service after maintenance, with a stale cache. A
restart clears the cache. (SRE Ch. 22, p. 332.)

**Classify every cache before you depend on it.**

| Cache class | Test that classifies it | Consequence when it is empty |
|---|---|---|
| Latency cache | The service serves the expected load with an empty cache | Higher latency only |
| Capacity cache | The service cannot serve the expected load with an empty cache | A restart becomes an outage |

**Make every new cache a latency cache.** Otherwise engineer it well enough to work as a safe
capacity cache. (SRE Ch. 22, p. 333.) See entry `I-22` in `integration-fault-catalog.md`.

**Increase load slowly when you add load to a cluster.** Warm the cache at a small rate, then
add traffic. (SRE Ch. 22, p. 333.) Keep every cluster under nominal load so caches stay warm.

---

## 6. Triggering conditions

A cascade needs a cause and a trigger. This table lists the triggers the book names. Each row
gives the signal that identifies it during the incident.

| Trigger | Signal that identifies it | First action | Source |
|---|---|---|---|
| Process death | Tasks die together. One request class precedes each death. | Block the query of death. Read `08-fault-injection-and-test-harness.md`. | p. 334 |
| Process update | A binary or config push touched many tasks at once | Stop the push. Check the change log. | pp. 334–335 |
| New rollout | A new binary, a config change, or an infrastructure change inside the fault window | Consider a revert, above all when it changed capacity or the request profile | p. 335 |
| Organic growth | No change to blame, and use grew without a capacity change | Add capacity. Then plan capacity again. | p. 335 |
| Planned change, drain, or turndown | A dependency or a location left service on a schedule | Restore the drained capacity, or shed load | p. 335 |
| Request profile change | The traffic mix, the payload cost, or the data size per user changed | Measure the cost per request against last week | pp. 335–336 |
| Resource limit | The job depended on slack CPU that another job consumed | Stay inside committed limits. Do not treat slack CPU as a reserve. | p. 336 |

**Check for recent changes during a cascading failure, and consider a revert.** This applies
above all when the change touched capacity or the request profile. (SRE Ch. 22, p. 335.)

**A change log is the prerequisite.** The service must log its own changes, so that a
responder can identify recent ones fast. (SRE Ch. 22, p. 335.) See entry `I-37` in
`integration-fault-catalog.md`.

**Update infrastructure needs its own capacity budget.** Push off-peak. Adjust the number of
in-flight task updates against the request volume. (SRE Ch. 22, pp. 334–335.)

**Modern.** A rolling deploy, a cluster autoscaler, and a service mesh retry policy each
belong to the "new rollout" row. Neither book covers them by name.

---

## 7. Testing for cascading failure

**The way a service fails is hard to predict from first principles.** (SRE Ch. 22, p. 336.)
You must measure it.

1. Load test each component separately, until it breaks. Components have different breaking
   points. You cannot know in advance which one breaks first. (p. 337.)
2. Record the breaking point. It feeds capacity planning and the next regression test.
   (p. 337.)
3. Test a gradual increase and an impulse. Caching makes the two patterns differ. (p. 337.)
4. Test the return to nominal load. Ask whether the degraded mode exits without a human.
   (p. 337.)
5. For a stateful or a caching service, track state across interactions. Check correctness at
   high load. (p. 337.)
6. Test the large clients. Learn whether they queue work, whether they use randomized
   exponential backoff, and whether an external event makes them all act together. (p. 338.)
7. Test the non-critical backends. Test them unavailable, and test them blackholed. (pp.
   338–339.)

**A well-designed component sheds or degrades under overload.** It should not reduce the rate
at which it serves requests successfully. (SRE Ch. 22, p. 337.) A susceptible component
crashes or serves a very high error rate.

**Blackholing is the vendor-outage test.** Drop the requests to a backend so that it never
answers. A vendor that hangs is more dangerous than a vendor that refuses. The frontend must
survive it. The frontend must not reject many requests, exhaust resources, or serve very high
latency. (SRE Ch. 22, pp. 338–339.) Read `01-integration-points.md`.

**Nygard's equivalent is the test harness.** Build one simulator for each integration point.
Give it switches for system and network faults. Then use every switch while the system carries
heavy load. (Release It! Sec. 4.1, p. 59.) Read `08-fault-injection-and-test-harness.md`.

**A code path that you never use is a code path that often does not work.** (SRE Ch. 22,
p. 324.) Run a small set of servers near overload to exercise the degradation path. See entry
`I-40` in `integration-fault-catalog.md`.

---

## 8. The immediate steps

Use the incident management protocol when a cascade starts. (SRE Ch. 22, p. 339.) Read the
table downward. Take the first row whose signal you can confirm.

| Step | Signal that selects it | Precondition | Hazard |
|---|---|---|---|
| 1. Increase resources | Idle capacity exists, and the service is not yet in a death spiral | Free resources are available now | It does nothing once the service is in a death spiral |
| 2. Stop health check failures and deaths | The scheduler kills tasks while they start | You can disable the check that kills | The service now hides a real death from you |
| 3. Restart servers | A garbage-collection spiral, deadlock, or in-flight requests with no deadline | You localized the fault | A restart can amplify a cold-cache outage |
| 4. Drop traffic | The cascade continues and no other step works | You can address the trigger too | It is the big hammer. Users lose service. |
| 5. Enter degraded mode | Utilization passed the degradation threshold | The mode exists, and payloads carry a class | An unexercised path can fail when you need it |
| 6. Eliminate batch load | Index updates, copies, or statistics jobs share the serving path | You can stop those jobs | Delayed batch work becomes a later backlog |
| 7. Eliminate bad traffic | A request class precedes each crash | You can identify the request class | A wrong filter drops good traffic |

Sources for the rows: SRE Ch. 22, pp. 339–341.

### Step 2 in detail — health checks

**Separate the process health check from the service health check.** The process check asks
whether the binary answers at all. The load balancer's service check asks whether the binary
can serve this request class now. Conflated checks let health checking keep the service
unhealthy. Half the tasks start while the scheduler kills the other half. No task ever serves.
(SRE Ch. 22, p. 339.)

**Modern.** The liveness probe and readiness probe of Kubernetes are this same separation.

### Step 3 in detail — restart

**Identify the source of the cascading failure before you restart servers.** (SRE Ch. 22,
p. 340.) Confirm that the restart will not only move the load elsewhere. Canary the restart
and perform it slowly. A restart during a cold-cache outage amplifies the outage. See entry
`I-46` in `integration-fault-catalog.md` and read `07-automated-remediation-safety.md`.

### Step 4 in detail — drop traffic

1. Address the condition that triggered the cascade. Add capacity if that is the cause.
2. Reduce the load until the crashes stop. Be aggressive. Admit about 1% of traffic.
   (SRE Ch. 22, p. 340.)
3. Let most servers become healthy again.
4. Increase the load gradually, so caches warm and connections establish.

**Prefer a mechanism that drops the less important traffic first.** Prefetch traffic is the
book's example. Repair or mask the root cause before you restore traffic. Otherwise the
cascade starts again when the traffic returns. (SRE Ch. 22, p. 340.)

### Step 5 in detail — degraded mode

**Engineer the degraded mode ahead of the outage.** It needs a way to identify the payload
class. (SRE Ch. 22, p. 341.) Choose the mode per integration point before the incident, and
record it. (Release It! Sec. 5.2, p. 117.) Read `09-vendor-slas-and-degradation.md`.

---

## 9. Chain reaction, cascading failure, and the bulkhead

A bulkhead partitions a resource so that one penetration does not sink the ship. It enforces
damage containment. (Release It! Sec. 5.3, p. 119.)

| Failure | What the bulkhead partitions | What it stops | What it does not do |
|---|---|---|---|
| Chain reaction | The homogeneous layer, into separate pools | It splits one reaction into two that run at different rates (Sec. 4.2, p. 62) | It does not repair the defect. Only the defect repair ends the reaction. (Sec. 4.2, p. 61) |
| Chain reaction, for the caller | Nothing on the caller's side | Nothing | It does not help the callers of the partition that dies. Use Circuit Breaker there. (Sec. 4.2, p. 64) |
| Cascading failure | The caller's pools, one pool for each integration point | One sick dependency drains only its own pool (Sec. 5.3, p. 119) | It does not shorten a call that has no timeout |
| Cascading failure, inside one process | The thread groups, including a reserved admin pool | It keeps a diagnostic path open when worker threads hang (Sec. 5.3, p. 122) | It does not reduce the load |
| A shared vendor for two consumers | The vendor service, into one pool for each consumer | One consumer's spike or defect no longer harms the other (Sec. 5.3, pp. 119–121) | It needs reserve capacity in each partition |

**The partitioning procedure has four steps.** Examine the business impact of each lost
capability. Cross-reference those impacts against the architecture. Identify boundaries that
are technically feasible and financially beneficial. Choose the granularity. (Release It!
Sec. 5.3, pp. 121–122.) Each partition then needs its own reserve capacity and its own demand
forecast. (Release It! Sec. 5.3, p. 121.)

**SRE states the same isolation as an in-flight limit.** Allow any single client at most 25%
of your threads. That gives fairness when one client behaves badly. (SRE Ch. 22, p. 331.)

**Timeouts and Circuit Breaker are the caller-side pair.** Timeouts let you return from a call
to a troubled integration point. The breaker stops the call while the dependency is sick.
(Release It! Sec. 4.3, p. 67.) See entries `I-19`, `I-20`, and `I-21` in
`integration-fault-catalog.md`.

**Bulkheads work downward in the stack only.** Avoid communication inside a layer on the user
request path. Peer backends that wait on each other through one thread pool can deadlock.
(SRE Ch. 22, pp. 333–334.)

---

## 10. War stories

**Cascading Failure and Shakespeare (SRE Ch. 22, pp. 341–342).** A documentary about
Shakespeare aired in Japan and named the Shakespeare service. Traffic to the Asian datacenter
passed capacity, and a major service update ran there at the same time. The safeguards from
the production readiness review helped. Graceful degradation dropped pictures and small maps
as capacity became scarce, and timed-out remote calls were either not retried or retried with
randomized exponential backoff. Tasks still failed one by one, and the scheduler restarted
them, which lowered the number of working tasks further. The team was paged and restored
service by adding tasks in the Asian cluster. The postmortem produced two automated remedies.
The first redirects traffic to neighboring datacenters on overload. The second grows tasks
with traffic. The lesson is the combination trigger. Organic growth plus a concurrent rollout
breaks a service that survives either one alone.

**The nine-step chain (SRE Ch. 22, p. 319).** A Java frontend carried poorly tuned
garbage-collection parameters. Under expected load it exhausted CPU. Requests slowed. More
requests stayed in progress, and they used more memory. Less memory remained for the cache,
so the hit rate fell. More requests then reached the backend, which exhausted CPU and threads.
Basic health checks failed, and the cascade began. The book states that a responder is
unlikely to diagnose this full chain during the outage. Separate owners for the frontend and
the backend make it harder still. The lesson is that a resource dependency chain, not a single
resource, produces the outage.

**Searching... (Release It! Sec. 4.2, p. 63).** A retailer ran a dozen search engines behind a
hardware load balancer, with health checks that removed dead engines. A vendor memory leak
made the engines start to die around noon. Because each engine had carried the same share all
morning, they died in an accelerating pattern. Five or six minutes passed between the first
two crashes, three or four minutes between the next pair, and seconds between the last two.
The loss of the last search server locked the whole front end. The vendor patch took months,
so the team ran restarts at 11 a.m., 4 p.m., and 9 p.m. The lesson is that health checks alone
do not save a layer whose nodes share one defect.

**Hammer Time (Release It! Sec. 4.3, p. 67).** A calling layer retried the lower layer's quick
errors, because history said those errors were spurious. The lower layer's errors did not
carry enough detail to separate a transient error from a serious one. A failed switch then
started to drop database packets. The retry loop escalated. The calling layer then used 100%
of its CPU on those calls and on the log entries for the failures. The lesson has two halves.
A change in the caller can be the mechanism that jumps the gap. An error taxonomy that cannot
separate transient from permanent makes every retry policy unsafe. Nygard's own conclusion is
that a circuit breaker would have helped.

**The cache that killed the site (Release It! Sec. 4.5, pp. 85–86).** A retail site read
in-store availability from a remote inventory system through a correct read-through cache. The
inherited accessor was synchronized, and the override made an untimed remote call on a miss.
The inventory backend then failed under front-end load. One thread inside that call blocked
every other caller, and the whole site stopped. Blocked threads, unbalanced capacity, and a
missing timeout combined. Nygard's sentence is the point. Nobody designed this failure mode
in, and nobody designed it out.

---

## 11. The closing caution

**Some changes that improve the normal case raise the risk of a full outage.** (SRE Ch. 22,
p. 342.) The book names four such changes. Each one belongs in a self-healing design.

| Change | The improvement it gives | The cascade risk it adds |
|---|---|---|
| Retry on failure | It hides a transient error from the user | It amplifies load exactly when load is the problem |
| Move load away from unhealthy servers | It avoids one sick server | It raises the load on the servers that remain |
| Kill unhealthy servers | It removes a bad instance | It removes capacity that was merely busy |
| Add a cache | It lowers latency and backend load | It can become a hard dependency with a cold-start outage |

**Evaluate each change for positive feedback before you ship it.** Ask one question. Does this
mechanism add load, or remove capacity, when the system is already failing? If the answer is
yes, it needs a limit, a budget, and a kill switch. Take care not to trade one outage for
another. (SRE Ch. 22, p. 342.) Read `07-automated-remediation-safety.md`.

---

## 12. Read next

| Question | File |
|---|---|
| Where is the fault, in us or in the vendor? | `04-fault-localization.md` |
| Which traffic do I shed, and when? | `05-overload-and-load-shedding.md` |
| Can the system act without a human? | `07-automated-remediation-safety.md` |
| Which coded defects apply to this change? | `integration-fault-catalog.md` |

**A note on scope.** This file explains how a failure grows and how you stop the growth. It
does not choose an architecture. Send capacity estimation and component selection to the
`system-design` skill. Send the safety of a retried write to the `data-systems-design` skill.
