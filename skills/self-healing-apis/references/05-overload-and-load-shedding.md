# Overload, Load Shedding and Graceful Degradation

**Sources:** *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy), Ch. 21 "Handling
Overload", pp. 297–312. Deadlines, cancellation and degradation mechanics from SRE Ch. 22
"Addressing Cascading Failures", pp. 323–331. The pool throttle case from *Release It!*
(Nygard), Ch. 16, pp. 262–263.

This file answers one question. What does a caller do when a vendor API, or our own service, has
less capacity than the traffic demands? Overload is not an exception. Avoiding overload is a
goal of load balancing, and some part of the system still becomes overloaded in the end
(SRE Ch. 21, p. 297). Design the overload path before the incident. Read
`03-stability-patterns.md` for the defenses on one call. Read `06-cascading-failure.md` for the
failure that crosses a layer boundary.

**`capacity-engineering` owns the measurement.** It finds the resource that reaches its limit
first and proves the limit with a load test. Read
`capacity-engineering/references/06-load-balancing-and-utilization.md` for utilization signals
and balancing policy. This file states what a caller does once the limit is reached.

---

## 1. Model capacity in the constrained resource

**Do not model capacity in requests per second.** Do not model it in a static feature of a
request either, such as the number of keys the request reads. The ratio between the proxy
metric and the real cost changes over time. Google states the reason directly. "A moving
target makes a poor metric" (SRE Ch. 21, p. 298).

**Model capacity in the resource that becomes exhausted first.** Measure the service in
available resources, for example 500 CPU cores and 1 TB of memory (SRE Ch. 21, p. 298). Define
the cost of a request as a normalized measure of the CPU time that the request consumed
(SRE Ch. 21, p. 298).

| Capacity model | When it is correct | Failure mode |
|---|---|---|
| Requests per second | Never, as the provisioning model | The cost per request drifts. The model reports headroom that does not exist. |
| A static request feature | Never | The same drift. A cheap key and an expensive key count the same. |
| CPU seconds per second | The default at Google | Needs a normalized cost per request across machine types. |
| CPU plus a second resource | Another resource becomes exhausted before CPU | More signals to maintain and to threshold. |

**Why CPU works as the provisioning signal.** With garbage collection, memory pressure appears
as more CPU. Without it, provision the other resources so they last longer than CPU. Where
over-provisioning a non-CPU resource costs too much, account for each resource separately
(SRE Ch. 21, p. 298).

**Signal that your model is wrong:** the observed cost per request changed across releases,
and the request count did not change.

**War story — the release that invalidated the capacity model (SRE Ch. 21, p. 298).** Google
records that the ratio between the proxy metric and the real resource cost sometimes moves
gradually and sometimes drastically. The book gives a new software version that made some
request features need far fewer resources. The request count stayed the same, so a
queries-per-second model reported the same load, and the machines told a different story. When
you localize a fault to a change, compare the resource cost per request across versions, not
the request counts. `04-fault-localization.md` holds the procedure.

---

## 2. The order of responses to a resource restriction

Google states the order in one line. "Redirect when possible, serve degraded results when
necessary, and handle resource errors transparently when all else fails" (SRE Ch. 21, p. 297).

| Step | Action | Precondition | Cost to the user |
|---|---|---|---|
| 1 | Redirect the request to another location | Another location has spare capacity and the data | Extra latency only |
| 2 | Serve a degraded response | The degraded mode exists and is tested | Less accurate or less complete data |
| 3 | Return a resource error | Neither step above applies | The feature fails |

**A degraded response is a real answer that costs less to compute.** It is less accurate than
the normal answer, or it holds less data (SRE Ch. 21, p. 297). The book gives a search over
part of the candidate set, and an answer from a local copy that can be stale.

**Write the degraded mode down before the outage.** Read `09-vendor-slas-and-degradation.md`.
Catalog codes I-39 and I-40 apply.

---

## 3. Per-customer limits

**Global overload happens often.** It is frequent for an internal service with many client
teams (SRE Ch. 21, pp. 298–299). Assume it, and plan the split.

**Under global overload, only the misbehaving customer receives errors.** Google calls this
vital. Every other customer must stay unaffected (SRE Ch. 21, p. 299).

