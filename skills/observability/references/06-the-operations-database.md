# The Operations Database and Supporting Processes

**Sources:** *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard
(Pragmatic Bookshelf, 2007), Sec. 17.7 "Operations Database", p. 299–305, and Sec. 17.8
"Supporting Processes", p. 305–309. Supporting material comes from Release It! Sec. 17.1,
p. 267–275, Sec. 17.2, p. 275, Sec. 16.8, p. 261, and Pattern 5.4 "Steady State", p. 124. Two
rules come from *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 6, p. 91, and App. F, p. 588–590.

The operations database is the place where a metric, a configuration, an expectation and an
observation meet. Nygard calls it the OpsDB. It accumulates status and metrics from every
server, application, batch job and feed in the extended system (Release It! Sec. 17.7, p. 299).

**A log and a monitoring system both show immediate behavior. Neither one serves the historical
view or the future view** (Release It! Sec. 17.7, p. 299). This file describes the store that
serves those two views, and the review processes that make the stored data useful.

---

## 1. The gap this store fills

Source: Release It! Fig. 17.10, Sec. 17.7, p. 299.

The figure rates each technology on a four-level scale. The levels are unsuitable, poor fit,
workable with effort, and well suited. The cells below carry the judgment that the prose on p. 299
states.

| Technology | Instantaneous behavior | Present status | Historical trending | Future forecasting |
|---|---|---|---|---|
| **Logging** | Good at immediate behavior | Workable, with effort | Not particularly good | Not particularly good |
| **Monitoring system** | Great | A good job | A gap, in Nygard's judgment | A gap, in Nygard's judgment |
| **Operations database** | Not the view it serves | It feeds the dashboard | Well suited | Well suited |

**You can derive present status from log files alone. The work is hard.** A person must trace
backward to the last state transition for each status variable. Write the status variables on a
recurring timer to make that analysis easier (Release It! Sec. 17.7, p. 299).

**Keep the book's hedge on monitoring systems.** Nygard states that they have a way to go on
historical and future trends. He calls this a gap until the management suites mature (p. 299).

**Modern.** A time-series database with long retention plus a metadata store now covers the
same two views. The rules below describe what the store holds and who reads it, not a product.

---

## 2. What the store is for

Source: Release It! Fig. 17.11, Sec. 17.7, p. 300.

The store gives the "single pane of glass". It presents the business metrics on the dashboard,
the system statistics, and the correlations between the two (Release It! Sec. 17.7, p. 300).

| Direction | Who | What moves |
|---|---|---|
| **Into the store** | Servers | Performance and utilization |
| **Into the store** | Applications | Status variables, business metrics, internal metrics |
| **Into the store** | Batch jobs | Start, end, abort, completion status, items processed |
| **Out of the store** | Reports | The historical record |
| **Out of the store** | Dashboard | The present status view |
| **Out of the store** | Capacity planning | Demand metrics and system metrics together |

**A log gives visibility into one application. This store unifies status and metrics across the
whole system** (Release It! Sec. 17.7, p. 300).

**Two uses appear only after the data accumulates.** Data mining reveals the correlation factors
that capacity planning needs. The historical record allows automatic baselining, which
determines what normal looks like across the site (p. 300).

**Job execution records make troubleshooting faster.** Nygard names one problem that they
solve, which is stale data from a channel partner. Correlate the normal start and end times of a
job with the number of items processed. The correlation shows which job is in danger of breaking
its window (Release It! Sec. 17.7, p. 300).

---

## 3. What the store holds

Source: Release It! Fig. 17.12, Sec. 17.7, p. 301–302.

| Class | Definition | The rule that governs it |
|---|---|---|
| **Feature** | A unit of business-significant functionality | Use the same features that the availability SLAs are written about, and that capacity planning measures (p. 301) |
| **Node** | Any active node that delivers the feature | Each Node has a unique id. Every producer reports under that id (p. 302) |
| **Observation** | A single data point collected from a Node | The heart of the store (p. 302) |
| **ObservationType** | The name and the concrete subtype of an Observation | The set of all ObservationTypes defines the universe of information in the store (p. 302) |
| **Measurement** | A periodic observation | All records are kept (p. 302) |
| **Status** | A recorded state transition | For the dashboard, only the last Status matters. For troubleshooting and trending, the frequency and the type of state changes is often significant (p. 302) |
| **Event** | A discrete occurrence | Recorded against its ObservationType, like the other observations (p. 301) |

**A Feature does not come from one server.** It probably spans at least three hosts. It also
spans an application on each tier, a pair of firewalls, a switch and a storage array
(Sec. 17.7, p. 301).

**Hosts and applications are usually enough for the Node set.** Add a hardware firewall, an SSL
accelerator, or a content caching server when it delivers the application actively. Add other
network equipment on the same test. Assign blocks of node ids to a team, so that the ids stay
contiguous (Release It! Sec. 17.7, p. 301–302).

