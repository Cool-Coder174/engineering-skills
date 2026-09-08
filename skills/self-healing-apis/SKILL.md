---
name: self-healing-apis
description: Fault-localization and self-healing engine for integrated vendor APIs, grounded in Release It! (Nygard) and Site Reliability Engineering (Google). Use it when a task touches a vendor API call, an SDK or client library, a webhook, a timeout, a retry, or a circuit breaker. Use it when a task touches a bulkhead, a health check, a deadline, load shedding, a degraded mode, a vendor SLA, or a rate limit. Use it when a vendor API is slow, hangs, returns garbage, or returns a well formed error. Use it to decide whether the fault is in us, in the network, or in the vendor. Provides a localization procedure, decision tables, a named integration-fault catalog, and deep-dive references.
---

# SELF HEALING APIS

**ROLE:** You are the principal engineer for vendor integrations. You own every call that
leaves our process.

**CORE FUNCTION:** You detect that an integrated API is faulty. You localize the fault to us,
to the network, or to the vendor. You heal the fault automatically where healing is safe.

This skill is a **knowledge and gate skill**. It does not run a workflow. `planner`,
`detail-planning`, `implement`, `verify`, and `code-review` consult it. You can also invoke
it directly for one integration question.

Nygard states the premise in one sentence. "Integration points are the number-one killer of
systems." (Release It! Sec. 4.1, p. 46.)

## How this skill relates to the other skills

| Skill | Question it answers |
|---|---|
| `system-design` | What must we build? |
| `data-systems-design` | Does the design stay correct across machines? |
| `self-healing-apis` (this skill) | Does a vendor fault become our outage? |
| `security-engineering` | Can an attacker break it? |
| `production-troubleshooting` | How do I find the cause of any production fault? |
| `observability` | What must the system expose, and what must page a person? |
| `slo-engineering` | What reliability target may we set, and what is the budget? |
| `capacity-engineering` | Which resource reaches its limit first? |
| `incident-response` | Who runs the incident, and what do we write down? |
| `reverse-branching` | How does a person undo this change? |

Section 9 gives the full split. Read it before you duplicate work that another skill owns.
**This skill owns one boundary only: the call that leaves our process.** Every general
method it uses belongs to one of the skills above.

---

# 1. ACTIVATION

Activate this skill when the work touches any row of this table.

| Trigger area | Examples |
|---|---|
| Outbound calls | a vendor API call, an SDK, a client library, a socket, a webhook we send |
| Waiting | a timeout, a deadline, a retry, a blocked thread, a pool checkout |
| Defenses | a circuit breaker, a bulkhead, a resource pool, a rate limit, a quota |
| Detection | a health check, a synthetic transaction, a probe, an alert on a dependency |
| Overload | load shedding, criticality, a degraded mode, back pressure |
| Symptoms | the vendor is slow, the vendor hangs, the vendor returns garbage |
| Contracts | a vendor SLA, support hours, a change calendar, an availability promise |
| Automation | an automatic restart, a traffic drain, a failover, a self-healing loop |
| Money | a payment, a transfer, or any call we cannot repeat safely |

**A firewall rule is an integration point.** Why would there be a hole in the firewall, if
not to call some other system? (Release It! Sec. 14.1, p. 243.)

If no row applies, this skill stays silent. Do not apply integration ceremony to a change
that makes no outbound call. Section 10 gives the proportionality rule.

---

# 2. THE SIX WAYS AN INTEGRATION POINT FAILS

Nygard names six failure classes in Sec. 4.1, pp. 46–59. Every vendor fault you meet is one
of these six, or a combination of them. Name the class before you name a remedy.

## Table 1 — The six failure classes

| Failure class | What the caller sees | Time to detect with no defense | Required defense |
|---|---|---|---|
| Connection refused | An immediate exception | Under 10 ms on the same switch (Sec. 4.1, p. 48) | Normal error handling. Count the failure in the breaker. |
| Connection waits in the listen queue | Nothing. No error and no answer. | Minutes. The operating system decides. (Sec. 4.1, p. 49) | Connect timeout. Circuit breaker. |
| Connection accepted, no answer | Nothing. The socket stays open. | Forever. Java `read()` blocks by default. (Sec. 4.1, p. 49) | Read timeout. Deadline. |
| Answer arrives slowly | A late answer that is valid | The caller's own patience | A deadline near the observed distribution. Fail Fast. |
| Answer is protocol garbage | A parse error, or a silent wrong value | Never, when the code does not check | Status check. Content-type check. Schema check. Size cap. |
| Answer is a well formed error | A defined error response | Immediately | Classify the error as retriable or permanent. Feed the breaker. |

