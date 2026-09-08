# Observability Gap Catalog

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 6, Ch. 7, Ch. 10, Ch. 11, App. B, App. D, App. E, App. F. *Release It!* — Michael T. Nygard
(Pragmatic Bookshelf, 2007). Sec. 12.2, Sec. 13.2, Sec. 14.2 to Sec. 14.4, Ch. 15, Ch. 16, Ch. 17.

A catalog of named observability gaps. A gap is a defect that you can search for and name. Each entry
has three parts:

- **Signature** — what the code, the configuration, the dashboard, or the process looks like. This is
  the text that you search for.
- **Consequence** — what goes wrong, and when.
- **Fix** — the required remedy.

**How to use it.** `code-review` runs the applicable groups against a difference. `verify` runs them
against an implemented phase. `planner` and `detail-planning` run them against a design. Report only
the codes that apply. "Not applicable" is a valid result, and it is better than an invented finding.

**Severity.** 🔴 causes data loss, corruption, or an unrecoverable state. It blocks the merge.
🟡 causes an outage or a wrong result under load. 🔵 is a risk to operation or to maintenance.

**Citation rule.** Every claim cites a chapter and a page, or carries the tag **Modern**. A **Modern**
entry names a defect that current practice knows and that neither book states.

---

## A. Signals that do not exist

### 🔵 M-01 — No exported variable interface
**Signature:** no statistics handler. A search finds no metrics registry and no `/varz` route.
**Modern:** a search also finds no `/metrics` route, which is the current convention.
**Consequence:** the operator cannot see which component fails, which queue is full, or where the
bottleneck is (SRE Ch. 10, p. 155).
**Fix:** add one handler that emits internal statistics. One fetch then collects every exported
variable in plain text (SRE Ch. 6, p. 82, and Ch. 10, p. 143). Expose every state variable, counter,
and metric, because the key metrics change over time (Release It! Sec. 17.6, p. 297).

### 🟡 M-02 — An integration point with no telemetry
**Signature:** a client class calls a vendor API. No counter carries a vendor label.
**Consequence:** you cannot separate a slow vendor from a slow network. You need the caller-side view
and the vendor-side view together to place the fault (SRE Ch. 6, p. 87).
**Fix:** expose Nygard's integration-point list of eleven items for every vendor. It counts network
errors, protocol errors, and application errors separately. It also carries the actual IP address of
the remote endpoint (Release It! Sec. 17.6, p. 298).

### 🟡 M-03 — A resource pool with no exposed counters
**Signature:** code creates a connection pool or a thread pool. No metric reports the checked-out
count or the count of blocked threads.
**Consequence:** thread exhaustion looks like health. In Nygard's outage the latency was high, the
threads were almost all busy, and the CPU on all three tiers was low (Release It! Sec. 16.6, p. 258).
**Fix:** expose the resource pool list, including the threads that block and wait for a resource
(Release It! Sec. 17.6, p. 297–298). Expose the threads busy more than five seconds and the request
backlog (Release It! Sec. 17.1, p. 270).

### 🔴 M-04 — A required event with no absence alarm
**Signature:** a daily feed, an extract, or a batch job. The dashboard has a failure state and no
entry for "did not run".
**Consequence:** the job stops and nobody sees the stop. Nygard traces a startling number of
business-level problems to one cause. Batch jobs failed invisibly for 33 days straight
(Release It! Sec. 17.1, p. 272).
**Fix:** record the expected event and alert on its absence, with an `ExpectedTime` expectation. Also
record the item count at start, the count processed at end, and any abnormal termination
(Release It! Sec. 17.7, p. 302–304). Red means a required event did not occur (Fig. 17.1, p. 273).

### 🔴 M-05 — No coarse-grain data check after a change
**Signature:** a deploy or a data push. No comparison of the new output volume to the previous volume.
**Consequence:** half of a dataset can vanish after a push, and no signal reports the loss
(SRE Ch. 7, p. 101). A file that shrinks to one forward slash matches everything. Google marked the
entire web as malware this way in 2009 (SRE App. B, p. 572).
**Fix:** alert on coarse-grain differences after every push (SRE Ch. 7, p. 101). Alert when the new
configuration is N% smaller than the previous version. Check for empty and truncated data. Serve the
previous state until a person approves the new data (SRE App. B, p. 571–572).