1. Negotiate the expected usage with each customer.
2. Provision capacity against the negotiated total.
3. Define a per-customer quota in CPU seconds per second.
4. Aggregate global usage in real time from all backend tasks.
5. Push the effective limit to each individual backend task.

**Signal to add per-customer limits:** one client can consume enough of the service to harm
another client. A second signal is that you cannot name the client that caused an overload.

**War story — 10,000 CPUs, and quotas that total more (SRE Ch. 21, p. 299).** A backend service
holds 10,000 CPUs allocated worldwide. It grants Gmail 4,000 CPU seconds per second, Calendar
4,000, Android 3,000, Google+ 2,000, and every other user 500. The quotas total 13,500 against
an allocation of 10,000. The owner accepts that oversubscription and relies on the customers not
reaching their limits at the same time. A quota is a blast-radius control first. It names the
customer that must absorb the error.

**Quotas can be set per criticality.** When a customer exhausts its global quota, a backend
rejects a given criticality only when it already rejects every lower criticality (SRE Ch. 21,
p. 303).

---

## 4. Client-side throttling and the adaptive throttle

**A rejection is not free.** For a backend whose reject path costs almost as much as its serve
path, a high rejection rate is itself an overload. The backend can become overloaded while it
spends most of its CPU on rejecting requests (SRE Ch. 21, p. 300).

**The client must reject some requests locally, before the network.** Google calls this
client-side throttling. The implementation is adaptive throttling (SRE Ch. 21, pp. 299–301).

1. Each client task keeps two counters over the last two minutes of its history.
2. `requests` counts the attempts that the application layer made, above the throttling system.
3. `accepts` counts the requests that the backend accepted.
4. The client sends every request while `requests` stays below `K × accepts`.
5. Past that cutoff, the client rejects new requests locally, with a probability.

**The book names the formula Client request rejection probability** (SRE Ch. 21, pp. 300–301).
It takes three inputs. They are `requests`, `accepts`, and the multiplier `K`. The book states
the behavior rather than a fixed rate. As the application attempt rate grows against the
backend accept rate, the probability of a local rejection increases.

**Locally rejected requests still count in `requests`.** So `requests` keeps exceeding
`accepts` after the client starts to throttle. Google states that this is the preferred
behavior (SRE Ch. 21, p. 301).

| Multiplier `K` | Behavior | Use it when |
|---|---|---|
| 2 | The default. Google prefers the 2x multiplier. | The normal case. It wastes some backend work and speeds the propagation of backend state to the clients. |
| Below 2, for example 1.1 | More aggressive. The backend rejects about one request for every ten it accepts. | The cost of rejecting a request is close to the cost of processing it. |
| Above 2 | Less aggressive. | The book gives no case for this. Treat the number as a local decision. |

**The design target.** Under sustained overload, Google reports that backends reject about one
request for each request they actually process (SRE Ch. 21, p. 301).

**Limit that the book states.** Client-side throttling "may not work well with clients that
only very sporadically send requests to their backends" (SRE Ch. 21, p. 302). A sporadic
client holds a very small view of backend state, and the book offers no cheap remedy. For a
vendor API called a few times an hour, use the circuit breaker in `03-stability-patterns.md`.

**Modern.** A vendor-facing client cannot see the vendor's accept decision unless the vendor
returns a distinguishable overload status. Ask the vendor for one.

---

## 5. Criticality

**Criticality is a property of the request, not of the service.** Google makes it a
first-class notion of the RPC system. Every request carries one of four values. Quota,
throttling and overload mechanisms all read it (SRE Ch. 21, p. 302).

| Value | Meaning | Default for |
|---|---|---|
| `CRITICAL_PLUS` | The most critical requests. Failure causes serious user-visible impact. | Nothing. A caller assigns it. |
| `CRITICAL` | User-visible impact, less severe than `CRITICAL_PLUS`. | Requests from production jobs |
| `SHEDDABLE_PLUS` | Partial unavailability is expected. | Batch jobs that can retry minutes or hours later |
| `SHEDDABLE` | Frequent partial unavailability and occasional full unavailability are expected. | Nothing. A caller assigns it. |

**Provision for all expected `CRITICAL` and `CRITICAL_PLUS` traffic** (SRE Ch. 21, p. 302).
The two sheddable classes are the capacity that you plan to give away under stress.

