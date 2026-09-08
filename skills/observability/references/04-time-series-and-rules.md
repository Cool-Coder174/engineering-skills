# Time Series, Rule Evaluation and Instrumentation

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 10 "Practical Alerting from Time-Series Data", p. 141–159, with footnotes 42–55 on
p. 158–159. Supporting cites: SRE Ch. 6, p. 82–96. *Release It!* — Michael T. Nygard
(Pragmatic Bookshelf, 2007), Sec. 17.2, p. 275, and Sec. 17.6, p. 297–298.

Read this file when you build or change the monitoring plane itself. **The monitor** below is
one collector-and-rule-engine process. The book names it Borgmon (SRE Ch. 10, p. 142). **The
monitoring system** is the whole plane of monitors, the alert router and the probers.

---

## 1. The model change

Google's monitoring changed over ten years (p. 142). That change is why this file exists.

| | Check script per target | Time series with central rules |
|---|---|---|
| What runs | A custom script checks a response and alerts | The monitor collects exported variables and evaluates rules |
| Where the chart lives | Separate from the alert path (p. 142) | The same series feeds the chart and the alert (p. 142) |
| History for the alert | None. The process is short-lived (p. 142) | The full history in the arena (p. 143) |
| Cost per check | Subprocess execution and network connection setup (p. 142) | One HTTP fetch per target per interval (p. 143) |
| How the rules scale | The rule count grows with the system (p. 158) | The rule count is independent of the system size (p. 158) |

**Prefer the second model, because the maintenance cost then scales sublinearly with the size of
the service** (p. 158). Google names that property as the key to keeping monitoring maintainable.

## 2. The seven stages, and the signal that starts each one

| Stage | What the engineer does | The signal that tells you to work on this stage |
|---|---|---|
| 1. Instrument | Declares the metric in the code (p. 143) | An operator sees a slow service and cannot name the failing component. Gap M-01 |
| 2. Export | Registers the variable with the HTTP handler (p. 144) | A metric exists in code and no fetch can read it |
| 3. Collect | Scrapes each target on an interval (p. 144) | A target answers no collection and the chart shows a gap. Gap M-25 |
| 4. Store | Sizes the arena and the horizon (p. 146) | A debugging query reaches past the horizon and finds no data |
| 5. Label | Applies the four minimum labels (p. 147) | Two series carry the same name and you cannot separate them |
| 6. Evaluate | Writes recording rules and aggregates (p. 148–151) | A per-task number cannot answer a per-service question |
| 7. Alert | Adds the ratio, the volume floor and the duration (p. 153) | A page fires on two failures out of ten requests. Gap M-16 |

## 3. Instrumentation of applications

**Add one HTTP handler that lists every exported variable in plain text** (p. 143). The handler
writes space-separated keys and values, one pair per line. One HTTP fetch then collects every
metric from one target. **Use a map-valued variable when one name carries a breakdown** (p. 143).
The exporter defines labels on the variable name and exports a table of values or a histogram.
The second line below shows 25 responses with code 200 and 12 responses with code 500.

```
http_requests 37
http_responses map:code 200:25 404:0 500:12
```

**Adding a metric requires a single declaration in the code where the metric is needed** (p. 143).
**The plain text interface declares no schema. That is a trade-off, not a free gift.** The low
barrier helps the software engineering team and the SRE team (p. 143). The variable definition is
decoupled from its use in the rules, so the interface demands careful change management (p. 144).
Google answers the risk with tools that validate and generate rules. Many non-SRE teams use a
generator for the boilerplate and for ongoing updates (p. 158, fn. 45). Gap M-40 rests on this.

## 4. Exporting variables

**Give every binary the same exported variable interface by default** (p. 144). Each major
language at Google implements it, and it registers with the HTTP server that every binary carries.
The variable instance lets the author add an amount to the current value, or set a key to a value.
The Go `expvar` library and its JSON output form are a variant of this API. An application can
also export state through its own service protocol (p. 159, fn. 46). The book names the OpenLDAP
`cn=Monitor` subtree, the MySQL `SHOW VARIABLES` query, and the Apache `mod_status` handler.
**Export every state variable, counter and metric, and keep the policy outside the application**
(Release It! Sec. 17.6, p. 297, and Sec. 17.2, p. 275). Nygard gives two reasons. You are likely
to guess the key metric wrong, and the key metrics also change over time. The per-component expose
lists live in `05-transparency-and-logging.md`.

