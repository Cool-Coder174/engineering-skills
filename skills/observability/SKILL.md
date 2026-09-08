---
name: observability
description: Decides what a system must expose, what must page a human, and what must only be recorded. Use this skill for monitoring, alerting, metrics, logs, dashboards, and on-call work. Trigger words include pager, SLI, SLO, error budget, health check, probe, heartbeat, telemetry, instrumentation, time series, percentile, saturation, alert fatigue, and runbook. Use it when a change adds a metric, an alert rule, a log line, or a dashboard panel. Use it when an operator cannot answer a question without a developer. Use it after an incident that the monitoring did not detect. The content comes from Site Reliability Engineering (Beyer, Jones, Petoff, Murphy) and Release It! (Nygard).
---

# OBSERVABILITY

**ROLE:** You are the engineer who decides what a system shows to a person.

**CORE FUNCTION:** You receive a change or a design. You name the signals that the system must
expose, the conditions that must page a person, and the conditions that only a store holds.

A system that a person cannot see decays. Nygard states that a system without transparency
cannot survive long in production (Release It! Sec. 17, p. 266). **This skill makes that decay
visible before the code runs.**

---

# 1. WHEN TO USE THIS SKILL

Use this skill for these twelve triggers:

1. You add a metric, a counter, or a gauge.
2. You add or change an alert rule.
3. You add a log message.
4. You add a dashboard panel.
5. You add a probe, a health check, or a heartbeat.
6. You add an integration point, a queue, a cache, or a resource pool.
7. You add a batch job, a data feed, or a scheduled task.
8. You define an SLI, an SLO, or an error budget.
9. You change an on-call rotation, or you write a runbook.
10. An operator cannot answer a question without a developer.
11. An incident happened, and the monitoring did not detect it.
12. A person ignores a page, or a page fires during normal operation.

Do not use it for a copy change, a rename, a test-only change, or a refactor that adds no call,
no state, and no schedule. **"Not applicable" is a valid result, and it beats an invented
finding.** Section 13 states how much output each change class needs.

---

# 2. THE THREE OUTPUTS

A monitoring system has three outputs only. A **page** means a person must act now. A
**ticket** means a person must act within a few days. A **log** means no person must read it
now (SRE App. B, p. 573–574).

## Table 1 — Which output does this condition get?

| The condition | Output | The test it must pass |
|---|---|---|
| A user-visible symptom breaks the SLO now | Page | Urgent, actionable, and needs a person to think (SRE Ch. 6, p. 92) |
| A user-visible symptom will break the SLO in days | Ticket | A person must act within a few days (SRE App. B, p. 573) |
| Zero redundancy, or a part of the service is nearly full | Page | A zero-redundancy state counts as imminent (SRE Ch. 6, p. 96, fn. 25) |
| A cause that is definite and imminent | Page | Only a very definite and very imminent cause may page (SRE Ch. 6, p. 93) |
| A cause that is possible but not imminent | Log | Spend the effort on symptoms instead (SRE Ch. 6, p. 93) |
| One machine of many fails | Log | Single-machine data is too noisy to be actionable (SRE Ch. 10, p. 141) |
| The disk of one machine is full, and no user is affected | Ticket | Not urgent. Not user-visible |
| The response to the condition is one fixed command | Ticket, then automate it | A rote response is a red flag (SRE Ch. 6, p. 95) |
| The condition already pages another team | Nothing | Question 5 of the alert review (SRE Ch. 6, p. 92) |
| Anything that you plan to send to an email alias | Not an output | Email alerts are of very limited value (SRE Ch. 6, p. 96) |

**A condition important enough to disturb a person must page, or must become a bug**
(SRE App. B, p. 574). Google nicknames email alerts alert spam, because people rarely read
them (SRE Ch. 6, p. 96, fn. 22). Move a subcritical condition to a dashboard that shows all
ongoing subcritical problems, and pair that dashboard with a log (p. 96).

---

# 3. THE FOUR GOLDEN SIGNALS

Measure latency, traffic, errors, and saturation. If you can measure four metrics of a
user-facing system, measure these four (SRE Ch. 6, p. 88–89).

## Table 2 — The four golden signals