**Node dependencies are optional, and they are burdensome to capture.** They aid
troubleshooting. Those same dependencies are also the pathways along which cascading failures
propagate (Release It! Sec. 17.7, p. 302).

**A Circuit Breaker records each of its state transitions as a Status entry** (Release It!
Sec. 17.7, p. 302). An application server records the change from enabled to disabled the same
way. Catalog code M-32 in `observability-gap-catalog.md` names the component that logs no
transition. The book leaves the SQL schema to the reader, because the ORM tool and the database
server should drive that mapping (p. 301).

---

## 4. Feeding the store

**Create a client-side API in the language that most of the system uses. Above all, keep it
simple** (Release It! Sec. 17.7, p. 302).

| Producer | The feed path | Source |
|---|---|---|
| An application in the main language | The client-side API | p. 302 |
| A shell script or a batch file | A command-line utility that writes to the store | p. 302–303 |
| A Java application | A generic MBean that records periodic samples and receives state-change notifications | p. 303 |

**A failure in this store must not affect the primary function.** Nygard states that everything
can fail, including this new database. The store is not critical to the financial success of the
system (Release It! Sec. 17.7, p. 303). Catalog code M-38 names the write that stops a service.

**Signal that the feed path is wrong:** a metric write sits on the request path with no timeout
and no failure branch. Make every write asynchronous, bounded and failure-tolerant (Release It!
Sec. 17.2, p. 275).

### 4.1 The batch job procedure

Source: Release It! Sec. 17.7, p. 302–303.

1. At start, call the command-line utility with the count of items that need work.
2. At the end, record how many items the job actually processed, not the count that had issues.
3. If the job fails completely, record an abnormal termination.

**A required event needs a record of its absence, not only a record of its failure.** Nygard
traces a startling number of business-level problems to one cause. Batch jobs failed invisibly for
33 days straight (Release It! Sec. 17.1, p. 272). Catalog code M-04 names this gap.

**The MBean option removes most of the instrumentation work.** Instrumenting an application
then means instantiating and configuring that MBean (Release It! Sec. 17.7, p. 303).

---

## 5. The expectation

Source: Release It! Fig. 17.13, Sec. 17.7, p. 303.

An ExpectationType corresponds to an ObservationType. It defines the name and the
characteristics of its Expectation (Release It! Sec. 17.7, p. 303).

| Subtype | What it states | Example |
|---|---|---|
| **NominalRange** | An allowed range for a metric | Web server CPU utilization above 5% and below 50% (p. 304) |
| **ExpectedStatus** | An allowed status | A Circuit Breaker in the closed state |
| **ExpectedTime** | A time frame in which an event must occur, or must not occur | The nightly extract completes inside its window (p. 300) |

**A violation of any expectation should trigger alerts in the monitoring system** (Release It!
Sec. 17.7, p. 303). The expectation lives in the store. Catalog code M-34 names the threshold
that a developer wrote into application source.

**The best source for an expectation is the historical data already in the store** (Release It!
Sec. 17.7, p. 303–304). Do not invent a threshold. Read the recorded range first. Set the
expectation to match reality, so that you avoid the negative effects of false positives
(Release It! Sec. 17.7, p. 304).

**The nominal test for a continuous metric.** A metric is nominal when it sits inside the mean
for the period, plus or minus two standard deviations. The period with the most stable
correlation for a traffic-driven metric is the hour of the week (Release It! Sec. 17.1, p. 271).

---

## 6. How an expectation matures

Source: Release It! Sec. 17.7, p. 304. The worked parameter is web server CPU utilization.

| Stage | The expectation | The signal to move to the next stage |
|---|---|---|
| **Loose** | Above 0% and below 80% | The store holds enough history to show the real range |
| **Tight** | Above 5% and below 50% | You learn more about the system linkages |
| **Rhythm** | Low at night, moderate in the morning, high in mid-afternoon | The recorded traffic follows a business cycle |

**Start loose.** In the beginning, expectations can be somewhat loose (Release It! Sec. 17.7,
p. 304). Tighten them as the processes come under better control (p. 304).

**The rhythm stage can be a continuous envelope or a step function.** A deviation above or
below that expectation triggers an alert (Release It! Sec. 17.7, p. 304). Nygard states that
the system then knows itself better (p. 304).

**Record every expectation change as a tracked change.** The SRE production meeting raises the
acceptable delay threshold of one alert from 60 s to 180 s. That change reduces unactionable
alerts. It carries a bug number and a named owner (SRE App. F, p. 589). Catalog code M-40 names
monitoring configuration with no test and no version control.

### 6.1 War story — the warning chime the operator silenced