**Four values were enough.** Google found four sufficient to model almost every service. More
values would need more resources to operate the criticality-aware systems (SRE Ch. 21,
p. 302). Do not invent a fifth level.

**Set criticality once, near the edge, and propagate it.** Google sets it as close as possible
to the browser or the mobile client, usually in the HTTP frontend. It overrides the value only
at specific points (SRE Ch. 21, p. 303). The RPC system then propagates it. If request A causes
outgoing requests B and C, then B and C use A's criticality by default.

**Criticality is orthogonal to latency.** It is also orthogonal to network quality of service.
The book's example is search-as-you-type, which is highly sheddable and latency-stringent at
the same time (SRE Ch. 21, p. 303). Do not derive one from the other.

---

## 6. Utilization signals and task-level protection

**Compute the shed decision from state local to the task.** Utilization is usually the current
CPU rate divided by the total CPUs reserved for the task. It can also include the reserved
memory in use (SRE Ch. 21, pp. 303-304).

| Signal | How to compute it | Where it is best |
|---|---|---|
| CPU utilization | Current CPU rate divided by reserved CPU | The general default |
| Executor load average | Count the active threads. Smooth with exponential decay. | The most generally useful signal (SRE Ch. 21, p. 304) |
| Memory pressure | Memory use above normal operating parameters | A backend that holds large state |
| Queue length | Depth of the request queue | Listed as a shed metric (SRE Ch. 22, p. 324) |
| Latency | Observed serving latency | Listed as a shed metric (SRE Ch. 22, p. 324) |

**Executor load average, in detail.** An active thread runs now, or it is ready to run and
waits for a free processor. Start to reject requests as the smoothed count grows beyond the
number of processors available to the task (SRE Ch. 21, pp. 303–304). The exponential decay
swallows a brief spike from a fan-out burst. A sustained high load still triggers rejection.

**Reject by criticality as utilization approaches the threshold. Use a higher threshold for a
higher criticality** (SRE Ch. 21, p. 304). A task that is itself overloaded rejects the lower
criticalities sooner. The adaptive throttling system keeps separate statistics for each
criticality (SRE Ch. 21, p. 303).

**The per-task invariant.** A backend task provisioned for a traffic rate should keep serving
at that rate, with no large effect on latency, whatever excess traffic arrives. The task must
not crash. Google expects this up to a rate somewhere above 2x, or even 10x, the provisioned
rate (SRE Ch. 21, p. 311). Load test for your own figure.

**An overloaded backend must not stop accepting all traffic.** A well-behaved backend accepts
only the requests it can process and rejects the rest gracefully (SRE Ch. 21, p. 312). A
backend that stops entirely defeats the load balancer above it.

---

## 7. Load shedding and graceful degradation

**The two are different actions.** Load shedding drops a proportion of the load as the server
approaches overload. The server then still does as much useful work as it can
(SRE Ch. 22, p. 323). Graceful degradation goes one step further and reduces the amount of work
that the server performs (SRE Ch. 22, p. 323).

| Action | What changes | Effect on one request |
|---|---|---|
| Load shedding | Fewer requests enter the server | The dropped request fails |
| Graceful degradation | Each request costs less | The request succeeds with a worse answer |
| Queue discipline change | The order of service changes | An old, likely worthless request is dropped |

**The queue discipline is a shed control.** A change from FIFO to LIFO, or the controlled
delay algorithm, drops requests that are unlikely to be worth processing. It works well
together with propagated RPC deadlines (SRE Ch. 22, p. 323). Section 10 covers the deadlines.

Answer three design questions before you write the code. Then follow four operating rules
(SRE Ch. 22, p. 324).

1. Which metric determines that shedding or degradation starts?
2. Does the system enter the degraded mode automatically, or does a human enter it?
3. What actions does the degraded mode take, and at which layer?
4. Expect a rare trigger, usually after a capacity planning failure or an unexpected load shift.
5. Exercise the code path on purpose. Run a small subset of servers near overload.
6. Monitor how many servers are in the degraded mode. Alert when too many are.
7. Build a fast way to disable a complex degradation path, or to tune its parameters.