### 🔵 M-06 — No record of recent changes beside the metrics
**Signature:** the dashboard shows charts only. No deploy marker. No list of configuration changes.
**Consequence:** a person asks what else happened when the latency increased. The dashboard has no
answer (SRE Ch. 6, p. 84).
**Fix:** show recent pushes on the dashboard. A push is any change to running software **or its
configuration** (SRE Ch. 6, p. 83). Pair the dashboard with a log, so a person can analyze historical
correlations (SRE Ch. 6, p. 96). Audit version control against the live configuration
(Release It! Sec. 14.2, p. 245).

### 🟡 M-41 — A start-up failure that the service does not announce
**Signature:** a start-up path that logs nothing on failure. A process that exits when one component
fails to start.
**Consequence:** the service fails during a night reboot and nobody knows, because the application
does not announce the failure. A halted application holds no state that a person can interrogate
(Release It! Sec. 14.3, p. 247).
**Fix:** start the components in a defined order and write each failure to the log. Enter a failure
state and stay running. Refuse connections until start-up completes (Release It! Sec. 14.3, p. 247).

### 🟡 M-42 — Per-node visibility with no aggregate
**Signature:** a dashboard with one panel for each host. A cache statistic or a queue depth that a
person can read for one node only.
**Consequence:** a scaling effect stays invisible. Each Nygard application server removed items from
the cache of every other server, and each new server would have made the fault worse
(Release It! Sec. 17.2, p. 275).
**Fix:** aggregate the same statistic across all nodes on one page. The fault was obvious as soon as
all the cache statistics appeared together (Release It! Sec. 17.2, p. 275).

---

## B. Measurements that lie

### 🟡 M-07 — The mean hides the tail
**Signature:** a series named `avg_latency` or `mean_response_time`. An alert that compares an average
to a threshold. A panel labelled "average".
**Consequence:** a service with an average latency of 100 ms at 1,000 requests per second can easily
serve 1% of requests in 5 seconds. The 99th percentile of one backend can easily become the median
response of your frontend (SRE Ch. 6, p. 89–90).
**Fix:** collect request counts in buckets by latency. Distribute the bucket boundaries approximately
exponentially, by factors of roughly 3 (SRE Ch. 6, p. 90).

### 🟡 M-08 — Failed requests removed from the latency series
**Signature:** the latency timer stops on the success path only. A block that records the duration
after a 200 response only.
**Consequence:** a timed-out request never enters the average, so a total outage reads as a mild
slowdown (Release It! Sec. 16.6, p. 258–259). A slow error is worse than a fast error
(SRE Ch. 6, p. 88).
**Fix:** record success latency and failure latency as two series (SRE Ch. 6, p. 88). Count aborted
requests (Release It! Sec. 17.1, p. 271) and timeouts (Release It! Sec. 17.6, p. 298) as counters.

### 🟡 M-09 — Only explicit errors are counted
**Signature:** the error metric counts HTTP 500 responses only. No policy threshold exists. No payload
check exists.
**Consequence:** a 200 response with the wrong content is not counted. A response that breaks a
one-second promise is not counted (SRE Ch. 6, p. 88).
**Fix:** count three classes of error: explicit, implicit, and by policy. A load balancer catches the
explicit class. Only an end-to-end test catches wrong content. Where response codes cannot express
the failure, add a secondary internal protocol (SRE Ch. 6, p. 88).

### 🔵 M-10 — A gauge where a counter belongs
**Signature:** a metric that reports events since the last sample as a value that can decrease. A
metric that the exporter resets on each scrape.
**Consequence:** activity between two sampling intervals is lost (SRE Ch. 10, p. 149).
**Fix:** use a counter for a total and a gauge for a current state (SRE Ch. 10, p. 149). Compute the
sum of rates, not the rate of sums. That order defends the result against a counter reset and against
a missed collection (SRE Ch. 10, p. 159, fn. 52).