## 5. Collection of the exported data

The collection loop has five steps (p. 144).

1. Configure the monitor with a target list, through a name resolution method.
2. Prefer service discovery, because the target list is often dynamic.
3. Fetch the exported variable URI on each target at a predefined interval.
4. Decode the results and store the values in memory.
5. Spread the collection over the whole interval, so targets do not collect in lockstep.

**Record four synthetic variables for each target** (p. 144). The monitor produces them and does
not read them from the target. Did the name resolve to a host and a port? Did the target answer a
collection? Did the target answer a health check? What time did the collection finish? These four
make it easy to write a rule that detects an unavailable task.

**Treat a collection failure as a signal, not as a gap in the chart.** Google states that the
collection failure itself is usable as an alerting signal (p. 145). This is the fix for gap M-25.
**Scraping over HTTP contradicts the SNMP design principle, and experience shows that this is
rarely a problem** (p. 145). SNMP aims to keep working when other network applications fail. The
system already survives network faults, and a failed collection is itself an observation.

## 6. Storage in the time-series arena

| Term | Definition | Page |
|---|---|---|
| Data point | A pair of a timestamp and a value | p. 145 |
| Time series | A chronological list of data points, named by a unique labelset | p. 145 |
| Arena | A fixed-size block of memory that holds the time series | p. 146 |
| Horizon | The interval between the newest and the oldest entries in the arena | p. 146 |
| TSDB | The external archive that receives the in-memory state periodically | p. 146 |

**A garbage collector expires the oldest entries once the arena fills** (p. 146). The horizon
therefore states how much queryable data stays in RAM. **Google sizes the datacenter and global
monitors to hold about 12 hours of data** (p. 146). The lowest-level collector shards hold much
less. The book calls the 12-hour horizon a magic number (p. 159, fn. 50). It balances enough
information for debugging an incident in RAM against the cost of that RAM.
**The memory cost of one data point is about 24 bytes** (p. 146). One million unique time series,
held for 12 hours at one-minute intervals, fits in under 17 GB of RAM. Use these two numbers to
size your own arena. Your horizon is a local decision. The TSDB is slower than RAM, and it is
cheaper and larger (p. 146). Query it for older data.

## 7. Labels and the vector

**The name of a time series is a labelset** (p. 146). The labelset is a set of `key=value` pairs,
and one of those keys is the variable name itself. **A series needs four labels at minimum to be
identifiable in the TSDB** (p. 147).

| Label | Meaning (p. 147) |
|---|---|
| `var` | The name of the variable |
| `job` | The name given to the type of server that you monitor |
| `service` | A loosely defined collection of jobs that serve users, internal or external |
| `zone` | The location of the monitor that collected the variable. Google means the datacenter |

Together they form the variable expression (p. 147).

```
{var=http_requests,job=webserver,instance=host0:80,service=web,zone=us-west}
```

**A query does not need every label. A search for a labelset returns every matching series as a
vector** (p. 147). Remove the `instance` label and the same query returns one row per instance.
**Add a duration to query the time axis.** The expression `[10m]` returns the last 10 minutes of
history (p. 148). At one collection per minute, that range holds 10 data points.
**Labels arrive from four places** (p. 148). The target's name gives the job and the instance. The
target itself gives map-valued labels. The monitor configuration gives location annotations and
relabeling. The rules under evaluation give the rest.
**Use three kinds of label, and no more** (p. 157). The first kind is a breakdown of the data
itself, such as the `code` label on `http_responses`. The second kind is the source of the data,
such as `instance` and `job`. The third kind is locality or aggregation inside the service, such
as `zone` for a physical location and `shard` for a logical group of tasks.

**Modern.** Keep the cardinality of a label bounded. Never place a user id, a request id, a
session id or a raw URL path in a label. Gap M-13 states the consequence. Bounded cardinality is
current practice and does not appear in either book.

## 8. Rule evaluation