**War story — the throttle at the connection pool (Release It! Ch. 16, pp. 262–263).** During a
Black Friday outage the only available throttle was a connection pool that served one
integration point. The team set the pool maximum and the checkout block time to zero on one
node. Nothing happened, because the maximum takes effect only when the pool starts. They then
called `stopService()` and `startService()` on that component, and the node recovered. They
repeated the change across the fleet, and the external monitor turned green about ninety seconds
later. Dynamic reconfiguration plus a component restart took under five minutes. A configuration
file change plus a full restart would have taken more than six hours. A runtime dial on a live
component is the fastest control you own.

**A throttle needs a floor above zero.** Through that weekend the team raised the maximum when
load was light and lowered it to one, not zero, when load was heavy. A pool with a zero
maximum is disabled, and that disabled the whole feature (Release It! Ch. 16, pp. 262–263).
Build a dial, not a switch.

---

## 8. Retry budgets

**A retry without a budget is a load amplifier.** A cap on attempts alone is not a budget.
With a three-attempt cap and nothing else, Google measured request volume growing to just
below 3X during an overload (SRE Ch. 21, p. 306).

| Budget | Rule | Source |
|---|---|---|
| Per request | At most three attempts. After the third failure the error travels to the caller. | SRE Ch. 21, p. 306 |
| Per client | Retry only while the client's ratio of retries to requests stays below 10%. | SRE Ch. 21, p. 306 |
| Per process | A server-wide budget, for example 60 retries per minute. Above it, fail the request. | SRE Ch. 22, p. 327 |

**The two budgets together.** The per-request budget alone permits growth to just below 3X.
The 10% per-client budget reduces the growth to about 1.1x in the general case (SRE Ch. 21,
p. 306). Implement both. Catalog code I-12 applies.

**Tell the backend how many attempts this request already had.** The client puts an attempt
counter in the request metadata. It is 0 on the first attempt. Each retry increments it, to a
maximum of 2 (SRE Ch. 21, p. 306).

**The backend uses those counters to suppress retries.** Backends keep histograms of the
attempt counters over recent history. When a backend must reject a request, it reads the
histograms to estimate whether other tasks are also overloaded. If they show many retries, the
backend returns "overloaded, do not retry" instead of the standard "task overloaded" error
(SRE Ch. 21, p. 306). The first error suppresses a retry. The second permits it.

**Whether to retry depends on how much of the fleet is overloaded** (SRE Ch. 21, pp. 304–305).

| Scope of the overload | Action | Reason |
|---|---|---|
| A large subset of tasks is overloaded | Do not retry. Let the error travel to the caller. | No task has spare capacity. A retry only adds load. |
| A small subset is overloaded, the typical case | Retry immediately | Other tasks hold capacity. The extra cost is a few network round trips. |

Google adds two caveats. With a perfect cross-datacenter load balancing system, the first row
would not occur. And no explicit logic sends a retry to a different task. The system relies on
probability, given the number of backends in the subset (SRE Ch. 21, p. 305).

---

## 9. Retry at exactly one layer

**Retry a rejected request only at the layer immediately above the layer that rejected it**
(SRE Ch. 21, p. 310). Every other layer returns "overloaded, do not retry" or a degraded
answer. Catalog code I-11 applies.

**The arithmetic of the violation.** Several layers retry the same request. The attempt count at
the bottom layer is then the product of the attempt counts at each layer (SRE Ch. 22, p. 327).
Three layers with three attempts each give twenty-seven calls to the bottom service.

**War story — the DB Frontend stack (SRE Ch. 21, pp. 307–310).** The chain is Frontend,
Backend A, Backend B, DB Frontend. The DB Frontend is overloaded and rejects. Backend B
retries within its budgets, because it is the layer immediately above. Once Backend B decides
that the request cannot be served, it returns exactly one of two things to Backend A. It
returns "overloaded, do not retry", or a degraded answer. Backend A then holds the same two
options toward the Frontend. Google states the alternative in one line. If multiple layers
retried, there would be a combinatorial explosion.

---

## 10. The request deadline

**Set a deadline on every outbound call.** No deadline, or an extremely high deadline, lets a
problem that started long ago keep consuming server resources until a restart (SRE Ch. 22,
p. 328). Catalog code I-05 applies.

