# The Four Golden Signals

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 6 "Monitoring Distributed Systems", p. 87–96. The four definitions are on p. 88–89. The
tail section is on p. 89–90. The resolution section is on p. 90–91. Supporting material comes
from SRE Ch. 10, p. 149 and p. 159, and from *Release It!* — Michael T. Nygard (Pragmatic
Bookshelf, 2007), Sec. 16.6, p. 258–259, and Sec. 17.6, p. 297–298.

The four golden signals are latency, traffic, errors and saturation. If you can measure only
four metrics of a user-facing system, measure these four (SRE Ch. 6, p. 88).

**Page a human when one signal is problematic. For saturation, page when the signal is nearly
problematic** (SRE Ch. 6, p. 89).

Keep the book's hedge. Google claims only decent coverage for a service measured this way
(SRE Ch. 6, p. 89). The four signals do not make a monitoring system complete. Each signal
also has a wrong measurement that is easy to write and hard to detect. This file names the
wrong measurement for each one.

---

## 1. The four signals in one table

Source: SRE Ch. 6, p. 88–89.

| Signal | What it measures | The wrong measurement | The right measurement |
|---|---|---|---|
| **Latency** | The time to service a request | One mean over all requests | Two series. Success latency and failure latency (p. 88) |
| **Traffic** | The demand on the system, in a metric that fits this system | One request count with no type | The metric that fits the system. See Section 3 (p. 88) |
| **Errors** | The rate of requests that fail | Only the explicit codes | Three classes. Explicit, implicit and by policy (p. 88) |
| **Saturation** | How full the most constrained resource is | 100% utilization as the target | A utilization target below 100%, plus a prediction (p. 89) |

**The four signals cover a user-facing system. They do not cover a subsystem by
themselves.** Saturation and performance of a subsystem such as a database often need direct
measurement on that subsystem (SRE Ch. 6, p. 95–96).

**Every signal in this file is a symptom, not a cause.** A symptom rule pages a human. A
cause rule helps a human debug. Read `02-symptom-based-alerting.md` before you route any of
these signals to a pager.

---

## 2. Latency

Latency is the time to service a request (SRE Ch. 6, p. 88).

### 2.1 Split the series in two

**Record success latency and failure latency as two separate series** (SRE Ch. 6, p. 88).

An HTTP 500 error that follows the loss of a database connection can return very quickly. A
500 error marks a failed request. If you count 500s in the overall latency, the calculation
misleads you (SRE Ch. 6, p. 88).

**Do not filter errors out of the latency calculation. Track error latency instead**
(SRE Ch. 6, p. 88). A slow error is worse than a fast error (SRE Ch. 6, p. 88).

The two mistakes are opposite, and both are common:

| The mistake | What the chart shows | What is true |
|---|---|---|
| Failed requests counted in one latency series | Latency rises when a backend fails fast | The service returns errors, and it returns them quickly |
| Failed requests removed from every latency series | Latency looks normal during a total outage | No request completes at all |

### 2.2 The signature in code

**Signal that the code is wrong:** the latency timer stops only inside the success branch. A
block records the duration only after a 200 response. Catalog code M-08 in
`observability-gap-catalog.md` names this defect.

**Fix:** stop the timer on every exit path. Then label the observation with the outcome.
Count aborted requests and timeouts as their own counters (Release It! Sec. 17.1, p. 271, and
Sec. 17.6, p. 298).

### 2.3 War story — the latency that read as a mild slowdown

Nygard was called into a retail site outage on Black Friday. Twenty minutes into the
incident, the team knew three things. Session counts were very high. Page latency was high.
CPU use on the web, application and database tiers was very low. Almost every
request-handling thread was busy, and many threads had worked on one request for more than
five seconds. The page latency was not merely high. Requests were timing out, so the real
latency was effectively infinite. The statistics reported the average of the requests that
completed. Requests that did not complete never entered the average. The summary metric
understated a total outage (Release It! Sec. 16.6, p. 258–259).