| Signal | What it measures | The wrong measurement | The right measurement |
|---|---|---|---|
| **Latency** | Time to service a request | One mean over all requests | Two series. Success latency and failure latency. A slow error is worse than a fast error (p. 88) |
| **Traffic** | Demand, in a metric that fits this system | One requests number with no type | HTTP requests per second, split static and dynamic. Network I/O rate. Concurrent sessions. Transactions per second (p. 88) |
| **Errors** | The rate of requests that fail | Only the explicit codes | Three classes. Explicit (HTTP 500), implicit (HTTP 200 with wrong content), and by policy (over one second when you promised one second) (p. 88) |
| **Saturation** | How full the most constrained resource is | 100% utilization as the target | A utilization target below 100%, plus a prediction of impending saturation. The 99th percentile over a one-minute window gives a very early signal (p. 89) |

**Page when one signal is problematic. For saturation, page when it is nearly problematic**
(SRE Ch. 6, p. 89). A load balancer that counts HTTP 500s catches the requests that failed
completely. Only an end-to-end test catches wrong content. Where response codes cannot express
a failure, add a secondary internal protocol for the partial failure modes (p. 88).

**The histogram replaces the mean.** A service with a mean latency of 100 ms at 1,000 requests
per second can easily serve 1% of requests in 5 seconds (p. 89–90). Collect request counts
bucketed by latency, with boundaries that grow by factors of roughly 3 (p. 90).

## Table 3 — Counter or gauge (SRE Ch. 10, p. 149)

| The quantity | Type | Reason |
|---|---|---|
| Total requests served, or bytes written | Counter | A counter keeps its meaning when events fall between two samples |
| Errors by response code | Counter, with a code label | Same. The code breakdown is the diagnosis axis |
| Current queue depth | Gauge | It is a state, not a total |
| Threads in use now, or connections checked out now | Gauge | Same |

**Prefer a counter.** A gauge collection is likely to miss activity between two sampling
intervals (SRE Ch. 10, p. 149). Compute the sum of rates, not the rate of sums. That order
defends the result against a counter reset and against a missed collection
(SRE Ch. 10, p. 159, fn. 52).

---

# 4. SYMPTOM AND CAUSE

The monitoring system answers two questions. What is broken, and why (SRE Ch. 6, p. 86).

## Table 4 — Symptom and cause (SRE Table 6-1, Ch. 6, p. 86–87. Rows 5 and 6 are derived)

| Symptom — page on this | A cause — debug with this |
|---|---|
| I serve HTTP 500s or 404s | Database servers refuse connections |
| My responses are slow | CPUs are overloaded by a bogosort, or a crimped Ethernet cable drops packets |
| Users in Antarctica do not receive animated cat GIFs | The content distribution network blacklisted some client IP addresses |
| Private content is world-readable | A new software push made the ACLs forgotten and allowed all requests |
| *Derived:* the checkout page shows "delivery unavailable" | A vendor connection pool has no connection free |
| *Derived:* the daily statement is missing a day | The batch job did not run |

**In a multilayered system, one person's symptom is another person's cause** (SRE Ch. 6,
p. 87). A slow database read is a symptom to the database engineer and a cause to the frontend
engineer. Across a vendor boundary, your cause is the vendor's symptom. Page on symptoms, and
keep cause rules as debugging aids (p. 93).

**The five alert-review questions.** Answer all five before a rule pages a person (p. 92).

1. Does this rule detect an undetected condition that is urgent, actionable, and user-visible?
2. Will I ever ignore this alert, and know that it is benign? How do I remove that case?
3. Does this alert definitely indicate that users are affected? Which cases must I filter out?
4. Can I act? Is the action urgent? Could a machine perform the action safely?
5. Do other people receive a page for this issue, which makes one page unnecessary?

Delete the rule when question 4 has no answer, or when question 5 shows that another team
already receives a page.

**The four pager rules** (SRE Ch. 6, p. 92-93).

1. A person can react with urgency only a few times a day before fatigue.
2. Every page must be actionable.
3. Every page response must require intelligence. A robotic response must not be a page.
4. A page must report a novel problem.

Never trigger an alert because something seems a bit weird (SRE Ch. 6, p. 84).

---

# 5. WHITE-BOX AND BLACK-BOX

## Table 5 — Two vantage points that fail in opposite directions

Sources: SRE Ch. 6, p. 82, p. 87. SRE Ch. 10, p. 155–156. Release It! Sec. 17.3, p. 276.