**The ordering rule.** A slow response is much worse than no response. (Release It!
Sec. 4.1, p. 50.) A slow answer holds a thread in the caller and a thread in the callee. A
refusal holds neither.

**Why a hang, and not an error.** A firewall that drops an idle connection sends no reset
packet and no ICMP message. The TCP stack retransmits into a black hole. Linux 2.6 with
`tcp_retries2 = 15` gives a twenty-minute write timeout. HP-UX gave thirty minutes. A read
can block with no end. (Release It! Sec. 4.1, pp. 53–56.)

**The transition that hides the outage.** In the airline case the first twenty callers
received an exception. After that, the calls stopped returning errors and started to block.
(Release It! Sec. 3.3, p. 40.) An error-rate alert goes quiet exactly when the fault gets
worse.

---

# 3. THE LOCALIZATION PROCEDURE

This section answers the question the issue asks. Where is the fault? Run these six steps in
order. SRE Ch. 12, pp. 170-182, gives the step names.

**`production-troubleshooting` owns the general loop.** Read
`production-troubleshooting/references/01-the-troubleshooting-model.md` for the method itself.
This section states only what changes when the suspect party is a vendor we do not operate.

1. **Problem report.** Record the expected behavior, the actual behavior, and the way to
   repeat it. File one ticket. Route it to the engineer on duty. (SRE Ch. 12, p. 172.)
2. **Triage.** Restore service first. Divert traffic, shed traffic, or disable the feature.
   Preserve the logs as you do it. (SRE Ch. 12, p. 173. Release It! Ch. 2, p. 24.)
3. **Examine.** Read the per-operation error rate and the latency distribution for this one
   integration point. (SRE Ch. 12, p. 175.) Read a thread dump. (Release It! Ch. 2, p. 29.)
4. **Diagnose.** Read one row of Table 2. Run the probe in its last column. Then read
   Table 2 again with the new evidence.
5. **Test and treat.** Design a test with two exclusive answers. Run the most likely test
   first. Record every change you make. (SRE Ch. 12, pp. 179–180.)
6. **Cure.** Prove the cause as far as production allows. Write the postmortem. Accept
   several joint causes when one cause does not explain the symptoms. (SRE Ch. 12, p. 182.)

**Two cautions on Table 2.**

- Correlation is not causation. At scale, coincidental correlations are certain. (SRE Ch. 12, p. 171.)
  A remediation built on a temporal correlation becomes superstition. An operations group ran a
  weekly database failover for six months on one. (Release It! Ch. 17, pp. 281–283.)
- Prefer the common cause. When you hear hoofbeats, think of horses and not zebras. (SRE
  Ch. 12, p. 171.) Then accept that several small faults can be jointly responsible.

## Table 2 — Symptom to fault location

| Observed signature | The fault is probably in | This rules out | Next probe | Source |
|---|---|---|---|---|
| Connection refused in milliseconds, from every caller | The vendor process, or the vendor's load balancer | The network path. It carried the reset packet. | Call the vendor host from a second network location. | Release It! Sec. 4.1, p. 48 |
| The connect attempt hangs. No reset packet arrives. | The network path, or a middlebox that drops packets | The vendor application. It never saw the packet. | Capture packets on both sides of the firewall. | Release It! Sec. 4.1, pp. 51–53 |
| Our threads block in a socket read. The vendor answers our probe. | The vendor application, behind an accepted connection | The network path. The handshake finished. | Read a thread dump. Send a known-good probe from outside our network. | Release It! Ch. 2, p. 29 |
| Our threads block on pool checkout. Few threads sit in a socket read. | Our own pool size, or our own capacity | The vendor, at first order | Read the pool metrics. Compare the in-flight count against the pool maximum. | Release It! Ch. 2, p. 31 |
| CPU is low on our hosts and most threads are busy | A blocked integration point below us | Our own CPU capacity | Move one hop out. Repeat the same two measurements. | Release It! Ch. 16, pp. 258–259 |
| Latency rises for one vendor operation only | The vendor, for that operation | The network path, and our shared client code | Compare error rate and latency per operation. | SRE Ch. 12, p. 175 |
| A response header names the backend that answered | The named tier | Every tier above the named one | Read the header. Discount the tiers above it. | SRE Ch. 12, pp. 176, 178 |
| Errors start at the same time every day | An idle timeout in a middlebox or in a pool | The vendor application | Add a quiet period to the load profile and repeat. | Release It! Sec. 4.1, pp. 53–56 |
| The error rate fell to zero and no answers arrive | Resource exhaustion inside our own process | A vendor that returns errors | Read the thread dump. Errors stop when the threads stop. | Release It! Sec. 3.3, p. 40 |
| About half of the probes succeed, with no pattern | A partial fault. Some vendor instances are sick. | A clean up-or-down state | Probe repeatedly. Record every failing response. | SRE Ch. 12, p. 175 |
| Latency is bimodal. A small share of calls never finish. | A subset of the vendor's keyspace or shards | A uniform vendor slowdown | Read the latency distribution, not the mean. | SRE Ch. 22, pp. 330–331 |
| The vendor's status page is green and our probe fails | Unresolved. Both sides are still possible. | Nothing | Run your own synthetic transaction from two locations. | Status pages are **Modern**. The probe is Release It! Ch. 13, p. 231 |
| The fault started inside the window of our push | Our change | Nothing yet | Read the change log. Annotate the error graph with the push window. | SRE Ch. 12, pp. 177–178 |
| Two unrelated consumers failed at almost the same time | A shared third dependency | An independent fault in each consumer | List the dependencies both consumers hold. Test the common one. | Release It! Ch. 2, pp. 25–26 |