### 🔵 M-11 — Resolution that hides the spike
**Signature:** a CPU sample once a minute. A collection interval with no stated reason. A per-second
collection of every metric in the service.
**Consequence:** a CPU observation over the span of a minute does not reveal the long-lived spikes
that drive high tail latency. A per-second collection of everything is very expensive
(SRE Ch. 6, p. 90).
**Fix:** sample inside the server and aggregate outside it. Record the CPU each second, increment a
bucket of 5% granularity, and aggregate every minute (SRE Ch. 6, p. 90–91).

### 🟡 M-12 — A percentile averaged across instances — **Modern**
**Signature:** a query of the form `avg(p99_latency)`. A per-instance quantile that the system stores
and then combines by a mean.
**Consequence:** the result is not a percentile of anything. The tail moves and the chart does not.
**Fix:** store latency buckets for each instance, aggregate the buckets, and compute the quantile from
the aggregate. The bucket method comes from SRE Ch. 6, p. 90. The averaging defect is a property of
current query languages. `data-systems-design`, Sec. 2.1 owns the design rule for percentiles.

### 🔵 M-13 — Label cardinality with no bound — **Modern**
**Signature:** a metric label that carries a user id, a request id, an email address, a session id, or
a raw URL path.
**Consequence:** the series count grows with traffic. The store fills. Queries slow down or fail.
**Fix:** use three kinds of label only. Use a breakdown of the data, such as a response code. Use the
source, such as instance and job. Use the locality, such as zone and shard (SRE Ch. 10, p. 157). Keep
identifiers in logs and in traces, never in metric labels.

---

## C. Alerts that must not page

### 🟡 M-14 — A page on a cause, not a symptom
**Signature:** an alert named after a component state, such as `DatabaseCpuHigh`. No SLO appears in
the rule.
**Consequence:** the page fires when no user is affected, and the real user-visible failure has no
rule. It is better to spend much more effort on catching symptoms than causes (SRE Ch. 6, p. 93).
**Fix:** page on symptoms and keep the cause rules as debugging aids (SRE Ch. 6, p. 93). Align every
paging alert with a symptom that threatens the SLO (SRE Ch. 11, p. 166). Page on a cause only when it
is very definite and very imminent, such as zero redundancy (SRE Ch. 6, p. 96, fn. 25).

### 🟡 M-15 — A page with a rote response
**Signature:** a runbook step for the alert that is one fixed command. The same alert name in the
on-call log each week with the same action.
**Consequence:** the page consumes a person and returns no judgment. Pages with rote, algorithmic
responses are a red flag (SRE Ch. 6, p. 95).
**Fix:** every page response must require intelligence. A page that merits a robotic response should
not be a page (SRE Ch. 6, p. 92). Find the root cause and remove it. Where removal is not possible,
the response deserves full automation (SRE Ch. 6, p. 93). `self-healing-apis` owns that work.

### 🟡 M-16 — An alert rule with one comparison and no duration
**Signature:** a rule with a single threshold, such as `error_ratio > 0.01`. No `for` clause and no
pending state.
**Consequence:** low traffic makes a ratio jump, so the rule pages on two failures out of ten
requests. Without a duration the alert flaps, and one missed collection sends a false page
(SRE Ch. 10, p. 153).
**Fix:** pair the ratio with an absolute rate and add a duration. The book's rule fires only when the
ratio is above 0.01 **and** the error rate is above 1 per second, held for 2 minutes. The duration is
at least two rule evaluation cycles (SRE Ch. 10, p. 153).

### 🟡 M-17 — Alerts routed to email
**Signature:** an alert destination that is an email alias or a mailing list. A filter rule in a
mailbox that files the alerts.
**Consequence:** nobody reads them. Google nicknames email alerts "alert spam", because people rarely
read them and rarely act on them (SRE Ch. 6, p. 96, fn. 22). Email alerting makes the inevitable
outage more severe (SRE App. B, p. 574).
**Fix:** allow three outputs only. They are a page, a ticket, and a log (SRE App. B, p. 573–574). Move
subcritical conditions to a dashboard (SRE Ch. 6, p. 96). A condition worth disturbing a person must
either page or become a bug in the bug tracker (SRE App. B, p. 574).