| | White-box | Black-box |
|---|---|---|
| Where it runs | Inside the process. The system exposes itself | Outside the process |
| When you add it | During development. Tighter coupling | After delivery, usually by operations. Loose coupling |
| Mechanism | Logs, a profiling interface, or an HTTP handler that emits internal statistics | A protocol check that a prober runs against a target |
| It detects | An imminent problem, a failure masked by a retry, a full queue, a bottleneck | An active problem only. The system is not working correctly, right now |
| It misses | The user's view. It sees only the queries that arrive (SRE Ch. 10, p. 156) | Anything that has not happened yet (SRE Ch. 6, p. 87) |
| How much to use | Heavy use | Modest but critical use (SRE Ch. 6, p. 87) |

**You need both.** With white-box monitoring alone, a query lost to a DNS error is invisible.
You can alert only on the failures that you expected (SRE Ch. 10, p. 156). With black-box
monitoring alone, a retry hides the fault until the retry stops working (SRE Ch. 6, p. 87). A
dead process writes no log message, and a hung process writes none either (Release It!
Sec. 17.4, p. 283). **Instrument both sides of every boundary.** You must know how fast the
web server perceives the database, and how fast the database believes itself to be. Without
both views, you cannot separate a slow database from a slow network (SRE Ch. 6, p. 87).

**The two-point probe method isolates the failing edge** (SRE Ch. 10, p. 156).

1. Run one prober against the load-balanced name that a user reaches.
2. Run a second prober against the servers in each datacenter behind the load balancer.
3. Compare the two answers.
4. If the second view shows that traffic is still served, suppress the alert.
5. If it does not, isolate the edge in the traffic flow graph where the failure occurred.

Validate the response payload at each point, and export values from it as time series (p. 156).

---

# 6. THE FOUR PERSPECTIVES

Each perspective serves a different reader. Each one needs a different technology.

## Table 6 — The four perspectives (Release It! Sec. 17.1, p. 267–275)

| Perspective | The question it answers | Best technology | Do not use it for |
|---|---|---|---|
| **Historical trending** | What did the system do over months? | The operations database | The dashboard. It lacks the immediacy of the present (p. 267) |
| **Predictive forecasting** | What will the system do next? | A model built on past data | The dashboard. A projection is sensitive, and it is not urgent (p. 269) |
| **Present status** | What has the system done? | The dashboard, fed by the operations database (p. 272, p. 300) | Deep diagnosis of one event |
| **Instantaneous behavior** | What is the system doing right now? | The monitoring system, thread dumps, stack traces, log errors, JMX (p. 274) | A wide audience. Restrict access. A JMX console can stop a server (p. 274) |

**Each technology has a fit** (Release It! Fig. 17.10, p. 299). Logging and monitoring are
both good at immediate behavior. Neither one is good at the historical view or the future
view. Present status is derivable from log files alone, and the work is hard. A monitoring
system is great at instantaneous behavior, and it does a good job of present status. Nygard
states that monitoring systems have a way to go on historical and future trends, and he calls
that a gap (p. 299). A prediction is built on a model. A linear projection is a bad model. A
projection about a projection squares the error (p. 269).

## Table 7 — Dashboard color (Release It! Fig. 17.1, Sec. 17.1, p. 273)

| Color | Condition |
|---|---|
| **Green** | **All** of these are true. All expected events occurred. No abnormal event occurred. All metrics are nominal. All states are fully operational |
| **Yellow** | **At least one** of these is true. An expected event did not occur. A medium-severity abnormal event occurred. A parameter is above or below nominal. A noncritical state is not fully operational, such as a breaker that cut a noncritical feature |
| **Red** | **At least one** of these is true. A **required** event did not occur. A high-severity abnormal event occurred. A parameter is far above or below nominal. A critical state is not at its expected value, such as "accepting requests" being false |

**The dashboard must show the absence of a required event, not only its failure.** Nygard
reports business problems traced to batch jobs that failed invisibly for 33 days straight
(p. 272). One dashboard carries several facets. Operations needs a component view, developers
an application view, and sponsors a business-process view. The dashboard must know the linkages
between these views (p. 272).

---

# 7. WHAT TO EXPOSE

Expose every state variable, every counter, and every metric (Release It! Sec. 17.6, p. 297).

## Table 8 — What to expose, by component

Sources: Release It! Sec. 17.1, p. 270–271, and Sec. 17.6, p. 297–298. Every counter carries
an implied "in the last n minutes" or "since the last reset" (p. 298).

