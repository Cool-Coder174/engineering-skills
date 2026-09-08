# Indicators, Objectives and Agreements

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 4 "Service Level Objectives", pp. 62–73. Service Level Terminology, pp. 62–65.
Indicators in Practice, pp. 65–69. Objectives in Practice, pp. 69–72. Agreements in
Practice, p. 73. Supporting pages: SRE Ch. 3, pp. 50–51. SRE Ch. 6, p. 88. SRE App. B,
p. 572. Release It! Sec. 13.2, pp. 230–232.

Read this file when you choose an indicator, when you write an objective, or when a document
uses the word SLA. This file owns the **definition**. `01-risk-and-the-error-budget.md` owns
the budget and the release gate. `03-availability-math.md` owns the arithmetic.

---

## 1. The three terms

The book separates three meanings because common use overloads the word SLA
(SRE Ch. 4, p. 62).

| Term | What it is | Who states it | The test |
|---|---|---|---|
| Indicator (SLI) | A defined quantitative measure of one aspect of the service level (SRE Ch. 4, p. 62) | The engineer | Can you plot it today, from real data? |
| Objective (SLO) | A target value, or a range of values, for a level that an indicator measures (SRE Ch. 4, p. 63) | The product owner and the engineer together | Does a number exist, and a window? |
| Agreement (SLA) | An explicit or implicit contract that includes the consequences of meeting or missing the objectives inside it (SRE Ch. 4, p. 65) | The business and the legal team | Ask what happens on a miss. A named consequence must exist. |

**The test that separates an objective from an agreement is one question.** The book asks
"what happens if the SLOs aren't met?" (SRE Ch. 4, p. 65). No explicit consequence means you
hold an objective, not an agreement (SRE Ch. 4, p. 65).

**Confusing the two terms is expensive in three ways.**

1. A team that believes it holds a contract plans no response to a breach. The book records
   that most people mean SLO when they say SLA. A person who reports an "SLA violation"
   almost always reports a missed objective (SRE Ch. 4, p. 73, footnote 16).
2. A real agreement breach can start a court case for breach of contract
   (SRE Ch. 4, p. 73, footnote 16).
3. An agreement is hard to change. The broader the constituency, the harder it is to change
   or to delete an agreement that proves unwise (SRE Ch. 4, p. 73).

**An objective is worth holding even with no agreement** (SRE Ch. 4, p. 65). Section 8 gives
the Google Search case.

**Signal that you must apply the test:** a requirement document, a vendor page, or a ticket
uses the word SLA. Apply the question. Then rename the document, or write the consequence.
Catalog code L-38.

---

## 2. Choose the indicator the user feels

**Do not use every metric the monitoring system tracks.** A handful of representative
indicators is enough to evaluate and to reason about system health. Section 10 gives the cost
of too many and of too few (SRE Ch. 4, p. 66).

### 2.1 Indicators by system category — SRE Ch. 4, p. 66

| System category | Indicators the book names | The question each one answers |
|---|---|---|
| User-facing serving system | Availability, latency, throughput | Could we answer? How long did it take? How many requests could we handle? |
| Storage system | Latency, availability, durability | How long to read or write? Can we reach the data? Is the data still there? |
| Big data system or pipeline | Throughput, end-to-end latency, per-stage latency | How much data moves? How long from ingestion to completion? |
| Every system | Correctness | Was the right answer returned, and the right analysis done? |

**Correctness is a property of the data, not of the infrastructure.** The book states that
correctness is usually not the reliability team's duty to meet. Track it as an indicator of
system health (SRE Ch. 4, p. 66). Catalog code L-07.

**Related.** The four golden signals of monitoring are a separate list from the indicator list
above (SRE Ch. 6, p. 88). The `observability` skill owns them, in
`observability/references/01-the-four-golden-signals.md`. This file owns the objective
that a signal is compared against.

### 2.2 Where you measure changes what you see

**Sources:** SRE Ch. 4, p. 63, p. 67. SRE App. B, p. 572. Release It! Sec. 13.2, p. 231.

| Measurement point | What it sees | What it cannot see | Use it when |
|---|---|---|---|
| The server | Server error rate and server latency | The page JavaScript, the network path, the client library | You need per-instance attribution |
| The client | The time for a page to become usable in the browser | Nothing the user feels | The objective is user-facing |
| A synthetic transaction | One named feature, from named locations | The real traffic mix | Traffic is low, or a per-feature agreement exists |
| The vendor status page | The vendor's own server side | Your path to the vendor | Never as the indicator. Use it only to corroborate. |

**The ideal indicator measures the service level directly.** Sometimes only a proxy is
available, because the measure you want is hard to obtain or to interpret. Client-side latency
is often the more user-relevant metric. Server latency is often the only latency you can
measure (SRE Ch. 4, p. 63).

