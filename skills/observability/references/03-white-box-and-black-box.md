# White-Box and Black-Box Monitoring

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 6, "Monitoring Distributed Systems", p. 82–96. Ch. 10, "Practical Alerting from
Time-Series Data", p. 141–159, and its Black-Box Monitoring section, p. 155–156.
*Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007). Ch. 16, p. 252–264.
Sec. 17.3, p. 276. Sec. 17.4, p. 283. Sec. 17.5, p. 283–289.

A system has two vantage points. White-box monitoring reads the instrumentation that the
system exports. Black-box monitoring tests the system the way a user tests it.

**You need both.** Google combines heavy use of white-box monitoring with modest but
critical use of black-box monitoring (SRE Ch. 6, p. 87). This file gives the rule for each
vantage point. It also gives the failure that each vantage point alone will miss.

---

## 1. The two definitions

**White-box monitoring reads metrics that the internals of the system expose**
(SRE Ch. 6, p. 82). The mechanism is a log, a profiling interface, or an HTTP handler that
emits internal statistics. Nygard states that a white-box technology runs inside the thing
that you observe (Release It! Sec. 17.3, p. 276). The system exposes itself on purpose. A
developer must integrate the instrumentation during development. The coupling is tight.

**Black-box monitoring tests externally visible behavior, as a user sees it**
(SRE Ch. 6, p. 82). The mechanism is a protocol check that runs outside the process. Nygard
states that a black-box technology sits outside the process (Release It! Sec. 17.3, p. 276).
Operations can add it after delivery. The coupling is loose.

**A process is opaque by default.** Nygard states that a process on a server reveals almost
nothing about itself (Release It! Sec. 17.3, p. 276). Both vantage points reduce that
opacity. Neither vantage point is optional.

---

## 2. The comparison

Sources: SRE Ch. 6, p. 82, p. 87. SRE Ch. 10, p. 155–156. Release It! Sec. 17.3, p. 276.

| | White-box | Black-box |
|---|---|---|
| Where it runs | Inside the process | Outside the process |
| Who adds it, and when | A developer, during development | Operations, after delivery |
| Mechanism | A log, a profiling interface, or a handler that emits internal statistics | A protocol check that a prober runs against a target |
| Coupling | Tight | Loose |
| It detects | An imminent problem, a failure that a retry masks, a full queue, a bottleneck | An active problem only |
| Its statement | This component is close to a limit | The system is not working correctly, right now |
| It misses | The user's view. It sees only the queries that arrive | Anything that has not happened yet |
| Orientation | Symptom-oriented or cause-oriented, by how informative the instrumentation is | Symptom-oriented |
| How much to use | Heavy use | Modest but critical use |

**The orientation of white-box monitoring is not fixed.** SRE states that white-box
monitoring is sometimes symptom-oriented and sometimes cause-oriented. The difference depends
on how informative the instrumentation is (SRE Ch. 6, p. 87). Read
`02-symptom-based-alerting.md` for the rule that decides which output a signal earns.

---

## 3. What white-box monitoring alone will miss

**White-box monitoring sees only the queries that arrive** (SRE Ch. 10, p. 156). A query
that a DNS error destroys never reaches the target. A query that a server crash destroys
makes no sound. SRE states the consequence plainly. You can alert only on the failures that
you expected (SRE Ch. 10, p. 156).

**A dead process exports nothing.** Nygard writes, "Dead processes log no tales."
(Release It! Sec. 17.4, p. 283). A hung process exports nothing either. The instrumentation
stops at the moment when you need it most. Nygard also states that every component can be up
while the end user gets a bad result. Blocked threads and cascading failure cause this
(Release It! Sec. 17.5, p. 287). A monitoring system reports the system's view of
itself. That view is not the user's view.

| The fault | Does white-box monitoring alone report it? |
|---|---|
| A DNS record resolves to the wrong address | No. The query never arrives |
| A load balancer stops forwarding traffic | No. The query never arrives |
| A server crashes before it records the request | No |
| A thread pool blocks and the process hangs | No. Export stops with the process |
| A certificate expires at the edge | No. The client fails before the handshake ends |
| A response carries HTTP 200 and wrong content | Only if you count implicit errors. Read `01-the-four-golden-signals.md` |

**Signal that you must add a black-box check.** A customer reported an outage before any
alert fired. Read gap code M-22 in `observability-gap-catalog.md`.

---

## 4. What black-box monitoring alone will miss