| Component | Expose these |
|---|---|
| **Request channel** | Total requests processed, average response time, requests aborted, requests per second, time of last request, accepting traffic or not |
| **Traffic** | Page requests total, page requests, transaction counts, concurrent sessions |
| **Thread pool** | Number of threads, threads busy, **threads busy more than five seconds**, high-water mark, low-water mark, times a thread was not available, request backlog |
| **Resource pool** | Enabled state, total resources, resources checked out, high-water mark, resources created, resources destroyed, times checked out, **threads blocked waiting for a resource**, times a thread has blocked waiting |
| **Database connection** | Number of SQLExceptions thrown, number of queries, average response time to queries |
| **Integration point** | State of the Circuit Breaker, number of timeouts, number of requests, average response time, number of good responses, **network errors, protocol errors, and application errors as three counters**, actual IP address of the remote endpoint, current concurrent requests, concurrent request high-water mark |
| **Circuit Breaker** | Current state, **manual override applied**, number of failed calls, time of last successful call, number of state transitions |
| **Cache** | Items in cache, memory used, hit rate, items flushed by the garbage collector, configured upper limit, time spent creating items |
| **Business transaction** | Number processed, number aborted, value, transaction aging, conversion rate, completion rate |
| **Batch job** | Items needing work at start, items actually processed at end, abnormal termination (Sec. 17.7, p. 302–303) |
| **Memory** | Minimum heap size, maximum heap size, generation sizes, garbage collection type and frequency, memory reclaimed |

**Why the list is long.** You are likely to guess the key metric wrong, and the key metrics
change over time. A medium-sized application can hold hundreds of parameters (p. 271). Expose
everything now, and keep the policy outside the application (p. 297). The choice of metric, the
threshold value, and the health status rule belong outside the application. Policy changes at
a very different rate than the code (Release It! Sec. 17.2, p. 275).

---

# 8. THE LOG CONTRACT

A log file is a human-computer interface (Release It! Sec. 17.4, p. 280). Write each message
for the person who reads it during an incident.

## Table 9 — What log level does this event get? (Release It! Sec. 17.4, p. 278, p. 283)

| The event | Level | Reason |
|---|---|---|
| A Circuit Breaker moves to open | ERROR | Action is probably required at the other end of the connection |
| The application cannot connect to the database | ERROR | A serious system problem |
| A user typed a bad credit card number | A warning, if you log it at all | Business logic and user input are not system problems |
| A `NullPointerException` | Decide by the effect | An exception is not automatically an error |
| An interesting state transition | Always log it | The record matters during a postmortem (p. 283) |
| A debug or trace message in production | Never ship it | A build step must remove any configuration that enables debug or trace (p. 278) |

**Anything logged at ERROR or SEVERE must require action by operations** (p. 278).

**The format rules** (Release It! Sec. 17.4, p. 277, p. 280–281, and Sec. 17.5, p. 285).

1. Write one line for each record. A two-line record defeats the reader and the program.
2. Use space-padded columns, so the eye can scan the file.
3. Use a single-character severity indicator.
4. Add a message code field, which helps a program parse the file.
5. Make the log file location configurable, so an administrator can use a separate device.
6. Use one uniform error format, so an unexpected error is still caught.

**The message-code method** (Release It! Sec. 17.4, p. 278–279).

1. Move every message into one resource bundle.
2. Prefix each message with its key, and use that key as the code.
3. Give the bundle to operations as the catalog of messages.

**Name the actor and the action.** A message such as "Reset required" does not say who must
perform the reset. Nygard traces six months of unnecessary weekly database failovers to that
one message (p. 280–283).

**The trace identifier.** Assign an identifier when the request arrives, and include it in
every message. Use a user id, a session id, a transaction id, or an arbitrary number (p. 283).
After an outage a person must read 10,000 lines, and the identifier gives one string to search
for. **Modern:** propagate that identifier on every outbound call, so one trace covers the
whole path. Cross-service trace context does not come from either book.

---

# 9. EXPECTATIONS AND THE OPERATIONS DATABASE

An expectation is an allowed range, an allowed time, or an allowed status. A violation of any
expectation must trigger an alert (Release It! Sec. 17.7, p. 303). The operations database
holds three kinds of observation. A **Measurement** is a periodic value. A **Status** is a
recorded state transition. An **Event** is a point-in-time occurrence (p. 301–302).

## Table 10 — How an expectation matures (Release It! Sec. 17.7, p. 303–304)

The worked parameter is web server CPU utilization.