**A proxy hides a class of fault.** Attention to the backend response latency can miss poor
user latency that the page JavaScript causes (SRE Ch. 4, p. 67). App. B repeats the rule under
the heading "Define SLOs Like a User" (SRE App. B, p. 572). Catalog code L-06.

**Signal to move the measurement point:** users report that the service is slow, and the
server dashboard shows no change. Instrument the client.

### 2.3 One quantity you cannot set an objective on

**You do not always choose the value.** The queries per second of incoming external requests
is determined by the desires of your users. You cannot set an objective for it
(SRE Ch. 4, p. 63). State throughput as a capacity number instead. Keep the objective on
latency, on availability, or on correctness. Catalog code L-10.

**Two indicators can couple behind the scenes.** Higher QPS often leads to larger latencies.
The book states that it is common for a service to have a performance cliff beyond some load
threshold (SRE Ch. 4, p. 64).

### 2.4 Work from the objective backward

**Start from what your users care about, not from what you can measure.** What users care
about is often hard or impossible to measure, so you approximate the need. A start from what
is easy to measure produces less useful objectives. The book reports that work from the
desired objective backward to the indicator is sometimes the better order (SRE Ch. 4, p. 69).

---

## 3. Aggregation, distributions and percentiles

**Aggregation needs care.** The monitoring system collects raw measurements over a window. It
then converts them into a rate, an average, or a percentile (SRE Ch. 4, p. 63, p. 67).

### 3.1 The window is part of the metric

**A metric with no stated window is not defined.** Requests per second looks simple. It
still hides a choice. The measurement can happen once a second, or it can average requests
over a minute. The minute average hides a burst that lasts a few seconds
(SRE Ch. 4, p. 67). Catalog code L-12.

### 3.2 Use the distribution, not the mean

**Treat a metric as a distribution.** The book states the rule once. "Most metrics are better
thought of as distributions rather than averages." (SRE Ch. 4, p. 67). An average of request
latencies obscures the tail. Most requests can run fast while a long tail of requests runs
much slower. Catalog code L-11.

**Percentiles let you read the shape of the distribution.** A high-order percentile, such as
the 99th or the 99.9th, shows a plausible worst-case value. The 50th percentile, the median,
emphasizes the typical case (SRE Ch. 4, p. 68).

**Variance matters more than the median at high load.** The higher the variance in response
times, the more the typical user experience feels the long tail. Queuing effects make this
worse at high load. User studies have shown that people typically prefer a slightly slower
system to one with high variance in response time. Some reliability teams therefore watch
only the high percentile values (SRE Ch. 4, p. 68).

### 3.3 Four statistical traps — SRE Ch. 4, pp. 68–69

| Trap | Why the data breaks the assumption | Required action |
|---|---|---|
| The mean equals the median | Computing data is often skewed. No request can answer in less than 0 ms. | Report both. Do not substitute one for the other. |
| The distribution is normal | The book states that it tries not to assume normality without verification first. | Verify the distribution before you use an approximation. |
| The tail is complete (L-13) | A timeout at 1,000 ms means no successful response can exceed 1,000 ms. | Record the timeout beside the latency indicator. Report a timeout as a failure. |
| An outlier threshold is safe to automate (L-15) | The book gives the example of a process that restarts a server with high request latencies. | Verify the distribution first. The action can fire too often, or not often enough. |

---

## 4. Standardize indicators — SRE Ch. 4, p. 69

**Standardize on common definitions so nobody reasons from first principles each time.** You
can then omit any feature that conforms to a standard template from the specification of an
individual indicator (SRE Ch. 4, p. 69).

| Template dimension | The book's example value |
|---|---|
| Aggregation interval | Averaged over 1 minute |
| Aggregation region | All the tasks in a cluster |
| Measurement frequency | Every 10 seconds |
| Which requests are included | HTTP GETs from black-box monitoring jobs |
| How the data is acquired | Through our monitoring, measured at the server |
| Data-access latency | Time to last byte |

**Build a set of reusable templates for each common metric.** Templates save effort. They
also make it simpler for everyone to understand a specific indicator (SRE Ch. 4, p. 69).

**This skill adds one field that the template does not carry.** Record the timeout in force
beside the latency indicator, because the timeout censors the distribution
(SRE Ch. 4, p. 68). The added field is a local decision, not a book rule.

---

## 5. Objectives in practice

### 5.1 The form of an objective

**An objective has one of two structures.** Write `SLI ≤ target`. Or write
`lower bound ≤ SLI ≤ upper bound` (SRE Ch. 4, p. 63). **An objective must also state how it
is measured and when it is valid** (SRE Ch. 4, p. 69). The table gives the book's own
examples (SRE Ch. 4, p. 70).