**Black-box monitoring reports an active problem only.** SRE calls black-box monitoring
fairly useless for a problem that is imminent and has not yet occurred
(SRE Ch. 6, p. 87). Zero redundancy is an imminent problem, and so is a nearly full part of
a service (SRE Ch. 6, p. 96, fn. 25).

**A retry hides the fault until the retry stops working.** White-box monitoring detects a
failure that a retry masks (SRE Ch. 6, p. 87). A probe sees a success. The failure count
inside the client tells the truth.

**A probe cannot name the failing component.** A probe reports that the system is not
working. It does not report which queue is full, or where the bottleneck sits. SRE states
that the transparent nature of white-box monitoring lets a team identify quickly which
components fail (SRE Ch. 10, p. 155). A probe also cannot separate a slow backend from a
slow network. Read Section 7.

---

## 5. The prober

**A prober runs a protocol check against a target and reports success or failure**
(SRE Ch. 10, p. 156). Google's prober can send an alert to the alert manager directly. It
can also export its own variables for a collector to scrape. SRE describes the prober as a
hybrid of the check-and-test model with richer variable extraction (SRE Ch. 10, p. 156).

**A prober must validate the payload, not only the status code.** SRE states that the
prober validates the contents of the response. The prober can also extract values and export
them as time series (SRE Ch. 10, p. 156). Teams export histograms of response times by
operation type and by payload size. Only an end-to-end test catches wrong content
(SRE Ch. 6, p. 88). A load balancer that counts HTTP 500 responses catches the requests that
failed completely. It does not catch a 200 response that carries the wrong body.

### The assertion ladder

| Level | What the check asserts | What it still misses |
|---|---|---|
| 1 | The name resolves and the port accepts a connection | Every application fault |
| 2 | The response carries the expected status code | Wrong content (SRE Ch. 6, p. 88) |
| 3 | The response payload matches an expected pattern (SRE Ch. 10, p. 156) | A slow response inside the timeout |
| 4 | The prober extracts values from the payload and exports them as time series (SRE Ch. 10, p. 156) | A fault in a feature that the probe does not exercise |

**Rule.** Write every new probe at level 3 or higher. A level 2 probe is gap code M-24.

**Rule.** Each probe exercises one user-visible operation. Nygard states that a system with
a Circuit Breaker can respond to requests while specific features do not work
(Release It! Sec. 13.2, p. 230). An aggregate up signal cannot represent that state.
Therefore write the probe per feature. `slo-engineering` owns the per-feature target itself.

---

## 6. Two vantage points, and the failing edge

**One probe tells you that something is wrong. It does not tell you where.** SRE runs the
prober against two targets. The first target is the load-balanced frontend name. The second
target is the web servers in each datacenter behind the load balancer
(SRE Ch. 10, p. 156).

**The comparison of the two results isolates the failing edge in the traffic flow graph**
(SRE Ch. 10, p. 156). The same comparison lets the team confirm that traffic still reaches
users when one datacenter fails, and suppress the alert.

| Frontend probe | Backend probe | What you know |
|---|---|---|
| Success | Success | The path is healthy at both points |
| Failure | Success | The fault sits between the user and the backend. Suspect DNS, the edge, or the load balancer |
| Failure | Failure | The fault sits in the backend or below it |
| Success | Failure in one datacenter | The load balancer removed that datacenter. Do not page. Raise a ticket |

**The frontend probe is symptom-oriented. It may page.** It reports that a user cannot use
the system now. That statement satisfies the pager rules in `02-symptom-based-alerting.md`.

**The backend probe is cause-oriented. It must not page on its own.** SRE tells you to
spend much more effort on catching symptoms than causes (SRE Ch. 6, p. 93). Keep the
backend probe as a debugging aid and as an alert suppression input. A page on a cause is
gap code M-14.

**Signal that you must add the second vantage point.** The team sees that the service
failed. The team cannot say which hop failed. Read gap code M-23.

---

## 7. Instrument both sides of every boundary

**You must know two numbers for every dependency.** You need to know how fast the web server
perceives the database to be. You also need to know how fast the database believes itself to
be (SRE Ch. 6, p. 87).

**Without both numbers you cannot separate a slow database from a slow network**
(SRE Ch. 6, p. 87). The client-side number carries the network, the connection pool, and the
server. The server-side number carries the server alone. The difference is the rest.

**Across a vendor boundary you own only the client side.** The vendor's white-box view is
not available to you. Your client-side instrumentation is the only evidence that you hold.
Nygard's integration point list names the fields to export. That list counts timeouts,
network errors, protocol errors, and application errors separately. It also carries the state
of the Circuit Breaker (Release It! Sec. 17.6, p. 298). An integration point with no telemetry
is gap code M-02.