| Stage | The expectation | Why |
|---|---|---|
| Start loose | Greater than 0% and less than 80% | A loose range avoids a false positive on day one |
| Tighten | Greater than 5% and less than 50% | The recorded history now shows the real range |
| Add the rhythm | Low at night, moderate in the morning, high in mid-afternoon | Traffic follows a business cycle |

**History sets the expectation.** The best source is the historical data already in the
database. Set the expectation to match reality, so you avoid the negative effects of false
positives (p. 303–304). A continuous metric is nominal inside the mean for the period plus or
minus two standard deviations. The period is the hour of the week, and the day of the month
means little (Release It! Sec. 17.1, p. 271).

**The observer must not stop the service.** Nygard states that the operations database is not
critical to the financial success of the system. No failure in it may have a noticeable effect
on the primary function (p. 303). Condense old data. Minute-by-minute samples older than a
week are not helpful (p. 304).

---

# 10. THE ALERT RULE SHAPE

## Table 11 — The shape of an alert rule (SRE Ch. 10, p. 149–153, p. 159, fn. 52)

| Part | The rule | The book's worked example |
|---|---|---|
| Ratio | Compare the error rate to the request rate | `dc:http_errors:ratio_rate10m > 0.01` |
| Volume floor | Add an absolute rate, so low traffic cannot page | `and dc:http_errors:rate10m > 1` |
| Window | Compute the rate over a history range, not one point | `[10m]` |
| Duration | Hold the condition for at least two rule evaluation cycles | `for 2m` |
| Severity | Route by a label, not by the rule name | `labels { severity=page }` |
| Aggregation | Drop the instance label before you sum | `sum without instance` |
| Counter safety | Compute the sum of rates, not the rate of sums | Survives a counter reset or a missed collection |
| Objective | Compare the result to the SLO. Fire when the SLO is missed or in danger | SRE Ch. 10, p. 151 |

**Both conditions must hold.** In the book's worked output the ratio is 0.15, which is fifteen
times the threshold, and the rule still does not fire. The absolute error rate is not above 1
per second (SRE Ch. 10, p. 153).

## Table 12 — What the availability target demands of the monitoring

`slo-engineering` owns the availability table, the downtime arithmetic, and the error budget.
Read `slo-engineering/references/03-availability-math.md` for the numbers. This table names
only the demand that each target places on detection.

| Target | What the target demands of the monitoring | Source |
|---|---|---|
| 99% | A ticket queue can carry most conditions | Derived |
| 99.9% | A 200-status probe more than once or twice a minute is probably unnecessarily frequent | SRE Ch. 6, p. 90 |
| 99.9% | A check of hard drive fullness more than once every one or two minutes is probably unnecessary | SRE Ch. 6, p. 90 |
| 99.99% | About 13 minutes of downtime per quarter, so on-call reaction time must be on the order of minutes | SRE Ch. 11, p. 161 |
| 99.999% | No step with a person on the critical path | Derived |

**Keep the word "probably".** The book hedges both probe intervals (SRE Ch. 6, p. 90). Derive
your own interval from the availability target, and record the derivation. A detection delay
that exceeds the allowed downtime makes the target unreachable.

## Table 13 — Which signal or rule to remove

Sources: SRE Ch. 6, p. 85–86, p. 91. Release It! Sec. 17.8, p. 308.

| The test | The action |
|---|---|
| The rule is exercised less than once a quarter | Candidate for removal |
| The signal appears on no prebaked dashboard and in no alert | Candidate for removal |
| The metric stopped producing useful information | Stop reviewing it |
| The rule tries to learn its own threshold or detect causality | Remove it. Avoid magic systems |
| The rule depends on a chain of other conditions | Remove it, unless every part of the chain is very stable |

**The one endorsed dependency rule.** "If a datacenter is drained, then do not alert me on its
latency" is acceptable. Traffic draining is a very stable part of the system.

## Table 14 — The three limits that gate a new alert rule

`incident-response` owns the on-call load budget. Read
`incident-response/references/06-interrupts-and-overload.md` for the full table, the team
size, and the toil ceiling. Check a new page against these three limits only.

| Measure | Limit | Source |
|---|---|---|
| Incidents per 12-hour on-call shift | 2 maximum | SRE Ch. 11, p. 163 |
| Alerts per incident | Approach 1 to 1 | SRE Ch. 11, p. 167 |
| Page frequency review | Quarterly, with management | SRE Ch. 6, p. 95 |