### 🟡 M-18 — One incident that produces many pages
**Signature:** an incident timeline that lists five or more distinct alert names with one cause. A
pager storm during a global outage (SRE App. D, p. 582).
**Consequence:** the on-call engineer triages duplicates instead of the incident (SRE Ch. 11, p. 167).
**Fix:** group related alerts. Inhibit a downstream alert while the upstream alert is active.
Deduplicate the alerts that carry the same labelset (SRE Ch. 10, p. 154). Silence duplicate alerts
during an incident, and tune toward one alert for each incident (SRE Ch. 11, p. 167).

### 🟡 M-19 — A threshold set above reality
**Signature:** an alert that fires during normal operation. A team that describes a nightly page as
normal. A page count in the production meeting that no bug tracks (SRE App. F, p. 589).
**Consequence:** false positives train people to ignore the alarm. Nygard reports an on-call engineer
who did not answer a page for 100% CPU, because that group is paged routinely for false alarms
(Release It! Sec. 16.8, p. 261).
**Fix:** derive the expectation from the historical data that the system already records. Tighten it
over time (Release It! Sec. 17.7, p. 303–304). Record every threshold change with a bug and an owner.
The production meeting does that when it raises a delay threshold (SRE App. F, p. 589).

### 🔵 M-20 — An alert nobody can act on
**Signature:** an alert whose runbook states no action. An alert with no named owner. An alert created
because something seemed a bit weird.
**Consequence:** the on-call engineer cannot respond. All paging alerts must be actionable
(SRE Ch. 11, p. 166). You should never trigger an alert simply because something seems a bit weird
(SRE Ch. 6, p. 84).
**Fix:** answer the five alert-review questions before the rule ships (SRE Ch. 6, p. 92). Delete the
rule when question 4 has no answer, or when question 5 shows that another team is already paged.

### 🔵 M-21 — A single static threshold as the only SLO alert — **Modern**
**Signature:** one rule that compares the current error ratio to the SLO target. No pair of windows.
**Consequence:** a slow burn spends the whole period's budget and never crosses the instantaneous
threshold. A short spike pages when the budget can absorb it.
**Fix:** alert on the rate at which the service consumes the error budget, over a long window and a
short window together. The book defines the error budget as 1 minus the SLO over a period, usually a
month (SRE App. B, p. 573). Alerting on a burn rate over two windows is later practice.
`slo-engineering` owns the budget policy. This code covers only the shape of the rule.



---

## D. The outside view

### 🟡 M-22 — White-box monitoring only
**Signature:** every alert reads a series that the service itself exports. No prober exists. No
synthetic transaction exists.
**Consequence:** you see only the queries that arrive. A query lost to a DNS error is invisible, and
you can only alert on the failures that you expected (SRE Ch. 10, p. 156). A dead process logs no
tales (Release It! Sec. 17.4, p. 283).
**Fix:** add a black-box probe against the user-facing name. Combine heavy use of white-box monitoring
with modest but critical use of black-box monitoring (SRE Ch. 6, p. 87). Define the SLO the way a user
sees it, and measure it client-side (SRE App. B, p. 572).

### 🟡 M-23 — A probe at one point in the path
**Signature:** one synthetic check against the public domain name. No check behind the load balancer.
**Consequence:** the probe reports a failure and does not report where the failure is
(SRE Ch. 10, p. 156).
**Fix:** run the prober against the load-balanced name **and** against the servers in each datacenter
behind the load balancer. Compare the two views. Confirm that traffic still reaches the users, and
suppress the alert. If it does not, isolate the edge in the traffic flow graph where the failure
occurred (SRE Ch. 10, p. 156).

### 🟡 M-24 — A probe that checks the status code only
**Signature:** a check that asserts HTTP 200 and reads no part of the body. A monitor definition with
no success pattern and no failure pattern.
**Consequence:** a 200 response with the wrong content passes (SRE Ch. 6, p. 88). A feature that
answers in 50 ms and returns an error to every user counts as available
(Release It! Sec. 13.2, p. 231).
**Fix:** validate the response payload and export values from it as time series (SRE Ch. 10, p. 156).
Define the success and failure patterns, the response time limit for each step, the frequency, and
the locations. Give the synthetic transaction a designated user id (Release It! Sec. 13.2, p. 231).