---

## 3. Traffic

Traffic measures the demand on the system. Measure it in a high-level metric that fits this
system (SRE Ch. 6, p. 88).

| The system | The traffic metric the book names |
|---|---|
| A web service | HTTP requests per second, divided by the nature of the request, such as static content against dynamic content |
| An audio streaming system | Network I/O rate, or concurrent sessions |
| A key-value storage system | Transactions per second and retrievals per second |

**Choose the metric before you write the counter.** A single number named `requests` with no
type cannot answer a question during an incident. It cannot tell you that dynamic traffic
doubled while static traffic fell.

**Traffic is also the denominator of the error ratio.** An alert rule on errors needs both a
ratio and a volume floor. Read `04-time-series-and-rules.md`, which carries the worked rule
from SRE Ch. 10, p. 153.

---

## 4. Errors

Errors are the rate of requests that fail (SRE Ch. 6, p. 88). The book names three classes.

| Class | Definition | Where the book says you catch it |
|---|---|---|
| **Explicit** | A failure the protocol states, such as an HTTP 500 | An HTTP 500 count at the load balancer does a decent job on completely failed requests (p. 88) |
| **Implicit** | An HTTP 200 response that carries the wrong content | Only an end-to-end test detects wrong content (p. 88) |
| **By policy** | A request that breaks a promise you made, such as one second | Your own policy threshold, applied to the latency distribution (p. 88) |

**A metric that counts only HTTP 500s counts one class of three.** Catalog code M-09 names
this gap.

**Where protocol response codes cannot express a failure, add a secondary internal protocol
to track the partial failure modes** (SRE Ch. 6, p. 88).

**The policy class needs a written promise.** The book gives one example. If you committed to
one-second response times, a request over one second is an error (SRE Ch. 6, p. 88). The
number itself is a local decision. Take it from the SLO for the feature, not from this file.

---

## 5. Saturation

Saturation measures how full the service is. It emphasizes the most constrained resource
(SRE Ch. 6, p. 89). In a memory-constrained system, show memory. In an I/O-constrained
system, show I/O.

### 5.1 A utilization target is essential

**Many systems degrade in performance before they reach 100% utilization, so a utilization
target is essential** (SRE Ch. 6, p. 89).

100% is not the target. The target is a number below 100%. The book does not give that
number. `capacity-engineering` owns the load test that produces it. This file owns only the
series that reports the current value against it.

### 5.2 Saturation is the signal you page on early

Saturation is the only one of the four signals that pages before the symptom exists. The
other three describe a failure that users can already see. Saturation describes a failure
that users will see (SRE Ch. 6, p. 89).

Three mechanisms give the early view:

1. **A higher-level load measurement.** Ask whether the service can handle double the
   traffic, 10% more traffic, or less traffic than it receives now (SRE Ch. 6, p. 89).
2. **The 99th percentile response time over a small window.** The book names one minute. That
   measurement can give a very early signal of saturation (SRE Ch. 6, p. 89). Latency
   increases are often a leading indicator of saturation (SRE Ch. 6, p. 89).
3. **A prediction of impending saturation.** The book's example is a statement that the
   database will fill its hard drive in four hours (SRE Ch. 6, p. 89).

**Signal to page on saturation:** the resource is nearly problematic, not yet problematic
(SRE Ch. 6, p. 89). A zero-redundancy state counts as imminent, and so does a nearly full
part of the service (SRE Ch. 6, p. 96, footnote 25).

### 5.3 Which measurement to use

| The service | The saturation measurement | Source |
|---|---|---|
| A simple service with no parameter that alters request complexity, and a configuration that rarely changes | A static value from a load test can be adequate | SRE Ch. 6, p. 89 |
| Most services | An indirect signal with a known upper bound, such as CPU utilization or network bandwidth | SRE Ch. 6, p. 89 |
| A complex service | The indirect signal, plus the higher-level load measurement | SRE Ch. 6, p. 89 |