**The budget is a capacity calculation.** An incident costs about six hours end to end, so a
12-hour shift absorbs two. If one component pages every day, another break will produce more
incidents than the shift permits (SRE Ch. 11, p. 163).

---

# 11. THE GAP SCAN (MANDATORY)

Run `references/observability-gap-catalog.md` against the change before you accept it. The
catalog holds forty-four named gaps, M-01 to M-44, in seven groups. Each entry gives a
**Signature**, a **Consequence**, and a **Fix**. Report only the codes that apply.

```md
### Gap Scan
| Code | Present | Evidence | Required fix |
|---|---|---|---|
| M-08 Failed requests removed from latency | Yes | client.py:64 starts the timer inside the 200 branch | Record a failure-latency series and a timeout counter |
| M-17 Alerts routed to email | No | Routes are page and ticket only | — |

```

An honest empty result is valid. `code-review` runs the applicable groups against a difference.
`verify` runs them against an implemented phase. `planner` and `detail-planning` run them
against a design.

---

# 12. THE OBSERVABILITY RECORD

This skill produces one artifact. `detail-planning` copies it into `executor.md`. `verify` and
`code-review` check the change against it. Sections 1 to 4 are mandatory for every surface.
Sections 5 to 13 follow Table 15.

```md
## Observability Record: [surface name]

### 1. What a user sees
`slo-engineering` owns the indicator, the objective, and the dependency inheritance rule. Copy
its answers here. Do not derive them in this record.

- Feature or business process: [name]
- SLI: [the measurement, in the user's terms]. SLO: [target] over [window].
- Measured at: [client-side / edge / server-side]. Client-side is the default (SRE App. B, p. 572).
- Worst external dependency: [name and its SLA]

### 2. The probe
`slo-engineering` owns the availability definition that these fields serve. This skill owns
the second vantage point and the payload assertion.

| Item | Value |
|---|---|
| What the synthetic transaction does | |
| Designated user id, so it does not pollute production data | |
| Frequency, and number of locations | |
| Maximum acceptable response time per step | |
| Response codes or text patterns that mean success | |
| Response codes or text patterns that mean failure | |
| Assertion level, 3 or higher | |
| Second vantage point, behind the load balancer | |
| Where the data is recorded | |

(SRE Ch. 10, p. 156.)

### 3. The four golden signals
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

### 4. Outputs
| Condition | Output | Rule | Duration | Route | Runbook action |
|---|---|---|---|---|---|
| | page / ticket / log | | | | |

Email is not an output. (SRE Ch. 6, p. 96. SRE App. B, p. 574.)

### 5. Alert review
Answer the five questions of Section 4 for every page. (SRE Ch. 6, p. 92.)

### 6. Required events
| Event | Expected time | Items expected | Alert on absence | Owner |
|---|---|---|---|---|

(Release It! Sec. 17.1, p. 272, and Sec. 17.7, p. 302-304.)

### 7. Instrumentation exposed, from the per-component lists in Table 8
| Component | Instance | Exposed list applied | Handler or endpoint |
|---|---|---|---|

### 8. Log contract
- Trace identifier field: [name]. Assigned at [entry point]. Carried on [outbound calls].
- Message code catalog: [path to the resource bundle]
- ERROR means: an operator must act.
- State transitions logged: [list]
- Fields that must never appear: [secrets and personal identifiers]
- Build step that removes debug and trace configuration: [name of the step]

### 9. Expectations
Nominal is the mean for the period plus or minus two standard deviations, over the hour of the
week. (Release It! Sec. 17.1, p. 271.) Stage is loose, tight, or rhythm.

| Parameter | Nominal range | Period | Source of the range | Stage |
|---|---|---|---|---|

### 10. Gap scan
List only the codes that apply. "Not applicable" is a valid result.

| Code | Present | Evidence | Required fix |
|---|---|---|---|

### 11. What an operator can answer alone
Name the three questions an operator must answer without a developer. Nygard states that
transparent systems communicate, and in communicating they train their attendant humans.
(Release It! Sec. 17, p. 265.)

### 12. Removal list
Remove configuration exercised less than once a quarter. Remove a signal that no dashboard and
no alert uses. (SRE Ch. 6, p. 91.)

| Signal or rule | Reason | Action |
|---|---|---|

### 13. Load check
`incident-response` owns the full on-call budget. Check these two limits here.

| Measure | This surface | Limit |
|---|---|---|
| Incidents per 12-hour shift | | 2 |
| Alerts per incident | | 1 |

(SRE Ch. 11, p. 163, p. 167.)
```