**Propagate the deadline. Do not invent one at each hop.** Set one absolute deadline high in
the stack. The whole RPC tree from the initial request shares it. Each hop subtracts the time
it already spent, so 30 seconds becomes 23 seconds and then 19 seconds (SRE Ch. 22,
pp. 328–329). Catalog code I-06 applies.

1. Subtract a few hundred milliseconds for transit and for client post-processing (p. 329).
2. Check the remaining deadline before each stage of the work (p. 328).
3. With a checkpointed catch-up operation, check the deadline after the code writes the
   checkpoint, not after the expensive step (p. 329).
4. Consider an upper bound for a non-critical backend, or for one that normally answers quickly.
   Understand the traffic mix first, or a large payload fails every time (p. 329).

**A deadline several orders of magnitude above the mean request latency is usually bad**
(SRE Ch. 22, p. 331). Catalog code I-07 applies.

| Call class | Deadline source | Cap | On expiry |
|---|---|---|---|
| User-facing synchronous call | Propagated from the edge request | Below the caller's remaining budget | Return an answer now. Queue the work. |
| Non-critical enrichment call | An upper bound that we set | Small, well below the user deadline | Skip the enrichment. Serve the degraded answer. |
| Background or batch call | A job-level deadline | Generous, but finite | Fail the item. Do not retry inside the loop. |
| Health probe | Fixed, independent of the work path | Shorter than the probe interval | Mark the endpoint unhealthy. |

**War story — the hardcoded 20-second deadline (SRE Ch. 22, p. 329).** Server A sends an RPC
to Server B with a 10-second deadline. B spends 8 seconds, then calls Server C. B should send
the 2-second remainder. Instead it sends a hardcoded 20-second deadline. C removes the request
from its queue after 5 seconds and works on it, and it believes it has 15 seconds left. Server
A already abandoned the request. Every CPU second that C spends is waste. The book states the
rule in one line. "You don't get credit for late assignments with RPCs" (SRE Ch. 22, p. 328).

**War story — bimodal latency and the 80.4% error rate (SRE Ch. 22, pp. 330–331).** Ten
frontends run 100 threads each, so 1,000 threads exist. At 1,000 queries per second and 100
milliseconds each, the normal demand is 100 threads. Some Bigtable row ranges then become
unavailable. Five percent of requests never complete, and each one holds a thread for the full
100-second deadline. That demands 5,000 threads. The frontend serves 19.6% of its requests, so
a 5% fault produces an 80.4% error rate. The deadline was three orders of magnitude above the
mean latency, and the mean hides the failure. Examine the distribution, and use the RPC
fail-fast option.

---

## 11. Cancellation propagation

**Tell the servers below you that their work no longer matters.** Cancellation propagation
advises the servers in an RPC call stack that their effort is no longer necessary (SRE Ch. 22,
p. 330).

**Signal to add it:** an initial RPC carries a long deadline, and a deeper RPC already failed
without a possible retry. Without cancellation, the initial call holds its resources until its
own deadline expires, although the request is already doomed (SRE Ch. 22, p. 330).

**Hedged requests require it.** A hedged request goes to a primary instance, and the same
request goes later to other instances. On the first answer, the caller cancels the extra
calls. That needs cancellation propagation through the whole stack (SRE Ch. 22, p. 330).
Without it, a hedge doubles the load.

**Modern.** A vendor HTTP API usually gives you no cancellation channel. Closing the
connection is a hint, not a contract. Assume the vendor keeps working, and design the
operation to be idempotent. Catalog code I-14 applies.

---

## 12. When this reference does not apply

State "not applicable" and stop, in these cases.

- The call is to a local library, and it crosses no process boundary.
- The service serves one internal caller at a known, fixed rate.
- The change touches no outbound call, no deadline, no retry, and no queue.
- The traffic is a single scheduled job, and a failed run repeats on the next schedule.

**Do not add a quota system, a criticality scheme, and an adaptive throttle to a service with
one client.** Add a deadline and a retry budget. Those two apply to every outbound call. The
rest of this file starts to pay when a second class of caller exists.

**Do not report a threshold that this file does not give.** Google gives 2 for the multiplier
`K`, three attempts per request, a 10% per-client retry ratio, and 60 retries per minute per
process. It gives no universal utilization threshold and no universal deadline. Derive those
from your own load test and your own latency distribution.
