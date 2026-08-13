# Reliability, Scalability, Maintainability

**Source:** DDIA Ch. 1, p. 3.

The non-functional requirements that most software projects state vaguely and then
discover expensively. Everything here is meant to be turned into a number, a mechanism,
or a written acceptance of risk.

---

## 1. Thinking about data systems

Modern applications are not "a database" — they are a composition of datastores, caches,
search indexes, stream processors, and batch pipelines, stitched together by application
code. The moment you compose them, **you** own the guarantees of the composite, not the
vendor. Questions that become yours the instant you add a second store:

- What happens when one store accepts a write and another rejects it?
- What does a client see while the two disagree?
- How does a new node/consumer catch up?
- How do you detect that they have diverged, without a user reporting it?

If a design cannot answer these, it is not finished.

---

## 2. Reliability

**Definition:** continuing to work *correctly* — correct function, at the desired
performance, under the expected load — even when things go wrong.

**Fault ≠ failure.** A fault is one component deviating from its spec. A failure is the
system as a whole stopping service. Fault-tolerance means preventing faults from becoming
failures. It is impossible to reduce faults to zero, so design for tolerance.

Deliberately triggering faults (chaos testing, killing processes in production) is how you
learn whether your tolerance is real. Untested fault-handling code is broken
fault-handling code — it is the least-exercised path in the system.

### 2.1 Hardware faults
Disks fail, RAM goes bad, power drops, someone unplugs the wrong cable. Historically
handled by redundancy (RAID, dual PSUs, hot-swap). As deployments grow, single-machine
redundancy is not enough; you also need **software fault-tolerance across machines**,
because it additionally enables rolling upgrades and zero-downtime patching.

### 2.2 Software errors
The dangerous class, because they are **correlated**: a bad input crashes *every* node
that receives it; a runaway process exhausts a shared resource; a dependency slows down
and cascades. Redundancy does not help, because all replicas fail the same way.

Mitigations: careful assumption checking at interfaces, thorough testing, process
isolation, allowing crash-and-restart, and — crucially — **measuring and alerting on
invariants in production** so you find out before your users do.

### 2.3 Human errors
Configuration errors by operators are the leading cause of outages. Design for it:

- Minimize opportunity for error: good abstractions, APIs and admin interfaces that make
  the right thing easy and the wrong thing hard.
- Provide realistic sandboxes for experimentation with real data and no real consequences.
- Test thoroughly at all levels, especially the corner cases that rarely occur in practice.
- **Make recovery fast:** fast rollback, gradual rollout, tools to recompute derived data.
- Detailed, clear monitoring: performance metrics and error rates (telemetry).
- Good management practices and training.

### 2.4 Stating a reliability requirement
Reliability requirements must be written as **RPO/RTO plus a fault model**:

- **RPO (Recovery Point Objective):** how much data may be lost, in time.
- **RTO (Recovery Time Objective):** how long recovery may take.
- **Fault model:** for each fault class — tolerated / degraded / not tolerated — with the
  resulting behavior and blast radius.

"Highly available" is not a requirement. "Survives loss of one AZ with ≤30s of write
unavailability and zero data loss" is.

---

## 3. Scalability

**Definition:** having reasonable ways to cope with growth in load. Scalability is not a
one-dimensional label; a system is not "scalable" or "unscalable" — you ask
"if the system grows in *this* particular way, what are our options for coping?"

### 3.1 Describing load
Load is described by **load parameters** — the numbers that actually stress *this*
system. Choose them deliberately:

- requests per second to a service
- ratio of reads to writes
- number of simultaneously active users
- hit rate on a cache
- **fan-out**: how many downstream operations one input operation causes

The canonical example: posting a tweet is cheap, but delivering it to followers' timelines
is expensive and its cost is determined by the *distribution* of follower counts — the
average is useless; the tail (celebrity accounts) dominates. Look for the equivalent
skewed distribution in your own system, and design for the tail, not the mean.

### 3.2 Describing performance
Two ways to ask the question:

1. Increase a load parameter, keep resources fixed → how is performance affected?
2. Increase a load parameter → how much must resources grow to keep performance unchanged?

Batch systems care about **throughput**; online systems care about **response time**.

**Response time is a distribution.** Report percentiles, not averages:

- **p50 / median:** half of users are faster, half slower — the "typical" experience.
- **p95, p99, p999:** the tail. These are your service level objectives. The p999 user is
  often your most valuable one, because they have the most data and the longest history.