---

# 13. PROPORTIONALITY

Rigor scales with blast radius. A full Observability Record for a copy change is itself a defect.

## Table 15 — How much output each change needs

| Change class | Required output from this skill |
|---|---|
| A copy change, a rename, no new call and no new state | Nothing |
| A new endpoint or job over a surface that is already instrumented | Gap scan only |
| A new integration point, queue, cache, or resource pool | Gap scan, the expose list for that component, and one alert rule |
| A new user-facing feature with its own SLO | Full Observability Record |
| A new service, or a new on-call surface | Full Observability Record, plus the alert load budget |
| Anything with money, a payment rail, or an irreversible action | Full Observability Record, plus the required-event check and the data-volume check |
| An incident review | The Detection section and the Removal list only |

**When this skill stays silent.** It stays silent for a change that adds no call, no state, no
schedule, and no new failure mode. "Not applicable" is a valid result, and it beats an invented
finding. Unused collection and alerting configuration is pure maintenance cost (SRE Ch. 6,
p. 91).

---

# 14. LANGUAGE DISCIPLINE, GLOSSARY, AND REFERENCES

## 14.1 Forbidden phrases

| Forbidden phrase | Must be replaced with |
|---|---|
| "we will monitor it" | The series name, the type, where it is exposed, and the output it feeds |
| "we will add an alert" | The rule, the volume floor, the duration, the route, and the runbook action |
| "add logging" | The level, the message code, the trace identifier, and the fields that must not appear |
| "the dashboard shows health" | The expected events, the parameters, and the color rule that combines them |
| "p99 latency" | The bucket boundaries and the aggregation method |
| "it is healthy" | The probe, its success pattern, and its vantage points |
| "the team watches it" | The reader, the cadence, and the action that the review produces |
| "we will know if it breaks" | The signal, the detection latency, and the output |
| "anomaly detection" | The expectation, its period, and the historical data it came from. Avoid a system that learns its own thresholds (SRE Ch. 6, p. 85) |

## 14.2 Glossary — one word, one meaning

| Word | Meaning in this skill |
|---|---|
| **signal** | A measured quantity that the system exports, or that a probe produces |
| **series** | A named, labelled sequence of timestamped values, from a counter or a gauge |
| **page** | A notification that wakes a person now |
| **ticket** | A notification that a person must act on within a few days |
| **log** | A record that no person must read now |
| **probe** | An outside check that acts as a user acts |
| **symptom** | A fault that a user can observe |
| **cause** | A fault that explains a symptom |
| **expectation** | An allowed range, an allowed time, or an allowed status |
| **nominal** | Inside the expectation |
| **incident** | A sequence of events and alerts with one root cause, and one postmortem (SRE Ch. 11, p. 163) |
| **push** | Any change to running software, or to its configuration (SRE Ch. 6, p. 83) |
| **expose** | To make an internal value readable from outside the process |
| **record** | To write a value into a store for later reading |

## 14.3 Reference index

| Reference | Content | Read it when |
|---|---|---|
| `references/01-the-four-golden-signals.md` | The four signals and the wrong measurement for each one. Success latency and failure latency. Saturation as the leading indicator. Distributions instead of averages. Resolution, and the cost of over-collection. Source: SRE Ch. 6 | You add or review any metric |
| `references/02-symptom-based-alerting.md` | Symptom and cause. The five alert-review questions. Page, ticket, and log. Why an alert that a person cannot act on is a defect. Alert fatigue as a reliability risk. Sources: SRE Ch. 6, Ch. 10, Ch. 11 | You add or review any alert rule |
| `references/03-white-box-and-black-box.md` | The two vantage points, and what each one alone misses. The prober. The difference between a symptom-oriented and a cause-oriented outside check. Sources: SRE Ch. 6, Ch. 10 | You add a probe, a health check, or a synthetic transaction |
| `references/04-time-series-and-rules.md` | Instrumentation, the exported variable, collection by scraping, the time-series arena, labels and the vector, rule evaluation, sharding, and configuration maintenance. Source: SRE Ch. 10 | You build or change the monitoring plane itself |
| `references/05-transparency-and-logging.md` | The four perspectives. Design for transparency from the start. Enabling technologies. What to log, log levels, message codes, and the rule that a log message is a user interface. Source: Release It! Ch. 17 | You add a log message, a dashboard, or an admin view |
| `references/06-the-operations-database.md` | Where metric, configuration, expectation, and observation meet. What the database holds. Feeding it. Setting expectations and detecting drift. The review cadences, and the dashboard that nobody reads. Sources: Release It! Sec. 17.7, Sec. 17.8 | You set a threshold, a baseline, or a review cadence |
| `references/observability-gap-catalog.md` | Forty-four named gaps, M-01 to M-44, in seven groups, with an index by symptom | Every review, and every incident that the monitoring did not detect |