**A mean saturation figure hides an imbalance.** CPUs and databases can carry a very
imbalanced load (SRE Ch. 6, p. 89). Read Section 6.

---

## 6. Distributions, not averages

**Do not design a monitoring system on the mean of a quantity** (SRE Ch. 6, p. 89). The book
names three examples of the mistake. Mean latency, mean CPU use of the nodes, and mean
fullness of the databases.

### 6.1 The arithmetic

A web service runs at an average latency of 100 ms and 1,000 requests per second. 1% of
requests might easily take 5 seconds (SRE Ch. 6, p. 89–90).

A user page can depend on several such services. The 99th percentile of one backend then
easily becomes the median response of your frontend (SRE Ch. 6, p. 90).

The book states the inverse in a footnote. If 1% of requests are 50 times the average, the
rest of the requests are about twice as fast as the average. Without a measured distribution,
the belief that most requests sit near the mean is hopeful thinking (SRE Ch. 6, p. 96,
footnote 23).

### 6.2 The bucket method

**Collect request counts bucketed by latency, rather than actual latencies** (SRE Ch. 6,
p. 90). The result renders as a histogram.

**Distribute the bucket boundaries approximately exponentially, in this case by factors of
roughly 3** (SRE Ch. 6, p. 90). The book gives these boundaries. 0 ms to 10 ms. 10 ms to
30 ms. 30 ms to 100 ms. 100 ms to 300 ms. The pattern continues from there.

Catalog code M-07 names the mean-based series.

### 6.3 War story — Bigtable and the mean-based SLO

The Bigtable service once measured its SLO on the mean performance of a synthetic
well-behaved client. Problems in Bigtable and in lower storage layers drove that mean with a
large tail. The worst 5% of requests were often much slower than the rest. Email
alerts fired as the SLO approached, and pages fired when the SLO was exceeded. Both fired
voluminously. The team spent large amounts of time on triage to find the few actionable
alerts, and it often missed the problems that affected users. The remedy had three parts. The
team improved Bigtable performance. It temporarily dialed back the SLO target to the 75th
percentile request latency. It disabled email alerts entirely (SRE Ch. 6, p. 93–94). The case
proves two things. A mean-based target produces noise that hides real user impact. A
deliberate relaxation of the target is a legitimate move.

### 6.4 Do not average a percentile — **Modern**

A query of the form `avg(p99_latency)` returns a percentile of nothing. Store latency buckets
per instance. Aggregate the buckets. Then compute the quantile from the aggregate. The bucket
method comes from SRE Ch. 6, p. 90. The averaging defect is a property of current query
languages and does not come from either book. Catalog code M-12 names it. The design rule for
percentiles belongs to `data-systems-design`, Section 2.1.

---

## 7. Choose an appropriate resolution

**Measure different aspects of a system with different granularity** (SRE Ch. 6, p. 90). One
collection interval for every metric is wrong in both directions. It hides a spike, and it
costs too much.

| The measurement | The book's guidance | Source |
|---|---|---|
| CPU load | A one-minute span does not reveal even quite long-lived spikes that drive high tail latencies | SRE Ch. 6, p. 90 |
| A probe for a 200 status, at a target of no more than 9 hours downtime per year (99.9% annual uptime) | More than once or twice a minute is probably unnecessarily frequent | SRE Ch. 6, p. 90 |
| Hard drive fullness, at a 99.9% availability target | More than once every 1 to 2 minutes is probably unnecessary | SRE Ch. 6, p. 90 |

Keep the word "probably". The book hedges both external intervals. Derive your own interval
from your availability target, and record the derivation.

### 7.1 The sampling procedure

**Signal to use this procedure:** the monitoring goal needs high resolution, and it does not
need extremely low latency (SRE Ch. 6, p. 90).

Sample inside the server. Aggregate outside it.

1. Record the current CPU utilization each second.
2. Increment the appropriate CPU utilization bucket each second, using buckets of 5%
   granularity.
3. Aggregate those values every minute.