## Table 3 — Health signal strength

Read this table before you trust a green signal. `observability` owns signal design, alert
rules, and the four golden signals. This table answers one narrower question. What does a
green signal about a vendor prove?

| Signal | What it proves | What it does not prove |
|---|---|---|
| The process is running | The process exists | Nothing about transactions. Release It! Sec. 4.5, p. 82 |
| A TCP port accepts a connection | A listener exists | Nothing about the work path |
| A static status endpoint returns 200 | One thread pool answers | Nothing about the pool that serves work. Release It! Ch. 2, p. 31 |
| The vendor's own status page is green | The vendor believes it is healthy | Nothing about our traffic or our route. **Modern** |
| The vendor's alerts are quiet | Nothing, when their alarms are untuned | Nothing. Release It! Ch. 16, p. 261 |
| Our synthetic transaction succeeds from two locations | The whole path worked for that request | Nothing about the tail. Release It! Ch. 13, p. 231 |
| Per-operation error rate and latency distribution | The shape of the failure | Nothing about the cause. SRE Ch. 12, p. 175 |
| A thread dump | Which layer holds the threads | Nothing about why that layer is slow. Release It! Ch. 2, p. 29 |
| Packet capture on both sides of a middlebox | Whether the packets crossed | Nothing about application state. Release It! Sec. 4.1, p. 52 |

**Instrument both sides of the boundary.** You must know how fast the caller believes the
vendor to be. You must also know how fast the vendor believes itself to be. Without both
numbers you cannot separate a slow vendor from a slow network. (SRE Ch. 6, p. 87.)

**The absence of a signal is a signal.** In the "5 a.m. Problem" the near-total absence of
packets on the wire located the fault. (Release It! Sec. 4.1, p. 53.)

---

# 4. DECISION TABLES

Use Tables 4, 5 and 6 to decide quickly. Read the reference named in Section 9 before you
commit to an expensive choice. Section 5 holds Tables 7 and 8.

## Table 4 — Which pattern applies to which direction

Nygard states the split. Timeouts apply to outbound requests. Fail Fast applies to incoming
requests. (Release It! Sec. 5.1, p. 114.)

| Pattern | Direction | What it does | Source |
|---|---|---|---|
| Use Timeouts | Outbound | Stops the wait for an answer that may never arrive | Sec. 5.1, p. 111 |
| Circuit Breaker | Outbound | Prevents the call while the dependency is sick | Sec. 5.2, p. 115 |
| Bulkheads | Both | Contains the damage inside one partition | Sec. 5.3, p. 119 |
| Steady State | Internal | Recycles every resource the system accumulates | Sec. 5.4, p. 124 |
| Fail Fast | Inbound | Refuses work the system cannot finish | Sec. 5.5, p. 131 |
| Handshaking | Inbound | Tells the caller to send less | Sec. 5.6, p. 134 |
| Test Harness | Test | Produces the out-of-spec faults a mock cannot produce | Sec. 5.7, p. 136 |
| Decoupling Middleware | Architecture | Removes the shared clock between caller and callee | Sec. 5.8, p. 141 |

**The Circuit Breaker state machine.** Closed sends the call to the vendor and counts
failures. At the threshold the breaker trips to Open. Open fails every call at once, with no
attempt at the real operation, and throws a different exception. After a timeout the breaker
moves to Half-Open. Half-Open permits exactly **one** trial call. Success resets the breaker
to Closed. Failure returns it to Open. (Release It! Sec. 5.2, pp. 115–117.)

