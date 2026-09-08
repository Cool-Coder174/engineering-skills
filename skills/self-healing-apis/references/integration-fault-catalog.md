# Integration Fault Catalog

**Sources:** *Release It! Design and Deploy Production-Ready Software*, Michael T. Nygard
(Pragmatic Bookshelf, 2007), Ch. 2 to Ch. 17. *Site Reliability Engineering: How Google Runs
Production Systems*, Beyer, Jones, Petoff and Murphy (O'Reilly, 2016), Ch. 6, Ch. 7, Ch. 12,
Ch. 13, and Ch. 19 to Ch. 24.

A catalog of named failure modes for code that calls a party we do not operate. An integration
point is any socket, process, pipe, or remote procedure call that leaves our process. Nygard
states that integration points are the number-one killer of systems
(Release It! Sec. 4.1, p. 46).

Each entry has three parts:

- **Signature** — what the code, the configuration, the design, or the process looks like.
  This is the text that you search for.
- **Consequence** — what goes wrong, and when.
- **Fix** — the required remedy.

**How to use it.** `../SKILL.md`, Section 6.2 runs the applicable groups against a change.
`code-review` runs them against a difference. `verify` runs them against an implemented
phase. `planner` and `detail-planning` run them against a design. Report only the faults that
apply. "Not applicable. This change makes no outbound call." is a valid result. That result
is better than an invented finding.

**Severity.** 🔴 causes data loss, corruption, or an unrecoverable state. It blocks the merge.
🟡 causes an outage or a wrong result under load. 🔵 risks operation or maintenance. Every
entry names a chapter and a page. An entry for practice the books predate carries **Modern**.

---

## A. Transport and connection

### 🟡 I-01 — No connect timeout
**Signature:** a client or a socket built with no connect timeout. `new Socket(host, port)`.
`http.Client{}` with no `Timeout` field. `requests.get(url)` with no `timeout` argument.
**Consequence:** the vendor accepts the SYN packet into a full listen queue and sends no answer.
The calling thread blocks inside the kernel. Nygard writes that connect timeouts are usually
measured in minutes.
**Fix:** set an explicit connect timeout on every client. Count each expiry in the Circuit
Breaker for that integration point. Release It! Sec. 4.1, p. 49.

### 🟡 I-02 — No read timeout
**Signature:** no call to `Socket.setSoTimeout`. A read on `HttpURLConnection`. A vendor SDK
that hides the socket and exposes no timeout option.
**Consequence:** the thread waits with no end. The remote side can send one byte every thirty
seconds and the thread stays. With no application timeout the code inherits the retransmit
timeout of the operating system. Linux 2.6 with `tcp_retries2 = 15` gives twenty minutes. HP-UX
gave thirty minutes. A read can block forever.
**Fix:** set a read timeout on every call. Choose a client that exposes separate connect and
read timeouts. When the SDK hides the socket, run the call on a worker thread and abandon it at
expiry. Release It! Sec. 4.1, pp. 49, 54–57. Release It! Sec. 4.5, p. 86.

### 🟡 I-03 — An idle pooled connection crosses a stateful middlebox
**Signature:** a long-lived connection pool, a firewall or a NAT device in the route, and no
keepalive traffic. Search for a pool with no validation query and no keepalive setting.
**Consequence:** the middlebox drops the table entry after its idle timeout. It sends no reset
packet and no ICMP message. Neither endpoint learns. The pool locks at the next traffic
increase. The fault appears at the same time every day.
**Fix:** enable keepalive traffic that resets the timer in the middlebox. Nygard used Oracle
dead connection detection. Know the checkout order of the pool, because a LIFO pool leaves most
connections idle under low traffic. Release It! Sec. 4.1, pp. 53–56.

### 🔴 I-04 — A write is replayed after a failover
**Signature:** a retry that treats an `IOException` after a virtual IP failover the same as a
"destination unreachable" error. A retry of an insert or an update with no key.
**Consequence:** the first attempt may have landed. The replay creates a second record or a
second payment. Oracle drivers re-execute an aborted query. They cannot repeat an update, an
insert, or a stored procedure call.
**Fix:** classify the failover error apart from an unreachable host. Retry reads under the
limits of the breaker. Attach an idempotency key to every write. Otherwise read the state of the
operation at the vendor first. Release It! Sec. 11.3, pp. 224–225. SRE Ch. 24, p. 392.

---

## B. Timeouts and deadlines

### 🟡 I-05 — No deadline on the outbound call
**Signature:** an outbound call with no deadline field and no context deadline.
**Consequence:** a request whose problem started long ago keeps consuming server resources. The
consumption continues until the process restarts.
**Fix:** set a deadline on every outbound call. SRE Ch. 22, p. 328.

### 🟡 I-06 — The deadline is invented at each hop
**Signature:** a hard-coded timeout constant inside a service that another service calls.
**Consequence:** the inner service works on a request that the outer caller already abandoned.
In Google's example, A gives B ten seconds. B spends eight seconds. B then gives C twenty
seconds. C works fifteen seconds on dead work.
**Fix:** propagate the deadline. Subtract the elapsed time at each hop. Subtract a few hundred
milliseconds for transit. SRE Ch. 22, pp. 328–329.

### 🟡 I-07 — The deadline is far above the observed latency
**Signature:** a deadline three orders of magnitude above the mean latency. A deadline of one
hundred seconds on a call whose mean is one hundred milliseconds.
**Consequence:** latency becomes bimodal. A small share of calls never finish and hold a thread
for the whole deadline. Google's worked case turned a 5% underlying fault into an error rate of
80.4%.
**Fix:** set the deadline from the observed latency distribution, not from the mean. Use the
fail-fast option of the RPC layer when a backend is unavailable. SRE Ch. 22, pp. 330–331.

### 🟡 I-08 — A blocking wait with no time bound
**Signature:** `connectionPool.getConnection()` with no checkout timeout. `Object.wait()` with
no argument. `poll()`, `offer()`, or `tryLock()` with no timeout argument.
**Consequence:** every request-handling thread blocks on checkout when the calls stop returning.
The process runs and completes no work. This is the mechanism of the airline outage and of the
Black Friday outage.
**Fix:** give every blocking wait a timeout. Nygard states that a safe resource pool always
limits the time a thread waits for a resource. Release It! Sec. 4.3, p. 66.
Release It! Sec. 5.1, p. 112. Release It! Ch. 16, p. 259.

### 🔵 I-09 — No cancellation for a doomed call
**Signature:** a call tree in which a deep failure or a deep timeout never reaches the calls
above it. No cancellation token and no context cancellation.
**Consequence:** the outer call holds threads, memory, and connections until its own deadline
expires. The work is already dead. A hedged request also leaks the copies that lost.
**Fix:** propagate the cancellation up the call stack as soon as a call becomes doomed. Cancel
the rest of the call tree. SRE Ch. 22, p. 330.

---

## C. Retries and load amplification

### 🟡 I-10 — Immediate retry with no randomized backoff
**Signature:** a retry loop with a fixed sleep, or with no sleep. A constant retry interval.
**Consequence:** an overload of 1% becomes a flood that sustains itself. Google's worked case
grows from 100 to 200 to 300 extra queries each second until the backend fails. With no random
jitter, retry ripples align and amplify.
**Fix:** use randomized exponential backoff on every retry. SRE Ch. 22, p. 326.
Release It! Sec. 5.1, p. 113.

### 🟡 I-11 — Retry at more than one layer
**Signature:** retry configuration present in two or more layers of one call path. A retry in
the SDK, a retry in our client wrapper, and a retry in the caller.
**Consequence:** attempts multiply. Three retries at each of three layers give 64 attempts
against the vendor for one user action.
**Fix:** retry only at the layer directly above the layer that rejects. Every other layer
returns "overloaded, do not retry", or it returns a degraded answer. SRE Ch. 22, p. 327.
SRE Ch. 21, p. 310.

### 🟡 I-12 — No retry budget
**Signature:** a retry policy with a maximum attempt count and nothing else. No per-client
ratio. No process-wide retry counter.
**Consequence:** a cap of three attempts alone still permits request volume to grow to just
below 3x during an overload.
**Fix:** set a budget of three attempts for each request. Add a per-client budget that stops
retries above a retry ratio of 10%. Consider a process-wide budget, for example 60 retries each
minute. SRE Ch. 21, p. 306. SRE Ch. 22, p. 327.

### 🟡 I-13 — Retry on a permanent error
**Signature:** a `catch` block that names the base exception type and then calls a retry helper.
A retry on any status other than 200. A retry that a comment justifies by past experience
instead of by an error class.
**Consequence:** the retry never succeeds and it consumes the budget. When the error taxonomy of
the vendor cannot separate transient from permanent, the caller sends more calls. In Nygard's
"Hammer Time" case the caller spent 100% of its CPU on calls and on logging.
**Fix:** classify each error as retriable or permanent. Never retry a permanent error or a
malformed request. Ask the vendor for a distinguishable overload status. HTTP 429 and the
`Retry-After` header are **Modern**. SRE Ch. 22, p. 327. Release It! Sec. 4.3, p. 67.

### 🔴 I-14 — Retry of a non-idempotent write with no idempotency key
**Signature:** a POST, a charge, a transfer, or a message send inside a retry loop, with no
client-generated key in the request.
**Consequence:** the first attempt landed and the answer was lost. The retry creates a second
record. Money moves two times. Recovery from a double execution can be impossible.
**Fix:** generate the key before the first attempt. Send the same key on every attempt. Require
either an idempotent operation or an unambiguous state query at the vendor. The
`Idempotency-Key` header is **Modern**. SRE Ch. 24, p. 392. Release It! Sec. 11.3, p. 224.

---

## D. Circuit breakers

### 🟡 I-15 — No circuit breaker on the integration point
**Signature:** a direct call to a vendor client with no wrapper that counts failures.
**Consequence:** the caller keeps calling a sick dependency. Threads accumulate in the call. The
problem of the vendor becomes our outage.
**Fix:** wrap the call in a Circuit Breaker. Closed passes the call and counts failures. At the
threshold the breaker trips to Open. Open fails at once. After a timeout the breaker moves to
Half-Open and passes exactly one trial call. Success resets it to Closed. Failure returns it to
Open. Release It! Sec. 5.2, pp. 115–117.

### 🟡 I-16 — One breaker shared by many dependencies
**Signature:** a single global breaker object, or a breaker keyed by nothing.
**Consequence:** one sick vendor opens the circuit for every vendor. A healthy dependency
becomes unreachable. The breaker also stops telling you which dependency failed.
**Fix:** create one breaker for each integration point. Nygard writes the remedy in the plural.
He tells you to use circuit breakers to protect the application from each of the allies.
Release It! Sec. 4.10, p. 104.

### 🔵 I-17 — The open-state failure looks like an ordinary failure
**Signature:** the breaker throws the same exception when it is open as when the call fails.
**Consequence:** the caller cannot choose a different path for "the dependency is off" and for
"this one call failed". The degraded mode never runs.
**Fix:** throw a different exception when the circuit is open. Report a system failure apart
from an application failure, so a user who types bad data does not trip an upstream breaker.
Release It! Sec. 5.2, p. 116. Release It! Sec. 5.5, p. 132.

### 🔵 I-18 — Breaker state is not exposed and not controllable
**Signature:** a breaker with no state metric, no log line for a state change, and no entry
point that trips or resets it.
**Consequence:** nobody can tell which dependency is at fault. The frequency of state changes is
a leading indicator of trouble elsewhere, and it is lost. Operations cannot disable a bad
integration without a deploy.
**Fix:** log every state change. Expose the current state, the count of failed calls, the time
of the last success, and the count of state transitions. Give operations a way to trip and to
reset the breaker directly. Publish a "manual override applied" flag. Release It! Sec. 5.2, pp.
117–118. Release It! Ch. 17, p. 271.

---

## E. Bulkheads and capacity

### 🟡 I-19 — One resource pool serves every integration point
**Signature:** a single connection pool or thread pool used by two or more vendors.
**Consequence:** one sick vendor drains the pool. Every other vendor becomes unreachable. The
shared pool also removes the only throttle you could have used during the incident.
**Fix:** give each integration point its own pool. In the Black Friday case a separate pool for
one vendor was the only available throttle, and it saved the retail weekend.
Release It! Sec. 5.3, p. 119. Release It! Ch. 16, p. 262.

### 🟡 I-20 — The request-handling thread makes the vendor call
**Signature:** a controller or a handler that calls the vendor client on the request thread.
**Consequence:** a remote problem becomes local downtime. Nobody can abandon the thread. Nygard
states that the coupling of request-handling threads to external integration calls turned a
remote problem into downtime.
**Fix:** give the dangerous call to a worker thread. Rendezvous on a result object. Abandon the
call at timeout. Release It! Sec. 3.3, p. 40. Release It! Sec. 4.5, p. 86.

### 🟡 I-21 — Unbalanced capacity across the boundary
**Signature:** a front-end thread count far above the stated concurrency of the vendor. Compare
the production ratio against the test ratio.
**Consequence:** the front end can always overwhelm the back end. Nygard's case ran 3,000
threads into 450 threads into a system that handled about 25 concurrent requests. He writes that
three thousand threads calling into seventy-five threads is not in the ballpark.
**Fix:** cap the in-flight requests for each dependency. Apply the breaker on our side and ask
for Handshaking on theirs. Reserve back-end capacity for each transaction type.
Release It! Sec. 4.8, pp. 97–99. Release It! Ch. 16, p. 261.

### 🟡 I-22 — A capacity cache sits in front of the vendor
**Signature:** a cache whose absence makes the service unable to carry the expected load. Search
for a service that cannot start under load without a warm cache.
**Consequence:** a restart, a new cluster, or a cache flush becomes an outage. Every request
becomes a vendor call.
**Fix:** classify the cache. A latency cache serves the expected load when it is empty. A
capacity cache does not. Make every new cache a latency cache, or engineer it well enough to be
a safe capacity cache. Increase load slowly to warm it. SRE Ch. 22, pp. 332–333.

### 🔵 I-23 — The router treats a fast error as health
**Signature:** a router or a client that selects the least-loaded endpoint by active requests.
**Consequence:** an endpoint that fails fast answers faster than a healthy one. The router sends
it more traffic. Google names this effect sinkholing.
**Fix:** count recent errors as if they were active requests. Never infer health from a short
response time alone. SRE Ch. 20, p. 295.

---

## F. Health checks and probes

### 🟡 I-24 — The health check runs on a pool that does not serve work
**Signature:** a health endpoint that returns a constant. A check served by a thread pool that
is not the pool of the work path.
**Consequence:** the check stays green through a total outage. In the airline case the monitor
probed an HTTP status page while every business thread was blocked. Detection came from the
downstream systems instead.
**Fix:** make the check execute the real work path, including the pool checkout. Give the check
its own timeout, because a check that traverses the broken path with no timeout hangs exactly
when it matters. Release It! Ch. 2, pp. 25, 31. Release It! Sec. 4.1, p. 55.

### 🔵 I-25 — The probe does not record the failing response
**Signature:** a prober that stores only success or failure.
**Consequence:** the responder must repeat the call by hand to learn that the answer was a 502
with no body. A self-healing loop cannot classify a failure that it never captured.
**Fix:** store the status, the headers, and the body of every failing probe. SRE Ch. 12, pp.
175–176.

### 🔵 I-26 — The status page of the vendor is the health signal
**Signature:** an alert rule, a runbook step, or a dashboard tile that reads the public status
page of the vendor. Vendor status pages are **Modern**.
**Consequence:** the page reports what the vendor believes about all customers. It says nothing
about our traffic, our route, or our account. It also lags the fault.
**Fix:** measure the vendor with your own synthetic transaction, on a defined cadence, from a
defined number of locations, against defined success patterns. Use the page of the vendor as
corroboration only. Release It! Ch. 13, pp. 231–232.

### 🟡 I-27 — Process health check conflated with service health check
**Signature:** one check answers both "is the binary running?" and "can it serve this class of
request now?". The Kubernetes split of liveness and readiness is the **Modern** form.
**Consequence:** the scheduler kills tasks that are merely overloaded. Health checking itself
keeps the service unhealthy. During a cascade, half the tasks start while the other half are
killed, and no task ever serves.
**Fix:** separate the two checks. Give the scheduler the process check. Give the load balancer
the service check. During a cascade, disable the kills that health checks drive until the tasks
stabilize. SRE Ch. 22, p. 339.

### 🔵 I-28 — The start-up path does not probe the integration point
**Signature:** an application that starts with an empty connection pool and makes the first
vendor call for a customer.
**Consequence:** a broken integration point stays hidden until the first customer request. An
exit at start-up also destroys the state that a responder needs.
**Fix:** create some connections at start-up as a self test. Nygard calls this another form of
Fail Fast. Report a failure state that a responder can still query. Do not exit.
Release It! Ch. 14, p. 247.

---

## G. Payload and protocol

### 🔴 I-29 — Unbounded result set or unbounded response body
**Signature:** a vendor query with no limit parameter. A loop over every item of a response. An
association traversal with no bound. No maximum body size on the client.
**Consequence:** memory exhaustion. Nygard's "Black Monday" case read a table that should hold
1,000 rows and held more than ten million. Every instance crashed, released the lock, and the
next instance repeated the crash. The team could keep no more than 25% of capacity running.
**Fix:** state the maximum that you accept in the request. Nygard writes that in any API or
protocol the caller should always state how much of a response it accepts. Set a maximum body
size. Test with production-sized data. Release It! Sec. 4.11, pp. 106–109. Sec. 5.7, p. 138.

### 🟡 I-30 — The response is parsed with no status, content-type, or schema check
**Signature:** `json.loads(response.text)` or an equivalent, with no status check and no
content-type check.
**Consequence:** a proxy error page, a login redirect, or an HTML body reaches the parser. A
test harness must be able to send HTML where XML is expected, because real systems do.
**Fix:** check the status, then the content type, then the schema. Write cynical code that
expects violations of form and of function. Release It! Sec. 5.7, pp. 137–138.
Release It! Sec. 4.1, p. 59.

### 🔴 I-31 — HTTP 200 treated as success
**Signature:** a success test that reads only the status code, on a vendor that reports failures
inside the body.
**Consequence:** an implicit error. The response is a 200 with the wrong content. The wrong
value is stored and the error rate stays at zero. Only an end-to-end test detects it.
**Fix:** define the error taxonomy for this vendor across three classes: explicit errors,
implicit errors, and errors by policy. Add a secondary internal protocol when the status codes
cannot express the failures. SRE Ch. 6, p. 88.

### 🟡 I-32 — A vendor SDK callback runs on a thread the caller does not control
**Signature:** `registerCallback`, `messageReceived`, or a listener interface from a vendor SDK.
**Consequence:** you cannot know which thread calls you or which monitors it holds. A slow or
synchronized callback plugs the threads of the library like a plugged drain. That blocks
`send()`, which then blocks the request handlers.
**Fix:** make the callback place work on your own queue and return at once. Acquire locks in a
consistent order. Inspect the library. Ask the vendor for a better client. Release It! Sec. 4.1,
pp. 57–58. Release It! Sec. 4.5, p. 87.

---

## H. Telemetry and localization

### 🟡 I-33 — The latency metric excludes timed-out calls
**Signature:** a latency metric recorded only on the success path.
**Consequence:** the statistic reports the average of the calls that finished. Calls that never
finished never enter the average. A total outage reads as a mild slowdown.
**Fix:** count timeouts and abandoned calls separately. Record the latency distribution, not the
mean. Release It! Ch. 16, pp. 258–259. SRE Ch. 6, pp. 89–90.

### 🟡 I-34 — Only one side of the boundary is instrumented
**Signature:** caller-side latency exists, and the timing that the vendor reports does not.
**Consequence:** you cannot separate a slow vendor from a slow network. Every incident becomes
an argument between two teams.
**Fix:** record the caller-observed latency and the vendor-reported latency for the same call.
The difference is the network and the queueing. Ask the vendor to return its own service time.
SRE Ch. 6, p. 87.

### 🔵 I-35 — No metric set for the integration point
**Signature:** the dashboard shows aggregate service health and no panel for each dependency.
**Consequence:** an aggregate up-or-down signal cannot localize a vendor fault. The problem is
worse behind an open breaker, where the system is up and one feature is not.
**Fix:** expose eleven values for each integration point. They are breaker state, timeout count,
request count, and average response time. They are good-response count, network-error count,
protocol-error count, and application-error count. They are the remote endpoint address,
concurrent requests, and the high-water mark. Track the four golden signals for each vendor.
Release It! Ch. 17, p. 298. SRE Ch. 6, p. 88. `observability` owns the alert rule that reads
these signals.

### 🔵 I-36 — No correlation identifier crosses the boundary
**Signature:** log lines with no request identifier. No identifier sent to the vendor and no
identifier recorded from the answer of the vendor.
**Consequence:** you cannot join our log entries to their log entries. Ten thousand log lines
after an outage have no anchor.
**Fix:** generate one identifier at the edge. Propagate it through every hop. Log it on both
sides. Record the own identifier of the vendor from the response. W3C trace context is
**Modern**. SRE Ch. 12, p. 186. Release It! Ch. 17, p. 283.

### 🔵 I-37 — No change log to correlate onset with a push
**Signature:** no record of binary versions, configuration changes, or vendor maintenance
windows on a common timeline.
**Consequence:** you cannot answer "what touched it last". You also cannot state with confidence
that no change sits inside the window, and that answer matters equally.
**Fix:** record deployments and configuration changes at every layer. Annotate the error graph
with the start and the end of each push. Include the change calendar of the vendor.
SRE Ch. 22, p. 335. SRE Ch. 12, pp. 177–178.

---

## I. Degradation, SLA, and inventory

### 🟡 I-38 — Availability promised above the worst dependency
**Signature:** an availability number written for "the system" while a dependency inside that
feature offers a lower number, or no number at all.
**Consequence:** the promise is arithmetic that does not hold. Five dependencies at 99.9% cap
the feature at 99.5%. A dependency with no SLA means the feature can carry no SLA.
**Fix:** write the SLA for each feature, not for the system. Multiply the dependency
availabilities. State a number at or below the product. Write exclusions for loss that external
systems cause. Release It! Sec. 4.10, pp. 103–105. Release It! Ch. 13, p. 231.

### 🟡 I-39 — No written degraded mode for the integration point
**Signature:** no entry in the Integration Record. No behavior documented for an open circuit.
**Consequence:** the team improvises during the incident. In the Black Friday case the
responders had to ask the developers what the code does when the pool returns null. Only then
could they apply the one available throttle.
**Fix:** choose a row of Table 8 in `../SKILL.md`. Ask the business sponsor to approve it. Write
the exact user-visible behavior. Release It! Sec. 5.2, p. 117. Release It! Ch. 16, p. 262.

### 🟡 I-40 — The degradation path is never exercised
**Signature:** a fallback branch with no test that reaches it. No harness port for this vendor.
No scheduled drill.
**Consequence:** Google states that the code path you never use is the code path that often does
not work. An untested failure path is a broken failure path.
**Fix:** build a Test Harness for this integration point. Produce the thirteen faults listed in
Section 5 of `08-fault-injection-and-test-harness.md`. Send a small share of traffic against the
degraded path on a schedule. Alert when too many callers enter the degraded mode. SRE Ch. 22, p.
324. Release It! Sec. 5.7, pp. 136–140.

### 🔵 I-41 — The integration point is absent from the dependency inventory
**Signature:** a firewall rule, an outbound host, or an SDK with no entry in the inventory.
**Consequence:** nobody can enumerate the feeds. Nygard's replatform case could not list its own
integrations well enough to open firewall rules. He believes every synchronous integration point
there caused at least one outage.
**Fix:** keep a record for each integration point with the destination name, the address, and
the desired route (Sec. 11.2, p. 222). Add the business stakeholder, the transport, the
frequency, and the volume (Sec. 4.1, p. 47). Add the support hours and the SLA of the vendor
(Sec. 4.10, p. 105). A hole in the firewall exists to call some other system, and that is an
integration point (Sec. 14.1, p. 243).

---

## J. Automated remediation safety

### 🔴 I-42 — Remediation with no rate limit
**Signature:** an automated action that iterates a target list with no cap on actions each
minute and no cap on total targets.
**Consequence:** the automation applies a defect at machine speed. Google's decommission
automation wiped the disks of almost every machine in every colocation site within minutes.
**Fix:** rate limit every automated remediation. Cap the fraction of the fleet that one run may
touch. Add sanity checks to the commands that the automation sends. SRE Ch. 7, pp. 117–118.
SRE Ch. 13, p. 196.

### 🔴 I-43 — An empty target set is treated as "all"
**Signature:** a filter, a selector, or a query whose empty result passes downstream with no
check. `if not targets: targets = all_hosts`. A selector built from an empty list.
**Consequence:** the degenerate input becomes the widest possible action. Google records the
lesson in one line: sometimes zero does mean all.
**Fix:** treat an empty result as an error. Filter it before it reaches the executor. Refuse to
act. SRE Ch. 7, pp. 117–118. SRE Ch. 13, p. 196.

### 🔴 I-44 — The remediation workflow is not idempotent
**Signature:** a multi-step remediation that restarts from the first step after a partial
failure, with no record of which steps completed.
**Consequence:** completed steps run a second time. The second run reinterprets state that the
first run already changed.
**Fix:** make the workflow idempotent. Record two synchronization points for each action: one
before it, and one after it. Name the action before you perform it. After a failure, read the
state of every named action and perform only the missing ones. SRE Ch. 7, p. 118.
SRE Ch. 24, pp. 391–392.

### 🟡 I-45 — No kill switch and no override state
**Signature:** an automated remediation with no single control that stops it, and no published
flag that says a human or an automation pinned the component.
**Consequence:** nobody can stop the automation during an incident. Google's response to a
runaway automation was to disable all team automation, and that response needs such a control.
**Fix:** build one control that stops every automated action. Test it. Publish a "manual
override applied" state for every integration point and every breaker. SRE Ch. 13, p. 195.
Release It! Ch. 17, p. 271. Release It! Sec. 5.2, p. 117.

### 🟡 I-46 — A restart runs before the fault is localized
**Signature:** an automatic restart that a health-check failure alone triggers, with no canary.
**Consequence:** the restart amplifies the fault. A cold cache, a crash loop, or a poisoned
downstream dependency all become worse. In the airline case a restart of the shared dependency
healed one consumer and not the other.
**Fix:** identify the source of the cascade first. Confirm that the restart will not merely move
load. Canary the restart on one instance, verify the recovery, then apply it slowly and fleet
wide. SRE Ch. 22, p. 340. Release It! Ch. 2, pp. 24–25. Release It! Ch. 16, pp. 262–263.

### 🟡 I-47 — The remediation removes its own observability
**Signature:** an automated action that changes or removes monitoring, logging, or alerting.
**Consequence:** the automation blinds the responders during the incident that it caused.
Google's turndown automation removed monitoring, and the on-call engineers had to reverse those
changes before they could measure the damage.
**Fix:** never let a remediation change the observability plane. Keep that plane separate, and
make sure its failure cannot affect the serving path. SRE Ch. 13, p. 196. Release It! Ch. 17, p.
302.

### 🔵 I-48 — The remediation trigger is a temporal correlation
**Signature:** a runbook entry or an automation rule that a past coincidence justifies. The
symptom appeared shortly before the fault last time. No stated mechanism.
**Consequence:** the automation performs a superstition at machine speed. An operations group
ran a weekly production database failover for six months, because one log message had once
appeared before a crash. There was a temporal connection and no causal connection.
**Fix:** require a stated mechanism for every automated trigger. Record why the rule exists.
Remove a rule whose mechanism nobody can state. Release It! Ch. 17, pp. 281–283. SRE Ch. 12, p.
171.

---

## Index by symptom

| Symptom | Check these entries |
|---|---|
| The call never returns | I-01, I-02, I-05, I-08, I-20, I-32 |
| The error rate fell to zero and no answers arrive | I-08, I-33 |
| A duplicate record, or money that moved two times | I-04, I-14 |
| The vendor became worse after our retry started | I-10, I-11, I-12, I-13 |
| One sick vendor made every feature fail | I-16, I-19, I-20 |
| A small share of calls never finish | I-07, I-09 |
| The health check is green through an outage | I-24, I-26, I-27, I-28 |
| The stored value is wrong and the error rate is zero | I-30, I-31 |
| We cannot say whether the vendor or the network is slow | I-34, I-36 |
| We cannot say which dependency failed | I-18, I-35 |
| The feature misses its availability number | I-38, I-41 |
| The automation made the outage larger | I-42, I-43, I-44, I-46, I-47 |