James Chiles reports a plant supervisor who saw an operator override a control-room warning
chime by reflex. The operator then denied the act. He denied it because he had no conscious
recollection of silencing the warning. The system had trained him to disregard the chime so
completely that he could silence it without awareness (Release It! Sec. 17.7, p. 304). An alarm
that fires during normal operation destroys the detection that it was bought for.

### 6.2 War story — the pager that rang three times every night

A developer told Nygard that her pager fired three times every night, and that this indicated
normalcy. A missing third page before a certain hour meant a problem. A fourth page meant a
problem. Nygard grants that this is a form of situational awareness. He states that he cannot
endorse it as a way of life (p. 304). A team builds a mental model around noise when the
expectation does not match reality.

### 6.3 War story — the on-call engineer who ignored a real page

During Nygard's Black Friday outage, a scheduling system ran at 100% CPU and its on-call
engineer did not respond. That group is paged routinely for transient CPU spikes that prove to
be false alarms. Nygard states that the false positives had trained them to ignore high CPU
conditions (Release It! Sec. 16.8, p. 261). The alarm that mattered was indistinguishable from
the noise. Catalog code M-19 names a threshold set above reality.

---

## 7. Detecting drift

**An application release can alter or invalidate the correlations that a projection is built
on** (Release It! Sec. 17.1, p. 269).

| The signal | The action | Source |
|---|---|---|
| An application release ships | Reexamine the correlations. Wait for an adequate body of new measurements before you judge | Sec. 17.1, p. 269 |
| A prediction is published | Attach a reference that names the set of projections used | Sec. 17.1, p. 269 |
| Four to six months pass | Recheck that the old correlations still hold true | Sec. 17.8, p. 307 |
| A popular hour loses popularity | Suspect that the system is too slow at that hour | Sec. 17.8, p. 308 |
| A driving variable reaches a plateau | Suspect a limiting factor, probably the responsiveness of the system | Sec. 17.8, p. 308 |
| A query plan changes, or a new query enters the expensive list | Suspect an accumulation of data somewhere | Sec. 17.8, p. 308 |
| A common query causes a table scan | Suspect a missing index | Sec. 17.8, p. 308 |

**Observe both trends and outliers. Both give insight** (Release It! Sec. 17.8, p. 307). A
prediction rests on a model, and a bad model gives a bad prediction (Sec. 17.1, p. 269).
Catalog code M-35 names a forecast with no model identity.

---

## 8. Condense the data

**For every mechanism that accumulates a resource, another mechanism must recycle that
resource** (Release It! Pattern 5.4, p. 124).

A large system accumulates a lot of data in this store. If you do not condense the data, data
collection eventually stops the system (Release It! Sec. 17.7, p. 304).

**How old is ancient?** Nygard states that the answer depends on your system. His own guidance
is that minute-by-minute samples older than a week are not helpful (p. 304). Keep that hedge.
The retention number for your system is a local decision.

---

## 9. The feedback loop

**Transparency only gives access to the data. A human in the loop must view and interpret it**
(Release It! Sec. 17.8, p. 305).

Nygard defines an effective feedback process as acting responsively to meaningful data. He names
two schools that describe the same loop. The Deming Cycle spirals through Plan-Do-Check-Act.
John Boyd's O-O-D-A is Observe, Orient, Decide, Act (Release It! Sec. 17.8, p. 305).

Every school reduces to these six steps (Release It! Sec. 17.8, p. 305):

1. Examine the current state, the historical patterns and the future projections.
2. Interpret the data. This always happens inside some person's mental model of the system.
3. Evaluate the potential actions. Include the cost of each one, and include no action at all.
4. Decide on a course of action.
5. Implement the chosen course of action.
6. Observe the new state of the system.

**Four rules from the O-O-D-A sidebar** (Release It! Sec. 17.8, p. 306–307).

- The loop requires correct observations. Wishful thinking and confirmation bias must not cloud
  them.
- Orientation updates a mental map from the previous map and the new observations.
- Political filtering of an observation is fatal. Spin is antithetical to O-O-D-A.
- The loop contains reinforcing feedback, so the system can go nonlinear and chaotic. Advantage
  comes from cycling faster than the opponent.

### 9.1 War story — the report that Outlook deleted

A reporting system generates a report and sends it to a distribution list. Half the people on
the list have a rule in Microsoft Outlook that deletes the report automatically. Nygard states
that the report is worse than useless. It costs effort to generate. Somebody must maintain it
through every change in the underlying system. It creates a false sense of security. Once in a
blue moon the report might show something serious, and by then every supposed consumer stopped
reading it long ago (p. 305). Catalog code M-37 names a signal with no reader.

---

## 10. The review cadences

Source: Release It! Sec. 17.8, p. 307–308. Nygard calls these the Keys to Observation. Build an
operational rhythm that makes improvement routine rather than occasional (p. 307).