**A Circuit Breaker prevents operations. It does not repeat them.** (Release It! Sec. 5.2, p. 115.)

**Handshaking has a limit over HTTP.** Most clients recognize only "200 OK", "403", and
"302". They treat "503 Service Unavailable" as a fatal error. CORBA, DCOM, and Java RMI
signal readiness equally poorly. (Release It! Sec. 5.6, p. 134.) A Circuit Breaker is the
stopgap for a service that cannot handshake. (Release It! Sec. 5.6, p. 135.)

## Table 5 — Retry decision

| Error class | Retry? | Backoff | Budget | Note |
|---|---|---|---|---|
| Connection refused | Yes | Randomized exponential | 3 attempts per request. 10% retry ratio per client. | SRE Ch. 21, p. 306 |
| Connect timeout | Yes | Randomized exponential | Same | The request never reached the vendor |
| Read timeout, no answer | Only with an idempotency key | Randomized exponential | Same | The write may already have landed |
| Explicit overload status from the vendor | Yes, after the stated delay | Honor the stated delay. Otherwise use exponential backoff. | Same | The `Retry-After` header is **Modern**. HTTP 429 is **Modern**. |
| A 5xx status that is not an overload status | Yes | Randomized exponential | Same | SRE Ch. 22, p. 327 |
| A 4xx status other than the overload status | No | — | — | Never retry a permanent error or a malformed request. SRE Ch. 22, p. 327 |
| Protocol garbage in the body | No | — | — | A repeat returns the same garbage |
| "Overloaded, do not retry" | No | — | — | The callee already decided. SRE Ch. 21, p. 306 |
| The breaker for this dependency is open | No | — | — | The breaker already decided. Release It! Sec. 5.2, p. 116 |
| A transient error with no distinguishable class | No, until the vendor gives a class | — | — | The "Hammer Time" case. Release It! Sec. 4.3, p. 67 |

**The arithmetic that makes the budget mandatory.** Three layers each retry three times.
That gives 4³ = 64 attempts against the vendor for one user action. (SRE Ch. 22, p. 327.) The
three-attempt cap alone still permits request volume to grow to just under 3x. The 10%
per-client budget reduces the growth to about 1.1x. (SRE Ch. 21, p. 306.)

**On timeout, return an answer.** Return a failure, a success, or a note that the work is
queued. Then retry slowly from the queue. An immediate retry is very likely to fail again,
because problems inside a data center last for a while. (Release It! Sec. 5.1, pp. 113–114.)

## Table 6 — Deadline and timeout budget

| Call class | Where the deadline comes from | Cap | On expiry |
|---|---|---|---|
| User-facing synchronous call | Propagated from the edge request | Below the caller's remaining budget | Return an answer now. Queue the work. |
| Non-critical enrichment call | An upper bound we set | Small, well below the user deadline | Skip the enrichment. Serve the degraded answer. |
| Background or batch call | A job-level deadline | Generous, but finite | Fail the item. Do not retry inside the loop. |
| Health probe | Fixed, and independent of the work path | Shorter than the probe interval | Mark the endpoint unhealthy. |
| Webhook we receive | Our own processing budget | Short. The sender has its own timeout. | Acknowledge first. Process the work later. **Modern** |

Rules that apply to every row.

- Set one absolute deadline high in the stack. Subtract the elapsed time at each hop. 30 s
  becomes 23 s becomes 19 s. (SRE Ch. 22, pp. 328–329.)
- Subtract a few hundred milliseconds for network transit. (SRE Ch. 22, p. 329.)
- Check the remaining deadline before each stage of the work. (SRE Ch. 22, p. 328.)
- A deadline several orders of magnitude above the mean latency is usually wrong. (SRE Ch. 22, p. 331.)
- Cap the outgoing deadline for a non-critical backend. Understand the traffic mix first. (SRE Ch. 22, p. 329.)
- For a website that uses services, "fast enough" is probably under 250 ms. (Release It! Sec. 5.1, p. 113.)

---

# 5. THE HEALING LADDER

Read Table 7 downward. Take the highest row that solves the problem. The rows are ordered by
blast radius. The last three rows need a human. `reverse-branching` owns the undo path for
those rows. `incident-response` owns the roles and the record once a person is paged.

## Table 7 — The healing ladder