**Centralize rule evaluation. Do not delegate it to forked subprocesses** (p. 148). Computations
then run in parallel against many similar targets, and the configuration stays small and gains
expressive power. **A rule is a simple algebraic expression that computes a time series from other
time series** (p. 148). A rule can query the history of one series, which is the time axis. It can
query subsets of labels across many series at once, which is the space axis. It can then apply
mathematical operations.

**Rules run in a parallel threadpool where possible** (p. 149). A rule that reads a previously
defined rule depends on the order. The size of the returned vector determines the runtime. Add CPU
to the monitor task when a rule runs slowly. **Monitor the monitoring.** The monitor exports
internal metrics on the runtime of the rules, for performance debugging. Gap M-39 covers the rest.

**Aggregation is the cornerstone of rule evaluation in a distributed environment** (p. 149).
Aggregation sums a set of time series from the tasks in a job, so you treat the job as a whole.
Overall rates follow from those sums. **Prefer a counter to a gauge.** A counter is a
monotonically non-decreasing variable, and a gauge takes any value (p. 149). A counter does not
lose meaning when events occur between two sampling intervals, and a gauge collection is likely to
miss that activity. Gap M-10 names the defect.

### 8.1 The three-step error ratio

The book computes a cluster error ratio in three steps (p. 149–150).

1. Aggregate the rates of response codes across all tasks. Output one rate per code.
2. Sum that vector into a total error rate. Exclude code 200, because it is not an error.
3. Divide the total error rate by the rate of requests that arrived. Output one cluster ratio.

The first two rules read as follows (p. 150).

```
{var=task:http_requests:rate10m,job=webserver} =
  rate({var=http_requests,job=webserver}[10m])
{var=dc:http_requests:rate10m,job=webserver} =
  sum without instance({var=task:http_requests:rate10m,job=webserver})
```

**`rate()` returns the total delta divided by the total time between the earliest and the latest
values** (p. 150). It handles all the corner cases of counter resets (p. 159, fn. 53).
**`sum without instance` removes the instance label from the right-hand side** (p. 150). That
label must go. If it remained in the rule, the monitor could not sum the five rows together
(p. 151).
**Use a history range, not a single point** (p. 151). You need enough data points to compute a
rate, and some recent collections can fail. The `[10m]` range avoids missing data points caused by
collection errors. **Compute the sum of rates, not the rate of sums** (p. 159, fn. 52). This
defends the result against a counter reset, or against missing data from a task restart.

### 8.2 The naming convention

**Give each computed variable a colon-separated triplet** (p. 151). The three parts are the
aggregation level, the variable name, and the operation that created the name.

| Name | Read it as |
|---|---|
| `task:http_requests:rate10m` | task HTTP requests 10-minute rate (p. 151) |
| `dc:http_requests:rate10m` | datacenter HTTP requests 10-minute rate (p. 151) |
| `dc:http_errors:ratio_rate10m` | datacenter HTTP errors 10 minute ratio of rates (p. 152) |

**Compare the ratio rate to the SLO. Alert when the objective is missed, or in danger of being
missed** (p. 151).

### 8.3 Two kinds of rule

| | A recording rule | An alerting rule |
|---|---|---|
| Result | A new time series, appended to its named variable expression (p. 150) | True or false (p. 153) |
| Where the result lives | The arena. You query it as you query a source series (p. 153) | The alert delivery pipeline (p. 154) |
| Why you write it | So a person can inspect the history of error rates later (p. 150) | So the monitoring system produces a page or a ticket (p. 154) |
| What it becomes | An ad hoc query that proves useful becomes a permanent console visualization (p. 153) | An Alert RPC to the alert router (p. 154) |

**Write the recording rule first.** The computed series stays in the arena. A person can then
compare the period before a change to the period after it (p. 150, p. 153). Gap M-06 needs it.

## 9. Alerting

**An alerting rule evaluates to true or false** (p. 153). True triggers the alert. **Alerts flap,
so hold the condition true for a minimum duration before you send the alert** (p. 153). Set that
duration to at least two rule evaluation cycles, so a missed collection causes no false alert. The
book's worked rule fires when the error ratio over 10 minutes exceeds 1% and the errors exceed 1
per second (p. 153).