| Cadence | What to review | What to search for |
|---|---|---|
| **Weekly** | The past week's problem tickets | Recurring problems. The problems that consume the most time. A subsystem that causes many problems. A development team that causes many problems. Problems related to one third party or integration point |
| **Monthly** | The total volume of problems and the distribution of problem types | A decrease in severity and a decrease in volume overall |
| **Daily or weekly** | Exceptions and stack traces in log files | The most common sources. Decide whether each one is a serious problem or a gap in error handling |
| **Ongoing** | Help desk calls | Common issues that point to a user interface improvement, or to a place where the system must tolerate more error |
| **Every four to six months** | The recorded correlations | Correlations that no longer hold true |
| **At least monthly** | Data volumes and query statistics | The most expensive queries. A changed query plan. A new entry on the expensive list. A table scan |
| **Monthly** | The daily and weekly envelope of demand and system metrics | A changing traffic pattern. A plateau in a driving variable |

**Expect a sawtooth in the monthly volume.** New code releases introduce new problems, so the
curve rises after a release (Release It! Sec. 17.8, p. 307). Search that release for the cause.

**When the ticket volume is too high to review completely, examine the top categories and also
sample tickets at random** (Release It! Sec. 17.8, p. 307).

**Name the reader and the cadence for every review.** A cadence with no named reader produces
the report that Outlook deletes.

### 10.1 Four questions for each metric under review

Source: Release It! Sec. 17.8, p. 308.

1. How does the metric compare to the historical norms? This is easy when the store holds
   enough data to form expectations.
2. If the metric continues its recent trend, what happens to the other correlated metrics?
3. How long could the trend continue? Which limiting factor will apply?
4. What results from that limiting factor?

**The worked example.** On a retail site, orders received and application server CPU utilization
are correlated to some degree. The beta between the two is the conversion rate (Release It!
Sec. 17.8, p. 308).

**The answer must motivate a decision.** Nygard names four. Do nothing. Add resources. Reduce
customer demand. Optimize the application code. Any of these decisions concludes the feedback
loop successfully (Release It! Sec. 17.8, p. 308–309).

---

## 11. Retire a signal, a report and a rule

**The focus of interest shifts over time.** In the early days the issues are mainly reactive.
As root causes are corrected, as new releases ship, and as traffic patterns change, the
emphasis moves from reactive analysis to predictive analysis (Release It! Sec. 17.8, p. 308).

| The test | The action | Source |
|---|---|---|
| The metric stopped producing useful information | Stop reviewing it | Release It! Sec. 17.8, p. 308 |
| The report is two years old and the system changed | Retire it. Nygard warns that it can be worthless or even misleading | Release It! Sec. 17.8, p. 308 |
| The rule is exercised less than once a quarter | Candidate for removal | SRE Ch. 6, p. 91 |
| The signal appears on no prebaked dashboard and in no alert rule | Candidate for removal | SRE Ch. 6, p. 91 |
| The emphasis moved from reactive to predictive | Stop reviewing some old things. Start reviewing new trends | Release It! Sec. 17.8, p. 308 |

**Unused collection and alerting configuration is pure maintenance cost** (SRE Ch. 6, p. 91).

---

## 12. Review questions for a threshold, a baseline or a cadence

Ask these seven questions. Each one maps to a code in `observability-gap-catalog.md`.

1. Does the store hold the history that this threshold was derived from? (M-19)
2. Does the threshold live outside the application source? (M-34)
3. Does the required event have an ExpectedTime expectation and an absence alarm? (M-04)
4. Can a failure of this store affect the primary function? (M-38)
5. Does the forecast name the release and the projection set it came from? (M-35)
6. Does this report, dashboard or signal have a named reader and a cadence? (M-37)
7. Is the expectation change recorded with a bug and an owner? (M-40)

**This file does not apply to a change that sets no threshold, no baseline and no cadence.** "Not
applicable" is a valid result. `slo-engineering` owns the reliability target itself.
`incident-response` owns the ticket review and the problem-volume review.

---

## 13. Related files

| File | Read it when |
|---|---|
| `01-the-four-golden-signals.md` | You set the utilization target or the nominal range for one of the four golden signals |
| `02-symptom-based-alerting.md` | The expectation you set will route a condition to a pager |
| `03-white-box-and-black-box.md` | The observation comes from a probe rather than from the process |
| `04-time-series-and-rules.md` | You write the alert rule that an expectation violation triggers |
| `05-transparency-and-logging.md` | You expose the counters and the state transitions that feed this store |
| `observability-gap-catalog.md` | You review a change, or you review an incident that the monitoring did not detect |
| `capacity-engineering/references/05-intent-based-capacity-planning.md` | You use the stored demand metrics and system metrics for a capacity forecast |
| `incident-response/references/05-tracking-outages.md` | You run the weekly ticket review or the monthly problem-volume review |