---

## 8. Probe frequency

Source: SRE Ch. 6, p. 90. `slo-engineering` owns the availability table that supplies the
target. Read `slo-engineering/references/03-availability-math.md` for the downtime
arithmetic.

| Check | The book's rule |
|---|---|
| A 200-status probe for a service that targets 99.9% availability | More than once or twice a minute is probably unnecessarily frequent |
| Hard drive fullness for a service that targets 99.9% availability | More than once every one or two minutes is probably unnecessary |
| Every other interval | A local decision. State the target, then derive the interval |

**Derive the interval from the availability target, not from habit.** A target of 99.99%
allows about 13 minutes of downtime per quarter. The on-call reaction time must therefore be
on the order of minutes (SRE Ch. 11, p. 161). A detection delay that exceeds the allowed
downtime makes the target unreachable. Cost is the counterweight. Collection, storage, and
analysis all cost money at high resolution (SRE Ch. 6, p. 90–91).

---

## 9. Treat a collection failure as a signal

**A gap in a chart is not silence. It is an observation.** A Borgmon records four synthetic
variables for each target (SRE Ch. 10, p. 144).

1. Whether the name resolved to a host and a port.
2. Whether the target responded to a collection.
3. Whether the target responded to a health check.
4. What time the collection finished.

**These four variables let you write a rule that detects an unavailable task**
(SRE Ch. 10, p. 144). SRE states that Borgmon uses the collection failure itself as a signal
(SRE Ch. 10, p. 145). Section 12 gives the case that this rule answers.

**Signal that this section applies.** An alert rule reads the last data point and has no
test for a missing collection. Read gap code M-25.

---

## 10. The health check is not a probe

**A health endpoint that returns a fixed body reports only that the process runs.** Nygard
states that every component in an environment can be up while the end user gets a bad result.
Blocked threads and cascading failure cause this (Release It! Sec. 17.5, p. 287).

**Record the answer of the health check as its own signal** (SRE Ch. 10, p. 144). A rule can
then compare that answer against the user-visible result.

**Keep the policy outside the application.** Three decisions belong outside the application
itself. They are which metrics trigger alerts, where to set the thresholds, and how to combine
state variables into an overall health status. These are policy decisions, and they change at
a very different rate than the code (Release It! Sec. 17.2, p. 275). A health endpoint that
hard-codes the policy is gap code M-34.

**Modern.** A container platform gives a liveness probe and a readiness probe. A liveness
probe decides whether the platform restarts the container. A readiness probe decides whether
the load balancer sends traffic. Neither one is a user-facing black-box check. Treat both as
white-box signals with an outside reader, and add a separate probe against the user-facing
name. A health handler that ignores its dependencies is gap code M-26.

---

## 11. The observer must not share fate with the target

**A monitoring agent dies with its host.** Nygard states that monitoring systems always use
a heartbeat. The heartbeat detects a failed agent, or a network fault between the agent and
the collector (Release It! Sec. 17.5, p. 285).

**Keep monitoring traffic off the network segments that carry production traffic.** On a
shared segment, a worm, a denial of service attack, or a configuration error disables the
monitoring automatically (Release It! Sec. 17.5, p. 285).

**Run more than one global collector.** One global collector is a scaling bottleneck and a
single point of failure (SRE Ch. 10, p. 154). Google runs two or more global replicas across
production zones, so a maintenance event does not remove the observer. The launch checklist
names one more item directly. Monitor the monitoring (SRE App. E, p. 586). Read gap code
M-39.

---

## 12. War stories

**Prober at two vantage points (SRE Ch. 10, p. 156).** Google runs the prober against the
load-balanced name `www.google.com`. It also runs the prober against the web servers in each
datacenter behind the load balancer. With both targets the team can confirm that traffic still
reaches users when one datacenter fails. The team then suppresses the alert for that
datacenter. When the two results disagree, the team isolates the edge in the traffic flow
graph where the failure occurred. What this proves. Fault localization across a request path
needs probes at more than one point in that path. A single probe reports that something is
wrong and not where.