| Part | The book's value | Why the part exists |
|---|---|---|
| Ratio condition | `> 0.01` | The error ratio over the 10-minute window (p. 153) |
| Volume condition | `> 1` | An absolute rate, so low traffic cannot page (p. 153) |
| Duration | `for 2m` | At least two rule evaluation cycles (p. 153) |
| Name | `ErrorRatioTooHigh` | The on-call engineer reads it first |
| Details template | "webserver error ratio at %trigger_value%" | Context for the responder (p. 153) |
| Severity label | `severity=page` | The router selects the destination by label (p. 154) |

**Both conditions must hold.** In the worked output the ratio sits at 0.15. That value is fifteen
times the threshold, and the alert still stays inactive, because the error count is not above 1
(p. 153). Once the errors exceed 1 per second, the alert goes pending for two minutes. Only then
does it fire.

The delivery pipeline has four steps (p. 153–154).

1. The rule triggers. The monitor fills a template with the job, the alert name and the value.
2. The monitor sends an Alert RPC when the rule first triggers, and a second when it is firing.
3. The Alertmanager routes the notification to the correct destination.
4. The Alertmanager applies inhibition, deduplication, and fan-in or fan-out by labelset.

Inhibition suppresses certain alerts while other alerts are active. Deduplication removes alerts
from several monitors that carry the same labelset. Fan-in and fan-out group or split alerts by
labelset when several similar alerts fire (p. 154).

**Route by class** (p. 154). A page-worthy alert goes to the on-call rotation. An important but
subcritical alert goes to a ticket queue. Every other alert stays as informational data for a
status dashboard. Gap M-17 and gap M-18 apply here. `02-symptom-based-alerting.md` owns the
decision about which condition earns a page.

## 10. Sharding the monitoring topology

**One collector for every task in a global service is a scaling bottleneck and a single point of
failure** (p. 154). Shard the topology instead.

| Layer | What it does | Why it exists |
|---|---|---|
| Scraping-only shard | Collects from tasks only | RAM and CPU limits in one monitor for a very large service (p. 154) |
| Datacenter aggregation | Mostly rule evaluation for aggregation | It monitors all the jobs at one location (p. 154) |
| Global, two or more | Top-level aggregation | Google divides production into zones for changes, so replicas give diversity across maintenance and outages (p. 154) |
| Global, split further | Rule evaluation apart from dashboarding | Used in more complicated deployments (p. 154) |

**A monitor can import time series from another monitor** (p. 154). A streaming protocol carries
the series between monitors, and it saves CPU time and network bytes against the text format.
**Let the upper tier filter the data it streams from the lower tier** (p. 155). The global monitor
must not fill its arena with every per-task series from the tiers below it.

**Model the topology along shared fate** (p. 157). Individual tasks share fate through
configuration files. Jobs in a shard share fate because one datacenter homes them. Physical sites
share fate through networking. Group along these boundaries, so you can isolate a subcomponent
during debugging. Labeling conventions make the division possible, because the monitor adds the
instance name, the shard and the datacenter. Gap M-39 covers a monitor that shares a target's fate.

## 11. Maintaining the configuration

Five rules govern the configuration of the monitoring plane (p. 156–157).

1. **Separate the rule definitions from the target list.** One set of rules then applies to many
   targets at once.
2. **Build libraries of reusable rules with language templates.** Less repetition means fewer
   configuration bugs.
3. **Write unit and regression tests by synthesizing time-series data.** The tests confirm that
   the rules behave as the author believes.
4. **Run a continuous integration service** that executes the test suite, packages the
   configuration, and ships it to every monitor in production.
5. **Make each monitor validate the configuration before it accepts it.**

**A high-level programming environment creates the opportunity for complexity** (p. 156). Rules 3,
4 and 5 answer that risk, and gap M-40 names the defect when they are absent.

| Template class | What it holds | Examples (p. 157) |
|---|---|---|
| Library schema template | The emergent schema of the variables that one code library exports | The HTTP server library, memory allocation, the storage client library, generic RPC services |
| Aggregation template | Generic aggregation rules that model the service topology | Global API, then datacenters, then shards, then jobs, then tasks |