- Distinguish **response time** (what the client observes: service time + network + queueing)
  from **latency** (waiting to be handled). Users experience the former.

**Queueing delay** frequently accounts for most of the tail. A server can process only a
few things in parallel, so a small number of slow requests blocks everything behind them
— **head-of-line blocking**. This means you must **measure client-side**: server-side
timings cannot see the queue.

**Tail latency amplification:** when a single end-user request fans out to many backend
calls, the slowest call determines the total. The more backend calls, the higher the
chance that *at least one* is slow, so a small fraction of slow backends dominates the
end-user experience.

**Two rules people break constantly:**
- **Never average percentiles.** Averaging p95 across machines or over time is
  mathematically meaningless. Aggregate the underlying histograms (t-digest,
  HdrHistogram, forward decay) and compute the percentile from the combined data.
- **Never let the load generator wait for responses.** If the client sends the next
  request only after the previous one returns, it artificially keeps the queue short and
  reports optimistic, invalid numbers (coordinated omission).

### 3.3 Coping with load
- **Scale up (vertical):** simpler; one machine; expensive at the high end.
- **Scale out (horizontal, shared-nothing):** distributes load; introduces distributed
  systems problems (Ch. 5–9).
- Good architectures are usually **a pragmatic mixture**. A few powerful machines is often
  simpler and cheaper than a large fleet of small ones.
- **Elastic** (auto-scaling) helps with unpredictable load, but manually scaled systems
  have fewer operational surprises. Choose elasticity because load is unpredictable, not
  because it sounds modern.
- **Stateless services scale out easily; stateful systems do not.** Keep the database on a
  single node until cost or availability requirements force otherwise — distributing state
  is where the complexity lives.
- Architectures are **workload-specific**. An architecture that works at 1x is unlikely to
  work at 10x. Expect to rethink the architecture on every order-of-magnitude increase.

### 3.4 Required planning artifact
For any change with growth implications, state:

```
Load parameters today:      [numbers]
Expected in 12 months:      [numbers]
Saturates first at ~Nx:     [which resource: CPU / IOPS / connections / single partition / lock contention]
Symptom when it saturates:  [what users see]
Next architecture:          [what you'd do — and roughly when to start]
```

---

## 4. Maintainability

The majority of software cost is not initial development but ongoing maintenance. Three
design principles:

### 4.1 Operability — make life easy for operations
Good operations can work around bad software; good software cannot survive bad operations.
Provide:

- Visibility into runtime behavior and internals; good monitoring.
- Support for automation and integration with standard tools.
- Avoidance of dependency on individual machines (machines can be taken down for
  maintenance without downtime).
- Good documentation and an easy-to-understand operational model ("if I do X, Y happens").
- Good default behavior, with the freedom to override defaults.
- Self-healing where appropriate, plus manual control for when it isn't.
- Predictable behavior; minimal surprises.

### 4.2 Simplicity — manage complexity
Symptoms of accidental complexity: explosion of state space, tight coupling of modules,
tangled dependencies, inconsistent naming and terminology, hacks aimed at solving
performance problems, special-casing to work around issues elsewhere.

**Accidental complexity** is complexity that is not inherent in the problem the software
solves — it arises only from the implementation. The primary tool for removing it is
**abstraction**: hide implementation detail behind a clean, understandable façade.

Complexity makes maintenance hard, which makes bugs more likely. Simplicity is therefore
a *reliability* concern, not an aesthetic one.

### 4.3 Evolvability — make change easy
Requirements will change. Agility at the level of a large system is a function of how
easily it can be modified — which comes back to simplicity and good abstractions. The
concrete test: **can you make a schema or interface change and deploy it without
coordinated downtime?** (See `04-encoding-and-evolution.md`.)

---

## 5. Checklist for the planner and reviewer

- [ ] Load parameters named with numbers, including the skewed/fan-out dimension
- [ ] Latency objective stated as percentiles, measured client-side
- [ ] Throughput objective stated for batch/async paths
- [ ] Availability objective stated with a definition of "down"
- [ ] RPO and RTO stated
- [ ] Fault model table completed (tolerated / degraded / not tolerated + blast radius)
- [ ] Correlated-failure analysis: what single bad input or dependency takes down all replicas?
- [ ] Human-error mitigations: rollback path, gradual rollout, sandbox, recompute path
- [ ] "What breaks at 10x" answered with a specific resource and symptom
- [ ] Monitoring covers invariants, not just CPU/memory
- [ ] Simplicity justified: is a single-node solution sufficient? If not, why not?