| Action | Precondition | Stop condition | Blast radius | Automatic? |
|---|---|---|---|---|
| Retry one request | The operation is idempotent or carries a key. Budget remains. The breaker is closed. | The budget is spent, or the error is permanent | One request | Yes |
| Fail one request fast | The breaker is open, or a required resource is missing | — | One request | Yes |
| Open the breaker | The failure count for this dependency passed its threshold | The half-open trial call succeeds | One dependency, one process | Yes |
| Serve the degraded answer | The degraded mode is recorded and tested | The dependency recovers | One feature | Yes |
| Throttle the pool for this dependency | A utilization signal passed its threshold | Utilization returns below the threshold | One caller class | Yes, with a floor above zero |
| Shed load by criticality | Criticality is set at the edge and propagated | Utilization returns below the threshold | The sheddable classes | Yes |
| Drain traffic from one location | An independent probe confirms the location cannot serve | The probe recovers | One location | Yes, rate limited |
| Restart one process | The fault is localized. The restart cannot amplify it. | N consecutive restarts fail | One process | Yes, canaried and rate limited |
| Move traffic to another region | Capacity is confirmed at the target. State loss is accepted. | — | One region | Human approval |
| Revert the release | The change log names a change inside the fault window | — | Global | Human approval |
| Page a human | Every automatic step is spent, or the action is not reversible | — | — | Always permitted |

**Sources for the rows.** Retry budgets, SRE Ch. 21, p. 306. Breaker states, Release It!
Sec. 5.2, p. 116. Degraded answers, SRE Ch. 21, p. 297. Pool throttle with a floor above zero,
Release It! Ch. 16, pp. 262–263. Criticality, SRE Ch. 21, pp. 302–303. Drain, SRE Ch. 13,
p. 195. Restart after localization, SRE Ch. 22, p. 340. Change log, SRE Ch. 22, p. 335.

**The four properties.** An automated remediation must be idempotent, rate limited,
observable, and reversible. Each property carries its own source. Idempotent and rate
limited, SRE Ch. 7, p. 118. Observable, SRE Ch. 7, p. 113. Reversible, SRE Ch. 13,
pp. 190-191. It must publish a "manual override applied" state. (Release It! Ch. 17, p. 271.)
It must stop and notify a human after repeated failure. (SRE Ch. 7, p. 110.)

**The self-denial rule.** An automatic retry storm is a denial of service that we perform on
ourselves. Nygard names this class "Attacks of Self-Denial". (Release It! Sec. 4.6, p. 88.)
A cache flush that fires too often belongs to the same class. (Release It! Sec. 10.2,
p. 209.)

**The trade rule.** Some changes that reduce background errors raise the risk of a full
outage. Retries, load shifts, killing unhealthy servers, and new caches are the four named
examples. (SRE Ch. 22, p. 342.) Check each new healing action for positive feedback.

## Table 8 — Degraded mode per integration point

Choose one row for each integration point. Record the choice before the outage. (Release It!
Sec. 5.2, p. 117.)

| Degraded mode | Use it when | What the user sees | Build this first |
|---|---|---|---|
| Cached answer | The answer stays valid for a known period | An answer with a stated age | A cache with a size bound and an invalidation rule |
| Queued answer | The work can finish later | A confirmation now, and a result later | A durable queue and a path for the late answer |
| Default answer | A safe constant exists | A conservative result | A recorded decision on the constant, approved by the sponsor |
| Partial answer | The feature is one part of a larger answer | The page or record without that part | A renderer that tolerates the missing part |
| Refusal | No safe answer exists | An immediate, clear refusal | A message and a documented error code |

**The cache row carries a trap.** A cache that the service cannot run without is a capacity
cache, not a latency cache. A cold capacity cache turns a restart into an outage. Make every
new cache a latency cache, or engineer it well enough to serve as a capacity cache. (SRE
Ch. 22, p. 333.)

**The refusal row is a legitimate answer.** The business sponsor decides it. Nygard's example
question asks whether the site should crash when it cannot check availability for in-store
pickup. (Release It! Sec. 4.5, p. 85.)

**Exercise the degraded path.** The code path you never use is the code path that often does
not work. (SRE Ch. 22, p. 324.) Run a small share of traffic against it on a schedule. Alert
when too many callers enter the degraded mode. (SRE Ch. 22, p. 324.)

---

# 6. INTEGRATION FAULT GATE (MANDATORY)

## 6.1 The twenty-one rules

Check every rule that applies to the change. Each rule carries its source.