| Case | The book's written objective |
|---|---|
| One target, defaults stated | 99% (averaged over 1 minute) of Get RPC calls will complete in less than 100 ms (measured across all the backend servers) |
| The same target, template applied | 99% of Get RPC calls will complete in less than 100 ms |
| The shape of the curve matters | 90% of Get RPC calls in less than 1 ms. 99% in less than 10 ms. 99.9% in less than 100 ms. |
| The workloads differ | 95% of throughput clients' Set RPC calls in under 1 s. 99% of latency clients' Set RPC calls with payloads under 1 kB in under 10 ms. |

**Split the objective when the workloads differ.** A bulk pipeline cares about throughput. An
interactive client cares about latency (SRE Ch. 4, p. 70).

### 5.2 One hundred percent is not the target

**The book calls a 100% objective both unrealistic and undesirable.** It can reduce the rate
of innovation and deployment. It can require expensive and overly conservative solutions. It
can do both (SRE Ch. 4, p. 70).

**Allow an error budget instead.** The error budget is the rate at which the objectives can
be missed. The book calls it an objective for meeting other objectives (SRE Ch. 4, p. 70).

**Track the budget daily or weekly.** Report to upper management monthly or quarterly
(SRE Ch. 4, p. 70. SRE Ch. 3, p. 51). Compare the violation rate against the budget. The gap
is the input to the process that decides when to release (SRE Ch. 4, p. 70).

### 5.3 Choosing the target — SRE Ch. 4, p. 71

| Lesson | What it forbids | What it requires |
|---|---|---|
| Do not pick a target based on current performance | Adopting today's measurement as the objective | Derive the number from the user and from cost |
| Keep it simple | A complicated aggregation inside the indicator | An indicator a person can reason about |
| Avoid absolutes | "Infinitely" scalable, "always" available | A number, a window, and a formula |
| Have as few objectives as possible | An objective nobody can quote | Enough coverage of the system attributes, and no more |
| Perfection can wait | An overly strict target you must relax later | A loose target you tighten as you learn |

**A target copied from current performance can lock the team into heroic work.** The system
then cannot improve without significant redesign. Catalog code L-02. **An absolute requirement
takes a long time to design and costs a lot to operate.** It also probably gives more than
users would be happy to have (SRE Ch. 4, p. 71). Catalog code L-01.

**Not every product attribute fits an objective.** The book states that it is hard to specify
user delight with an objective (SRE Ch. 4, p. 71).

**The objective is a lever in both directions.** A good objective is a legitimate forcing
function for a development team. An overly aggressive objective wastes work on heroic effort.
An objective that is too lax produces a bad product (SRE Ch. 4, p. 72).

---

## 6. Control measures

**Indicators and objectives are elements of a control loop** (SRE Ch. 4, p. 72). The four
steps are these.

1. Monitor and measure the indicators of the system.
2. Compare the indicators to the objectives. Decide whether action is needed.
3. If action is needed, determine what must happen to meet the target.
4. Take that action.

**The book's worked example runs the loop once.** Step 2 shows that request latency is
rising and will miss the objective in a few hours. Step 3 tests the hypothesis that the
servers are CPU-bound. The team decides to add more servers to spread the load
(SRE Ch. 4, p. 72).

**Without the objective, the loop has no trigger.** The book states it plainly. "Without the
SLO, you wouldn't know whether (or when) to take action." (SRE Ch. 4, p. 72).

**Signal for step 3:** the indicator is still inside the objective, and the trend predicts a
miss inside the window. Act on the trend, not only on the breach.

---

## 7. Objectives set expectations

**Publish the objective.** A published objective sets expectations about how the service
performs. It reduces unfounded complaints, such as a report that the service is slow
(SRE Ch. 4, p. 64).

**A missing objective produces two opposite faults** (SRE Ch. 4, p. 64).

- **Over-reliance.** Users believe the service is more available than it is.
- **Under-reliance.** Prospective users believe the system is flakier than it is, and they
  avoid it. Catalog code L-36.

**A published objective also helps a user choose.** The book's example is a team that wants
to build a photo-sharing website. That team may avoid a service that promises very strong
durability and low cost in exchange for slightly lower availability. The same service can
fit an archival records system well (SRE Ch. 4, p. 72).

**Two tactics keep the expectation realistic** (SRE Ch. 4, pp. 72–73).

| Tactic | Mechanism | What it buys |
|---|---|---|
| Keep a safety margin | Hold a tighter internal objective than the advertised one | Room to respond to a chronic problem before it becomes visible outside. Room for a reimplementation that trades performance for cost or for maintenance. |
| Do not overachieve | Create a planned outage, throttle some requests, or design the system so it is not faster under light load | No hidden dependency on performance you never promised |

**Users act on measured behavior, not on the published number.** The book states it once.
"Users build on the reality of what you offer, rather than what you say you'll supply"
(SRE Ch. 4, p. 73). Catalog codes L-35 and L-37.