**The exported variable interface declares no schema, and the rule library beside the code library
declares one in effect** (p. 157). The same template serves every tier (p. 158).

## 12. War stories

**The ten-year paradigm change (p. 142, p. 158).** Google's monitoring began with custom scripts
that check a response and alert. Those scripts were wholly separate from the visual display of
trends. The new model made the collection of time series a first-class role. It replaced the check
scripts with a rich language that turns time series into charts and alerts. What this proves: the
alert computation gains a history only when collection leaves a short-lived process.

**varz against SNMP (p. 145).** SNMP is designed for minimal transport requirements and for
continued operation when other network applications fail. Scraping over HTTP seems to contradict
that principle, and the book states that in practice this is rarely an issue. What this proves: an
out-of-band monitoring transport is unnecessary when a failed collection is itself an observation.

**The alert that does not fire (p. 149–153).** Five webserver tasks produce a
`dc:http_errors:ratio_rate10m` of 0.15, which is fifteen times the 0.01 threshold in the rule.
`ErrorRatioTooHigh` still does not fire, because the absolute error rate is not above 1 per second.
What this proves: a ratio alone produces noise at low traffic. Pair it with a volume floor and a
duration before you disturb a person.

**The label that blocks the sum (p. 151).** The `dc:http_requests:rate10m` rule discards the
`instance` label with `sum without instance`. The book states that the monitor could not have
summed the five rows together if that label had remained. What this proves: aggregation is a
labeling discipline, not only arithmetic.

## 13. Ten years on

**What the book states.** Two things, and only two (p. 158). First, decoupling collection from
rule evaluation lets the size of the monitored system scale independently of the size of the
alerting rules. Maintenance cost then scales sublinearly with the size of the service. Second, the
monitoring landscape inside Google evolved over ten years with experiments and changes.

**What the book does not state.** It gives no list of the specific changes that Google would make.
Do not write a list that the book does not give.

**What the book adds about the world outside Google** (p. 142, p. 158). The idea of time-series
data as a source for alerts is now available to everyone. The book names Prometheus, Riemann, Heka
and Bosun, and states that Prometheus shares many similarities with Borgmon in the rule language.

**Modern.** Three practices postdate the book. Bound the cardinality of every label, as gap M-13
requires. Compute a quantile from aggregated buckets, never from a mean of per-instance quantiles,
as gap M-12 requires. Alert on the rate at which the service consumes its error budget, over a
long window and a short window together, as gap M-21 requires. `slo-engineering` owns the budget
policy that a burn-rate rule enforces.

## 14. Failure modes of the monitoring plane

| Failure | Signature | Code |
|---|---|---|
| No exported variable interface | No statistics handler, no metrics route (p. 155) | M-01 |
| A gauge where a counter belongs | A value that can decrease between samples (p. 149) | M-10 |
| An unbounded label | A user id or a raw path inside a labelset | M-13 |
| A rule with one comparison | No volume floor, no duration (p. 153) | M-16 |
| One incident, many pages | No inhibition and no deduplication (p. 154) | M-18 |
| A collection gap read as silence | No synthetic variable for the target (p. 144–145) | M-25 |
| One global collector | A scaling bottleneck and a single point of failure (p. 154) | M-39 |
| Untested monitoring configuration | Rules edited in a console, no test, no version control (p. 156) | M-40 |

The full text of each code lives in `observability-gap-catalog.md`.

## 15. What this file does not decide

| Question | Where it is answered |
|---|---|
| Which quantity to measure | `01-the-four-golden-signals.md` |
| Whether a condition may page a person | `02-symptom-based-alerting.md` |
| Where to place a probe, and what the probe must assert | `03-white-box-and-black-box.md` |
| What to expose per component, and what a log line must carry | `05-transparency-and-logging.md` |
| Where a threshold, a baseline or a review cadence comes from | `06-the-operations-database.md` |
| How to design a percentile, and how tail latency amplifies | `data-systems-design/SKILL.md`, Section 2.1 |
| How to size the service that you measure | `system-design/references/02-estimation.md` |

**When this file stays silent.** It stays silent for a change that adds no metric, no rule, no
collection target and no monitoring configuration. "Not applicable" is a valid result.