1. Every outbound call has an explicit connect timeout and read timeout. Release It! Sec. 4.1, p. 49.
2. Every blocking wait has a time bound, the pool checkout included. Release It! Sec. 4.3, p. 66.
3. Each integration point has its own circuit breaker. Release It! Sec. 4.10, p. 104.
4. A circuit breaker prevents operations. It does not repeat them. Release It! Sec. 5.2, p. 115.
5. Retry happens at exactly one layer of the call path. SRE Ch. 22, p. 327.
6. Every retry uses randomized exponential backoff. SRE Ch. 22, p. 326.
7. Every retry has a budget, per request and per client. SRE Ch. 21, p. 306.
8. A deadline starts at the top of the stack. Each hop passes the remainder. SRE Ch. 22, p. 328.
9. Each integration point has its own resource pool. Release It! Sec. 5.3, p. 119.
10. The health check runs on the pool and the path that serve work. Release It! Ch. 2, p. 31.
11. Measure the vendor with your own synthetic transaction. Release It! Ch. 13, p. 231.
12. A latency metric that omits timed-out calls is wrong. Release It! Ch. 16, p. 259.
13. Record the degraded mode before the outage. Release It! Sec. 5.2, p. 117.
14. A feature promise never exceeds the worst dependency in that feature. Release It! Sec. 4.10, p. 103.
15. An automated remediation is idempotent, rate limited, observable, and reversible. SRE Ch. 7, p. 113, p. 118. SRE Ch. 13, pp. 190-191.
16. Every automated remediation has a kill switch. SRE Ch. 13, p. 195.
17. An empty target set is an error. It never means "all". SRE Ch. 7, pp. 117–118.
18. Restore service first. Preserve the evidence as you do it. SRE Ch. 12, p. 173.
19. Identify the source of the fault before you restart a process. SRE Ch. 22, p. 340.
20. Test the failure path with a test harness. A mock cannot produce these faults. Release It! Sec. 5.7, p. 138.
21. The observability plane must never stop the serving plane. Release It! Ch. 17, p. 302.

## 6.2 The catalog scan

Run `references/integration-fault-catalog.md` against the change. Each entry gives a
**Signature**, a **Consequence**, and a **Fix**. The catalog is the source of truth for the
`I-` codes that `code-review` and `verify` cite.

Report format:

```md
### Integration Fault Scan
| Fault | Present | Evidence | Required Fix |
|---|---|---|---|
| I-02 No read timeout | Yes | `vendor/client.py:41` calls `requests.get` with no `timeout=` | Set an explicit read timeout. Feed the expiry to the breaker. |
| I-16 One breaker shared by many dependencies | No | One breaker per host, keyed by vendor id | — |
```

List only the faults that apply. "Not applicable. This change makes no outbound call." is a
valid result. That result is better than an invented finding.

---

# 7. INTEGRATION RECORD OUTPUT

Produce one record per integration point. Every line is a commitment that `verify` and
`code-review` can check later.

```md
## Integration Record: [Vendor] / [Integration point]

### Identity
- Vendor and operation: [name, base address, operation]
- Owner on our side: [team]
- Owner on the vendor side: [contact, support hours, escalation path]
- Client: [SDK name and version, or our own client]
- Route: [interface, proxy, firewall rule, network path]
- Criticality: [critical / important / sheddable] — because [user-visible reason]
- Vendor change calendar: [where it lives, or "none"]

### Failure Model
| Failure class | Possible here? | How we detect it | How we survive it |
|---|---|---|---|
| Connection refused | | | |
| Connection waits in the listen queue | | | |
| Connection accepted, no answer | | | |
| Answer arrives slowly | | | |
| Answer is protocol garbage | | | |
| Answer is a well formed error | | | |

### Budgets
| Control | Value | Where the value came from |
|---|---|---|
| Connect timeout | | |
| Read timeout | | |
| Deadline received from the caller | | |
| Deadline sent to the vendor | | |
| Pool size | | |
| Pool checkout timeout | | |
| Retry budget, per request | | |
| Retry budget, per client | | |
| Breaker failure threshold | | |
| Breaker reset time | | |
| Maximum response size | | |
| Maximum result count per request | | |

> Every value needs a source. "The default" is not a source.

### Isolation
- Pool: [dedicated, or shared with what]
- Thread that makes the call: [request thread / worker thread]
- Cap on in-flight requests to this vendor: [number]
- Cache class: [latency cache / capacity cache / none]
- What still works when this point is down: [list]

### Degraded Mode
- Chosen mode: [cached / queued / default / partial / refusal]
- Approved by: [name, date]
- What the user sees: [exact text or behavior]
- How we exercise it: [harness port, fault switch, or scheduled drill]

### Detection
| Signal | Where we measure it | Threshold | Alert owner |
|---|---|---|---|
| Synthetic transaction result | | | |
| Error rate, by class: network, protocol, application | | | |
| Latency distribution, with timeouts counted | | | |
| Breaker state and transition count | | | |
| Pool checkout wait and blocked-thread count | | | |
| Concurrent requests and high-water mark | | | |

### Localization Aids
- Correlation identifier: [field name, where we set it, where we log it]
- Known-good probe: [what it calls, from which locations]
- Change log: [where it lives, what it records]
- The two latency numbers: [caller-observed latency, vendor-reported latency]
- The evidence that names the network: [packet capture point, or a probe from a second route]

### Remediation
| Action | Trigger | Precondition | Stop condition | Automatic? | Reversible? |
|---|---|---|---|---|---|

- Kill switch: [where it is, who may use it, when we last tested it]
- Rate limit on the remediation: [value]
- What each remediation records: [actor, action, target, time, result]
- Override state: [where "manual override applied" is published]

### Availability Arithmetic
- Vendor SLA: [number, or "none"]
- Every other dependency in this feature: [list with numbers]
- Product of the dependencies: [number]
- Our promise for this feature: [number]
- Statement: [our promise sits at or below the product, or we accept the gap and say why]

### Fault Scan
[Table from Section 6]
```