### 🔵 M-25 — A collection gap treated as silence
**Signature:** a rule that reads the last data point and has no test for a missing collection. No
per-target variable that records whether the target answered.
**Consequence:** a target that stops answering produces a gap in the chart, not an alarm.
**Fix:** record four synthetic variables for each target. Record whether the name resolved, whether
the target answered a collection, whether it answered a health check, and what time the collection
finished. Then use the collection failure itself as a signal (SRE Ch. 10, p. 144–145). For delayed
data, alert well before the data is expected to expire (SRE App. B, p. 571).

### 🟡 M-26 — A health endpoint that ignores its dependencies — **Modern**
**Signature:** a handler that returns 200 and a fixed body. A search inside the handler finds no
dependency check and no state test.
**Consequence:** every component reports itself as up while the user gets a bad result. A monitoring
system reports the system's view of itself, not the user's view (Release It! Sec. 17.5, p. 287).
**Fix:** record the health-check answer as its own signal (SRE Ch. 10, p. 144). Keep the user-facing
probe as the ground truth (SRE App. B, p. 572). A liveness answer separated from a readiness answer
is current platform practice.

### 🟡 M-43 — An availability target with no monitoring device behind it
**Signature:** a requirement that says the system shall be available 99.9% of the time. No named
monitoring device. No stated success pattern. No stated frequency.
**Consequence:** nobody can measure the target. The argument then starts a year later, after the
incident. A person measures availability by a mouse click (Release It! Sec. 13.2, p. 230–231).
**Fix:** name the device that runs the synthetic transaction. State the response time limit for each
step, the patterns that mean success and failure, the frequency, and the locations
(Release It! Sec. 13.2, p. 231–232). `slo-engineering` owns the target itself and the dependency
inheritance rule. Read `slo-engineering/references/05-gathering-availability-requirements.md`.

---

## E. Logs as an operator interface

### 🟡 M-27 — A debug or trace level shipped to production
**Signature:** a logging configuration file inside the release artifact with a level of DEBUG or
TRACE.
**Consequence:** real problems are buried in method traces. One debug message left enabled drove
weekly production database failovers for six months (Release It! Sec. 17.4, p. 278, p. 281–283).
**Fix:** add a build step that automatically removes any configuration that enables a debug or a trace
log level (Release It! Sec. 17.4, p. 278). Make the log file location configurable, so an
administrator can place the logs on a separate device (Release It! Sec. 17.4, p. 277).

### 🟡 M-28 — A remediation with no proven cause in the runbook
**Signature:** a runbook entry whose evidence is that the action appeared to work once. A remediation
whose link to the symptom is only a time order.
**Consequence:** superstition hardens into procedure. In Nygard's case there was no causal connection
between the message and the database crash, only a temporal connection. Human pattern detection
over-reports patterns, because a false positive is cheap (Release It! Sec. 17.4, p. 281–283).
**Fix:** record the cause, not only the action. Use the postmortem Root Causes field and a technique
such as 5 Whys (SRE App. D, p. 579, p. 583). A root cause is a defect. After its repair, you have
confidence that the event will not happen again in the same way (SRE Ch. 6, p. 83).

### 🟡 M-29 — A log message that does not name the actor
**Signature:** a message such as "Reset required" with no subject. A message catalog that contains
passive phrasing with no owner of the action.
**Consequence:** the operator performs the wrong action. Nygard's example never says who must perform
the reset, and the code performed the reset itself (Release It! Sec. 17.4, p. 280–283).
**Fix:** write each message for the person who reads it during an incident. Name the actor and the
action. Reserve ERROR for a serious system problem that requires action by operations. A Circuit
Breaker that trips to open is such an error (Release It! Sec. 17.4, p. 278).

### 🔵 M-30 — A log format that neither a person nor a program can read
**Signature:** a multi-line log record. The JDK `java.util.logging` two-line default format. Records
with no severity field and no message code field.
**Consequence:** the eye cannot scan the file, and `grep` has no idea how to handle a two-line record.
Nygard states that this format defeats man and machine. A monitor that matches log patterns misses an
error logged in another format (Release It! Sec. 17.4, p. 280–281, and Sec. 17.5, p. 284–285).
**Fix:** use one line for each record, space-padded columns, a severity indicator, and a message code
field. Externalize all messages into one resource bundle, prefix each with its key as a code, and give
the bundle to operations (Release It! Sec. 17.4, p. 278–281).