**Signal that you overachieve:** measured availability sits far above the objective across
several quarters, and dependent teams have no fallback path.

---

## 8. War stories from the chapter

**The Global Chubby planned outage (SRE Ch. 4, pp. 64–65).** Chubby is Google's lock service
for loosely coupled distributed systems. The global instance places each replica in a
different geographic region. Over time the team found that failures of the global instance
consistently produced service outages, and many of those outages reached end users. True
global Chubby outages were so infrequent that service owners added dependencies on the
assumption that Chubby would never fail. Its high reliability gave a false sense of security,
because those services could not function when Chubby was unavailable. The fix was to make
global Chubby meet its objective and not significantly exceed it. In any quarter where a true
failure has not dropped availability below the target, the team synthesizes a controlled
outage. This reveals an unreasonable dependency shortly after somebody adds it.

**Google Search has no public agreement (SRE Ch. 4, p. 65).** Google wants everyone to use
Search fluidly and efficiently, and it has not signed a contract with the whole world. Search
therefore carries no public agreement. Consequences still exist when Search is unavailable.
The book names a hit to reputation and a drop in advertising revenue. Other Google services,
such as Google for Work, do carry explicit agreements with their users. The lesson is that
indicators and objectives are worth defining and managing whether or not an agreement exists.

**Two averages that lie (SRE Ch. 4, pp. 67–68).** Figure 4-1 plots the 50th, 85th, 95th, and
99th percentile latencies for one system. A typical request is served in about 50 ms. Five
percent of requests run 20 times slower. An alert on the average latency alone would show no
change across the day. The tail latency, the topmost line, changes significantly across that
same day. The second case is the alternating second. One system serves 200 requests a
second in even-numbered seconds and 0 requests in the odd-numbered seconds. It carries the
same average load as a system that serves a constant 100 requests a second, and twice the
instantaneous load. The average hides the part of the behavior the user feels, and the
aggregation window belongs inside the definition of the metric.

---

## 9. Agreements and their consequences

**An agreement adds a consequence to an objective.** The consequence is easiest to recognize
when it is financial, such as a rebate or a penalty. It can take other forms
(SRE Ch. 4, p. 65). Business and legal teams pick the consequence and the penalty. The
reliability team helps those teams understand the difficulty of meeting the objectives inside
the agreement (SRE Ch. 4, p. 73).

**The reliability team does two things for an agreement.** It helps the business avoid the
consequences of a missed objective. It also helps define the indicators, because an objective
way to measure the objectives must exist, or disagreements arise (SRE Ch. 4, p. 65).

**Be conservative in what you advertise.** Much of the advice on objective construction also
applies to an agreement (SRE Ch. 4, p. 73).

**Define the agreement per feature, not for the whole system.** Nygard requires a definition
per feature or per business process. A single line such as a 99.9% availability claim for
"the system" leaves vagueness behind every word. The first incident then turns into a
blamestorm a year later (Release It! Sec. 13.2, pp. 230–232). Catalog codes L-03 and L-04.
`05-gathering-availability-requirements.md` holds the eight questions that define "down".

**Signal that an agreement is not yet an agreement:** the document names a percentage and
names no consequence, no exclusion, and no measurement method.

---

## 10. How many indicators, and how many objectives

**Fewer is better, and the book gives the test.** Choose just enough objectives to cover the
attributes of the system. Defend each one you pick. If you can never win a conversation about
priorities by quoting a particular objective, it is probably not worth having that objective
(SRE Ch. 4, p. 71). Catalog code L-05.

**Both directions carry a cost at the indicator level too.** Too many indicators make it hard
to give the right level of attention to the ones that matter. Too few leave significant
behavior of the system unexamined (SRE Ch. 4, p. 66).

**A complicated indicator defeats its own purpose.** A complicated aggregation obscures
changes to system performance. It is also harder to reason about (SRE Ch. 4, p. 71).

**A footnote carries the stronger form.** If you can never win a conversation about
objectives, it is probably not worth having a reliability team for that product
(SRE Ch. 4, p. 71, footnote 17).

---

## 11. Related files

This file names 16 catalog codes as it goes — L-01 to L-07, L-10 to L-13, L-15, and L-35 to
L-38. `slo-defect-catalog.md` holds the full entry for each one.

| File | What it owns |
|---|---|
| `01-risk-and-the-error-budget.md` | The error budget, the release gate, and the risk conversation |
| `03-availability-math.md` | Percentage to minutes, the aggregate formula, and the dependency ceiling |
| `04-the-toil-budget.md` | Toil, the 50% cap, and on-call load |
| `05-gathering-availability-requirements.md` | The per-feature requirement and the eight variables that define "down" |
| `slo-defect-catalog.md` | Every `L-` code, with signature, consequence, and fix |
| `../SKILL.md` | The SLO Record and the defect gate |