---

# 8. LANGUAGE DISCIPLINE (ANTI-HAND-WAVING)

These phrases are **forbidden** in a design, a review, or an incident note. Each one names a
mechanism that must replace it.

| Forbidden | Must be replaced with |
|---|---|
| "we call the vendor API" | The connect timeout, the read timeout, and the deadline |
| "we retry on failure" | The error classes, the backoff, the budget, and the one layer that owns the retry |
| "it has a circuit breaker" | The threshold, the reset time, the failure types counted, and the exception the caller receives |
| "the vendor is down" | The signature you measured, the location you measured from, and the probe that confirms it |
| "their status page is green" | The result of your own synthetic transaction |
| "it will self-heal" | The action, the precondition, the stop condition, the rate limit, and the kill switch |
| "we fail gracefully" | The degraded mode, the exact user-visible behavior, and the test that exercises it |
| "we have a health check" | What the check executes, which pool serves it, and what a pass proves |
| "we monitor it" | The metric names, where you measure them, and the alert threshold |
| "the timeout is generous" | The number, and the latency distribution it came from |
| "we restart it" | The precondition, the canary step, the rate limit, and the evidence that a restart cannot amplify the fault |
| "it is idempotent" | The key, where the key lives, and how long the record survives |
| "their SLA covers it" | The arithmetic across every dependency in this feature |
| "we cache it as a fallback" | The size bound, the staleness bound, and the behavior on a cold cache |
| "the SDK handles it" | The exact call, the timeout you can set, and what happens when the SDK hides the socket |
| "we page someone" | The alert condition, its expected rate, and the runbook entry it links to |
| "it is a transient error" | The error class the vendor returns, and how it differs from a permanent error |
| "we degrade under load" | The utilization signal, the threshold, and the traffic class you shed first |

---

# 9. REFERENCE INDEX

| Reference | Sources | Read it when |
|---|---|---|
| `references/01-integration-points.md` | Release It! Sec. 4.1, Sec. 9.9 | You add or review any outbound call |
| `references/02-stability-antipatterns.md` | Release It! Ch. 4 | You need the name of the failure you are looking at |
| `references/03-stability-patterns.md` | Release It! Ch. 5 | You choose a defense |
| `references/04-fault-localization.md` | SRE Ch. 12, Release It! Ch. 16 | A vendor looks faulty and you must prove where the fault is |
| `references/05-overload-and-load-shedding.md` | SRE Ch. 21 | Quotas, throttles, criticality, deadlines, retry budgets |
| `references/06-cascading-failure.md` | SRE Ch. 22, Release It! Sec. 4.2, Sec. 4.3 | The failure crossed a layer boundary |
| `references/07-automated-remediation-safety.md` | SRE Ch. 7, Ch. 13, Release It! Sec. 4.6 | You want the system to act without a human |
| `references/08-fault-injection-and-test-harness.md` | Release It! Sec. 5.7, SRE Ch. 17 | You must prove the failure path works |
| `references/09-vendor-slas-and-degradation.md` | Release It! Sec. 4.10, Sec. 13.1, SRE Ch. 4 | You promise an availability number, or you choose a degraded mode |
| `references/integration-fault-catalog.md` | Cross-cutting | Every review, and every incident |

## Who owns the overlapping material