## 14.4 The five skills that share the word "system"

| Skill | The question it answers | Level |
|---|---|---|
| `system-design` | What must we build? | Architecture |
| `data-systems-design` | Does the design stay correct across machines? | Distributed data |
| `systems-programming` | Does the code stay correct against the kernel? | One machine |
| `security-engineering` | Can an attacker break it? | Threat |
| `observability` (this skill) | Can a person see it, and must it wake a person? | Operation |

## 14.5 Who owns each overlap

This skill never restates a rule that another skill owns. It links instead.

| Overlap | Owner | What `observability` does instead |
|---|---|---|
| Requirement capture, root-cause analysis, the phased plan | `planner` | Supplies the probe fields and the signal list that the plan must carry |
| The SLI, the SLO, the error budget, the availability table, burn-rate policy | `slo-engineering` | Reads the objective. It names the series that measures it |
| On-call load, pager budget, alert fatigue, escalation, the postmortem | `incident-response` | Names the three limits that gate one new rule, and the Detection field |
| Saturation limits, headroom, load tests, the utilization target | `capacity-engineering` | Names saturation as a signal. It does not set the limit |
| Finding the cause of a live fault | `production-troubleshooting` | Supplies the evidence that the method reads |
| Capacity numbers, load estimation | `system-design` | Measures what the estimate predicted. It does not size the system |
| Percentile design rules, tail amplification, isolation, dual writes | `data-systems-design` | Owns the collection shape only. Buckets, counters, and label discipline |
| Durable writes inside the logging path | `systems-programming` | Owns the log format and the log contract, not the write mechanics |
| Secrets, personal data, audit records, console access | `security-engineering` | Names the gap M-33, and hands the classification rule over |
| Timeouts, Circuit Breakers, bulkheads, load shedding, fallback | `self-healing-apis` | Owns the integration-point expose list, and the alarm on breaker state |
| Bounded remediation, stop-and-notify, automated failover | `self-healing-apis` | Owns the give-up signal, and the rule that a rote page must be automated |
| Revert mechanics, canary stages, the rollback decision | `reverse-branching` | Owns the deploy marker M-06, the post-push data check M-05, and the probe |
| The executor specification for one phase | `detail-planning` | Supplies the Observability Record as that phase's observability section |
| Applying invariants in code | `implement` | Supplies the expose lists and the log contract as implementation targets |
| Running the M- scan against a difference or a phase | `code-review`, `verify` | Owns the catalog, and the check on the Detection field |
| Pipeline order, gates, and history | `engineer-workflow` | Places the gap scan after implementation and before review |

**Where a sibling must call this skill.** `self-healing-apis` must call this skill before it
adds any automatic remediation. The book gates automation on alert-review question 4
(SRE Ch. 6, p. 92). `reverse-branching` must call this skill for the verification signal of any
automatic revert. In Nygard's incident the team declared recovery when the external synthetic
monitor turned green, about ninety seconds after the change (Release It! Sec. 16.10, p. 263).

## 14.6 One statement of scope

Neither book gives a retry budget, a Circuit Breaker mechanism, a rollback procedure, or a
canary analysis method. Those belong to `self-healing-apis` and `reverse-branching`. Neither
book gives cross-service trace context or label cardinality limits. Both carry the tag
**Modern** wherever this skill names them. Burn-rate alerting is also **Modern**, and
`slo-engineering` owns the policy that a burn-rate rule enforces.

---

**Attribution:** the concepts, the terminology, and the page references in this skill and in its
references come from two books. The first is *Site Reliability Engineering: How Google Runs
Production Systems* by Betsy Beyer, Chris Jones, Jennifer Petoff, and Niall Richard Murphy
(O'Reilly Media, 2016). The second is *Release It! Design and Deploy Production-Ready Software*
by Michael T. Nygard (Pragmatic Bookshelf, 2007). Page numbers refer to those editions.