This procedure lets you observe brief CPU hotspots without a very high cost in collection and
retention (SRE Ch. 6, p. 90–91).

### 7.2 The cost of over-collection

**Per-second collection of every metric might yield interesting data. Such frequent
measurements may be very expensive to collect, to store and to analyze** (SRE Ch. 6, p. 90).

Catalog code M-11 names both faults. A resolution that hides the spike, and a resolution that
costs too much.

Two removal rules limit the cost over time (SRE Ch. 6, p. 91):

- Remove collection, aggregation and alerting configuration that is rarely exercised. The
  book gives less than once a quarter as one team's line.
- Remove a signal that no prebaked dashboard exposes and no alert uses.

**The rules that catch real incidents most often should be as simple, predictable and
reliable as possible** (SRE Ch. 6, p. 91).

---

## 8. Counter or gauge

Source: SRE Ch. 10, p. 149.

A counter only increases. A gauge takes any value.

| The quantity | Type | Reason |
|---|---|---|
| Total requests served | Counter | A counter keeps its meaning when events fall between two samples |
| Bytes written | Counter | The same reason |
| Errors by response code | Counter, with a code label | The same reason. The code label is the diagnosis axis |
| Current queue depth | Gauge | It is a state, not a total |
| Threads in use now | Gauge | The same reason |
| Connections checked out now | Gauge | The same reason |

**Prefer a counter.** A gauge collection is likely to miss activity between sampling
intervals (SRE Ch. 10, p. 149). Catalog code M-10 names the reverse choice.

**Compute the sum of rates, not the rate of sums.** That order defends the result against a
counter reset. It also defends the result against a missing collection after a task restart
(SRE Ch. 10, p. 159, footnote 52). The `rate` function handles the counter reset itself
(SRE Ch. 10, p. 159, footnote 53).

---

## 9. The record for one surface

Fill this table for every user-facing surface. `../SKILL.md`, Section 12 carries the full
Observability Record.

| Signal | Series name | Type | Exposed where | Buckets or labels |
|---|---|---|---|---|
| Latency, successful requests | | histogram | | |
| Latency, failed requests | | histogram | | |
| Traffic | | counter | | |
| Errors, explicit | | counter by code | | |
| Errors, implicit | | counter | | |
| Errors, by policy | | counter | | |
| Saturation | | gauge | | |
| Saturation prediction | | derived series | | |

---

## 10. Review questions for any new metric

Ask these six questions. Each one maps to a catalog code in
`observability-gap-catalog.md`.

1. Does a mean appear in the series name, in the alert rule, or on the dashboard panel?
   (M-07)
2. Does the latency timer stop on the failure path as well as the success path? (M-08)
3. Does the error metric count the implicit class and the policy class? (M-09)
4. Is a total reported as a gauge? (M-10)
5. Does a stated reason exist for the collection interval? (M-11)
6. Does any query average a percentile across instances? (M-12)

**This file does not apply to a change that adds no call, no state and no new failure mode.**
"Not applicable" is a valid result.

---

## 11. Related files

| File | Read it when |
|---|---|
| `02-symptom-based-alerting.md` | You route one of these four signals to a pager |
| `03-white-box-and-black-box.md` | You add a probe, a health check, or an end-to-end test for the implicit error class |
| `04-time-series-and-rules.md` | You write the ratio, the volume floor, the window and the duration for an alert rule |
| `05-transparency-and-logging.md` | You expose the counters that a component must publish |
| `06-the-operations-database.md` | You set the utilization target, the nominal range, or the review cadence |
| `observability-gap-catalog.md` | You review a change, or you review an incident that the monitoring did not detect |
| `data-systems-design/references/01-reliability-scalability-maintainability.md` | You need the design rule for percentiles and tail latency amplification |
| `capacity-engineering/references/01-defining-capacity.md` | You need the utilization target itself, rather than the series that reports it |
| `slo-engineering/references/02-slis-slos-and-slas.md` | You need the objective that the policy error class is measured against |