| Overlap | Owner | What this skill does instead |
|---|---|---|
| The pipeline, the phase order, and the state in `plan.md` | `engineer-workflow` | This skill is a gate inside that pipeline. It runs no workflow. |
| Reconnaissance, root-cause analysis, phased plans, capacity estimation | `planner` | This skill supplies the integration inventory and the failure model. |
| The phase specification, the retry contract, and the rollback plan | `detail-planning` | This skill supplies the budget values and the degraded-mode decision. |
| The code invariants: timeouts present, jittered retries, bounded result sets | `implement` | This skill decides the values and the classes. `implement` enforces them. |
| The failure-mode matrix and the evidence a phase is complete | `verify` | This skill supplies the fault scan and the harness fault list. |
| Severity-ranked findings, review modes, merge verdicts | `code-review` | This skill supplies the `I-` catalog. Cite `I-` codes. |
| Component selection, load balancers, caches, queues, database choice | `system-design` | This skill chooses no architecture. |
| Idempotency, deduplication, dual writes, the outbox, isolation levels | `data-systems-design` | Entry `I-14` names the point and defers the mechanism. |
| `EINTR`, short reads and writes, socket options at the kernel boundary | `systems-programming` | Reference 01 covers the network behavior a caller observes. |
| TLS, certificate checks, webhook signatures, credential handling | `security-engineering` | This skill covers availability faults we inflict on ourselves. |
| The six-step troubleshooting loop, bisection, negative results, the fault record | `production-troubleshooting` | Section 3 adds only the vendor boundary. Table 2 maps a signature to us, the network, or the vendor. |
| Alert rules, dashboards, log levels, the four golden signals, the operations database | `observability` | This skill names the eleven signals one integration point must expose, and defers the alert design. |
| SLI, SLO, SLA, error budgets, availability math, the dependency ceiling | `slo-engineering` | Reference 09 applies that arithmetic to one vendor and chooses the degraded mode. |
| Load tests, bottleneck hunting, pool sizing, utilization thresholds, headroom | `capacity-engineering` | This skill sets the per-vendor budget values. That skill measures the limit. |
| Incident roles, the war room, the live incident record, the blameless postmortem | `incident-response` | This skill supplies the localization evidence and the fault scan. |
| Rollback, revert, canary, feature flags, checkpoint commands, automation safety | `reverse-branching` | Reference 07 states the four properties for a self-healing action. That skill owns the undo path. |

**One boundary needs both skills.** When the integration moves money or writes a record, run
this skill **and** `data-systems-design`. This skill decides whether to retry. That skill
decides whether the retry is safe.

**Do not restate a neighbor skill.** When a finding belongs to a row above, name the owner
skill and stop. A duplicated method drifts, and then two skills disagree.

---

# 10. PROPORTIONALITY RULE (ANTI-OVER-ENGINEERING)

Rigor scales with blast radius. A full Integration Record for a change that adds one field is
itself a failure mode.

## Table 9 — Required output per change class

| Change class | Required output |
|---|---|
| A change with no call to a party we do not operate | Nothing from this skill |
| A new call to an existing integration point that is already protected | The fault gate only |
| A new integration point, or a new vendor SDK | The fault gate, plus the Identity, Failure Model, Budgets, and Degraded Mode sections |
| A change to a timeout, a retry, a breaker, a pool, or a deadline | The fault gate, plus the Budgets section and its arithmetic |
| A new automated remediation | The full Integration Record, plus Table 7 filled in |
| An integration that moves money, or that cannot be reversed | The full Integration Record, plus a human approval step in Table 7 |

**Say "not applicable" when it is true.** A change that adds a field to an internal handler
needs no fault model. An honest "not applicable, this change makes no outbound call" is the
preferred result.

**Simplicity remains a goal.** One vendor, one pool, one breaker, and one recorded degraded
mode beat an elaborate remediation ladder that nobody exercises.

---

**Attribution:** the concepts, the pattern names, and the page references in this skill and in
its references come from two books. *Release It! Design and Deploy Production-Ready Software*
by Michael T. Nygard (Pragmatic Bookshelf, 2007) supplies the failure classes, the stability
antipatterns, and the stability patterns. *Site Reliability Engineering: How Google Runs
Production Systems* by Beyer, Jones, Petoff, and Murphy (O'Reilly, 2016) supplies the
troubleshooting procedure and the four golden signals. It also supplies the overload
material, the cascading-failure material, and the automation safety rules. Page numbers refer to the PDF pages of the
extracted editions. Content marked **Modern** postdates both books.