### 🟡 M-31 — No trace identifier in the log record
**Signature:** log lines with a timestamp and a message and no request id, session id, or transaction
id. An outbound call that does not carry the inbound identifier.
**Consequence:** after an outage a person must read 10,000 lines with no string to search for
(Release It! Sec. 17.4, p. 283). Node dependencies are the pathways by which cascading failures
propagate (Release It! Sec. 17.7, p. 302).
**Fix:** assign an identifier when the request arrives and include it in every message
(Release It! Sec. 17.4, p. 283). **Modern:** propagate that identifier on every outbound call, so one
trace covers the whole path.

### 🔵 M-32 — State transitions not logged
**Signature:** a component with a state machine, such as a Circuit Breaker or a service that drains,
that logs nothing on a transition.
**Consequence:** the postmortem timeline cannot be built. Nygard states that the record of state
transitions will be important during post-mortem investigations (Release It! Sec. 17.4, p. 283).
**Fix:** log every interesting state transition, even when you also send a notification. Record each
transition as a `Status` entry in the operations database. The type and the frequency of state
changes is often significant for troubleshooting (Release It! Sec. 17.7, p. 302).

### 🔴 M-33 — A secret or a personal identifier in a log line — **Modern**
**Signature:** a log statement that formats a whole request object, a token, a card number, or an
email address. A search for the field name finds it inside a log call.
**Consequence:** you cannot undo the disclosure. Every reader of the log, and every system that ships
the log, now holds the secret.
**Fix:** name every field that you log, and never format a whole object. Nygard gives the neighboring
rule for memory. Core files are memory dumps and contain the passwords, so disable core dumps on
production applications (Release It! Sec. 12.2, p. 228). `security-engineering` owns the rule.

---

## F. Expectations, records, and review

### 🟡 M-34 — Policy baked into the application
**Signature:** a threshold constant, an alert condition, or a health roll-up rule inside application
source. A monitoring framework woven into system internals.
**Consequence:** policy changes at a very different rate than the application code, so every threshold
change needs a release. The coupling is excessive (Release It! Sec. 17.2, p. 275).
**Fix:** expose everything and leave the policy for later (Release It! Sec. 17.6, p. 297). Three
decisions belong outside the application. They are which metrics trigger alerts, where to set the
thresholds, and how to combine state variables into one health status. Keep the monitoring system as
an exoskeleton (Release It! Sec. 17.2, p. 275).

### 🔵 M-35 — A forecast or a baseline with no model identity
**Signature:** a capacity projection, a nominal range, or an anomaly baseline with no reference to the
release that produced it.
**Consequence:** an application release can alter or invalidate the correlations on which the
projections are built. The alert then fires against a world that no longer exists
(Release It! Sec. 17.1, p. 269).
**Fix:** reexamine the correlations after each application release. Attach to every prediction a
reference to the projections you used (Release It! Sec. 17.1, p. 269). Recheck old correlations every
four to six months (Release It! Sec. 17.8, p. 307).

### 🟡 M-36 — An incident with an empty Detection field
**Signature:** a postmortem whose Detection section is blank. An incident that a customer reported
before any alert fired.
**Consequence:** the same class of fault stays invisible. The next occurrence costs the same time.
**Fix:** complete the Detection field of the postmortem. It records what alerted and how
(SRE App. D, p. 579–583). Add the missing rule. Resist a knee-jerk action item that over-fits this one
incident. Specific monitoring is one such item where a unit test catches the problem much earlier
(SRE App. D, p. 583, fn. 165). Record near misses under "Where we got lucky" (SRE App. D, p. 581).
`incident-response` owns the postmortem itself.