**Black Friday, all request handlers red (Release It! Ch. 16, p. 256–263).** An online
retailer ran an external monitor that shopped the site the way a customer does, from New York
and from San Francisco. The monitor turned red across all 75 request-handling instances. The
site lost orders at a rate near one million dollars an hour. The internal statistics looked
healthy. CPU use was low on the web, application, and database tiers. Average page latency
looked high but finite, because a request that timed out never entered the average. Thread
dumps then showed the real state. The 3,000 request-handling threads were blocked on a
connection pool with no timeout. The pool waited on an order management system. That system's
threads were blocked in turn on an external home-delivery scheduler. The scheduler accepted
about 25 concurrent requests and received about 90. What this proves. The outside check
detected the outage that every inside metric called healthy. The inside instrumentation then
named the component that the outside check could not name.

**Recovery declared by the outside check (Release It! Sec. 16.10, p. 263).** The team fixed
that incident with a runtime change, not a deploy. They set the scheduling connection pool
limits to zero on one instance. They recycled the component and verified that instance. They
then applied the change everywhere. The team declared recovery when the external monitor
reported success, about 90 seconds after the change. What this proves. The customer-perspective
probe is the verification signal for a remediation. An internal metric that moves is not proof
that the user is served.

**The pager that training had disabled (Release It! Sec. 16.8, p. 261).** During the same
incident the scheduling system paged its own on-call engineer several times about a CPU
condition at 100%. The engineer did not respond. That group was paged routinely for transient
CPU spikes that proved to be false alarms. The one alarm that mattered was indistinguishable
from the noise. What this proves. A threshold set loosely against reality destroys the
detection capability that the alarm was bought for. Nygard states that expectations must match
reality, so that you avoid the negative effects of false positives
(Release It! Sec. 17.7, p. 303–304).

**Scraping over HTTP against the SNMP objection (SRE Ch. 10, p. 145).** SNMP is designed
with minimal transport requirements. It continues to work when most other network applications
fail. Collecting variables over HTTP appears to contradict that design. Google reports that
this is rarely an issue. The system is already built to survive network and machine faults.
Borgmon also turns the failed collection into an alerting signal.
What this proves. You do not need a separate transport for the observer if you treat a
failed collection as a first-class observation.

---

## 13. The procedure

Run these steps when you add a probe, a health check, or a synthetic transaction.

1. Name the user-visible operation that the probe exercises. One operation per probe.
2. Write the probe at assertion level 3 or higher. Validate the payload, not the status code.
3. Run the probe against the user-facing name. Route its alert as a page.
4. Run the same probe against the servers behind the load balancer. Route its result to a
   ticket or to a dashboard.
5. Derive the interval from the availability target. State the target in the record.
6. Export the four synthetic variables for every collection target (SRE Ch. 10, p. 144).
7. Instrument both sides of every boundary that the probe crosses.
8. Place the observer outside the fate of the target. Add a heartbeat.
9. Record the probe, the interval, the vantage points, and the output in the Observability
   Record. Read `../SKILL.md`, Section 12.

**The pager rules decide the output, not the vantage point.** A page that satisfies the four
pager rules is a good page. The vantage point that triggered it is irrelevant
(SRE Ch. 6, p. 93).

---

## 14. Modern practice

**Modern.** A hosted synthetic monitoring service performs the black-box role from outside
your network. It gives a vantage point that your own infrastructure cannot give. The book's
rules apply to it without change. Validate the payload. Use more than one vantage point.
Derive the interval from the availability target.

**Modern.** A distributed tracing library gives the two numbers of Section 7 for one
request. A trace is white-box instrumentation with a shared identifier. A service mesh
sidecar reports the same two numbers without application code. Neither one replaces a probe,
because both record only the requests that arrived.

---

## 15. Gap codes that this file governs

| Code | Short name |
|---|---|
| M-02 | An integration point with no telemetry |
| M-22 | White-box monitoring only |
| M-23 | A probe at one point in the path |
| M-24 | A probe that checks the status code only |
| M-25 | A collection gap treated as silence |
| M-26 | A health endpoint that ignores its dependencies |
| M-39 | The monitor shares fate with the target |

Read the full entries in `observability-gap-catalog.md`.

---

## 16. Where to read next

| Question | File |
|---|---|
| Which metric do I expose, and how do I measure it? | `01-the-four-golden-signals.md` |
| Does this condition page, raise a ticket, or only get logged? | `02-symptom-based-alerting.md` |
| How do I build the collection, the labels, and the rules? | `04-time-series-and-rules.md` |
| What do I log, and what must an operator answer alone? | `05-transparency-and-logging.md` |
| Where do the expectation and the baseline live? | `06-the-operations-database.md` |
| Which gaps apply to this change? | `observability-gap-catalog.md` |