### 🔵 M-37 — A report, a dashboard, or a signal with no reader
**Signature:** a scheduled report mailed to a list with no named reader. A series that appears on no
prebaked dashboard and in no alert rule. A rule that has not fired in a quarter.
**Consequence:** the artifact costs money and returns no value. The one time it shows something
serious, the consumers stopped reading it long ago (Release It! Sec. 17.8, p. 305). Unused
configuration is pure maintenance cost (SRE Ch. 6, p. 91).
**Fix:** name the reader and the cadence, and run the review cadences of Sec. 17.8, p. 307–308. Remove
configuration exercised less than once a quarter, and any signal that no dashboard and no alert uses
(SRE Ch. 6, p. 91). Condense samples older than a week (Release It! Sec. 17.7, p. 304).

---

## G. The observer itself

### 🔴 M-38 — The observer can stop the service
**Signature:** a metric write, a log write, or a telemetry database call on the request path with no
timeout and no failure path. A logging call inside a lock.
**Consequence:** the observability plane stops the serving plane. Nygard states that a failure in the
operations database must not affect the system's primary function (Release It! Sec. 17.7, p. 303).
**Fix:** make every telemetry write asynchronous, bounded, and failure-tolerant. Keep the monitoring
plane outside the serving plane (Release It! Sec. 17.2, p. 275). Do not crash mail servers by sending
yourself email alerts in your own server code (SRE App. E, p. 586).

### 🔵 M-39 — The monitor shares fate with the target
**Signature:** a monitoring agent on the same host as the workload with no heartbeat. Monitoring
traffic on the production network segment. One global collector.
**Consequence:** the host fails and the agent fails with it. A network worm or a configuration error
automatically disables monitoring at the moment you need it (Release It! Sec. 17.5, p. 285). One
global collector is a single point of failure (SRE Ch. 10, p. 154).
**Fix:** use a heartbeat to detect a failed agent, and keep monitoring traffic off the segments that
carry public traffic (Release It! Sec. 17.5, p. 285). Run two or more global collectors in different
zones (SRE Ch. 10, p. 154).

### 🔵 M-40 — Monitoring configuration with no test and no version control
**Signature:** alert rules edited in a web console. No rule file in the repository. A rule that names
a variable which a rename can break.
**Consequence:** a rule breaks silently. The exported variable is decoupled from its use in the rules,
so the interface requires careful change management (SRE Ch. 10, p. 144).
**Fix:** keep the rule definitions separate from the target list. Build unit and regression tests by
synthesizing time-series data. Run a continuous integration service that ships the configuration, and
make each monitor validate that configuration before it accepts it (SRE Ch. 10, p. 156–157).


### 🔵 M-44 — An administration interface that a script cannot drive
**Signature:** a graphical console as the only way to read the state of a component or to change it.
No command-line utility.
**Consequence:** the operator repeats the same clicks on every server. A clean shutdown needed several
minutes on each of six servers, so the operators used `kill -9` instead
(Release It! Sec. 14.4, p. 248).
**Fix:** make every administration duty scriptable (Release It! Ch. 15, p. 250). Restrict access to
any interface that can harm the service, because a management console can stop a server
(Release It! Sec. 17.1, p. 274).

---

## Index by symptom

| Symptom | Check these codes |
|---|---|
| A customer reported the outage before the monitoring did | M-04, M-22, M-24, M-26, M-36, M-43 |
| The chart looks healthy and the site is down | M-07, M-08, M-09, M-26, M-42 |
| The on-call engineer ignores the page | M-15, M-17, M-18, M-19, M-20 |
| We cannot tell whether the fault is ours or the vendor's | M-02, M-23, M-31 |
| We cannot tell which change caused this | M-06, M-31, M-35 |
| The data went stale and nobody noticed | M-04, M-05, M-25 |
| The postmortem timeline cannot be built | M-27, M-30, M-31, M-32 |
| The dashboard is green and a feature is broken | M-04, M-26, M-34, M-43 |
| The monitoring itself failed during the incident | M-25, M-38, M-39, M-40 |
| The metric store is slow or full | M-11, M-13, M-37 |
| The service did not restart, and nobody knew | M-25, M-41 |
| An operator needs a developer to answer a question | M-01, M-34, M-44 |
| More servers made the fault worse | M-01, M-42 |
| The alert rule broke after a rename | M-40 |
| The tail latency moved and the chart did not | M-07, M-10, M-12 |
