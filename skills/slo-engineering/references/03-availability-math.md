# Availability Arithmetic

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 3, pp. 48–61, Ch. 4, pp. 62–73, App. A, pp. 567–570, App. B, pp. 572–574.
*Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007), Sec. 4.10, pp. 102–105,
Sec. 13.1, pp. 229–230, Sec. 13.2, pp. 230–232.

`system-design` carries a nines table in
`system-design/references/02-estimation.md`. That skill uses the table to size
capacity. This file uses the same arithmetic for budget accounting. It answers four
questions. How many minutes does the objective allow? How many minutes remain? What ceiling
do the dependencies set? What number makes an alert fire?

Read this file when a person states a percentage. Read it again before you promise that
percentage to anybody.

---

## 1. Choose the formula before you choose the number

**A percentage means nothing until you name the formula.** Two services can report 99.9%
from the same month of data and describe different realities.

| Situation | Formula | Source |
|---|---|---|
| One instance. It is wholly up or wholly down. | uptime ÷ total time | SRE Ch. 3, p. 50 |
| Many replicas, or a load that varies by hour | successful requests ÷ all requests, over a rolling window | SRE Ch. 3, p. 50 |
| A batch job, a pipeline, or a queue consumer | records processed successfully ÷ records received | SRE Ch. 3, p. 51 |

**Signal that the time formula is wrong.** The service runs several replicas, and only some
of them fail. Or the load varies across the day or the week (SRE App. A, p. 570). In both
cases the time formula hides a partial outage. Use the aggregate formula. The defect code is
L-14 in `slo-defect-catalog.md`.

**Google does not use the time formula for its own serving systems.** A globally
distributed service serves some traffic somewhere at any moment. It is at least partly up at
all times (SRE Ch. 3, p. 50).

**Worked number.** A system serves 2.5M requests in a day. The daily target is 99.99%. The
system may serve up to 250 errors and still meet the target (SRE Ch. 3, p. 50).

---

## 2. The availability table

The table gives the allowed unavailability window for each level. It assumes no planned
downtime (SRE App. A, p. 567).

| Availability | Per year | Per quarter | Per month | Per week | Per day | Per hour |
|---|---|---|---|---|---|---|
| 90% | 36.5 days | 9 days | 3 days | 16.8 hours | 2.4 hours | 6 minutes |
| 95% | 18.25 days | 4.5 days | 1.5 days | 8.4 hours | 1.2 hours | 3 minutes |
| 99% | 3.65 days | 21.6 hours | 7.2 hours | 1.68 hours | 14.4 minutes | 36 seconds |
| 99.5% | 1.83 days | 10.8 hours | 3.6 hours | 50.4 minutes | 7.20 minutes | 18 seconds |
| 99.9% | 8.76 hours | 2.16 hours | 43.2 minutes | 10.1 minutes | 1.44 minutes | 3.6 seconds |
| 99.95% | 4.38 hours | 1.08 hours | 21.6 minutes | 5.04 minutes | 43.2 seconds | 1.8 seconds |
| 99.99% | 52.6 minutes | 12.96 minutes | 4.32 minutes | 60.5 seconds | 8.64 seconds | 0.36 seconds |
| 99.999% | 5.26 minutes | 1.30 minutes | 25.9 seconds | 6.05 seconds | 0.87 seconds | 0.04 seconds |

Source for every cell: SRE App. A, pp. 567–570.

**Each nine is one order of magnitude.** Each extra nine moves the service one order of
magnitude closer to 100% (SRE Ch. 3, p. 50).

**Quote the row for the window you can observe.** A yearly figure hides a week that failed
completely. Report the number for the window the budget resets on. Track it more often than
that. See Section 4.

**One deploy can exceed a whole month at four nines.** 99.99% permits 4.32 minutes a month
(SRE App. A, p. 569). Nygard states the same level as slightly more than four minutes of
downtime a month (Release It! Sec. 4.10, p. 102). The two books agree.

**The book rounds the year cell two ways.** Ch. 3 gives 52.56 minutes a year at 99.99%
(SRE Ch. 3, p. 50). App. A gives 52.6 minutes (SRE App. A, p. 569). Pick one form and use it
everywhere in your record.

**Planned downtime sits outside this table.** An outage that is occasional, regular and
scheduled counts as planned downtime, not unplanned downtime (SRE Ch. 3, p. 53). Do not charge
it to the budget. An outage the team labels "planned" after it ends is defect L-44.

---

## 3. The error budget in minutes

**The budget is one minus the objective** (SRE App. B, p. 573). A 99.99% target leaves a
0.01% unavailability budget. The usual period is a month.

Convert the objective in four steps.

1. Write the objective and its window.
2. Subtract the objective from 100%. That fraction is the budget.
3. Read the minutes for that window from the table in Section 2.
4. Multiply the fraction by the request count for the window. That gives the budget in
   events.

| Objective | Budget fraction | Budget per month | Budget per quarter |
|---|---|---|---|
| 99% | 1% | 7.2 hours | 21.6 hours |
| 99.5% | 0.5% | 3.6 hours | 10.8 hours |
| 99.9% | 0.1% | 43.2 minutes | 2.16 hours |
| 99.95% | 0.05% | 21.6 minutes | 1.08 hours |
| 99.99% | 0.01% | 4.32 minutes | 12.96 minutes |
| 99.999% | 0.001% | 25.9 seconds | 1.30 minutes |

Minutes and seconds come from SRE App. A, pp. 567–570. The fraction comes from
SRE App. B, p. 573.

**State the budget in minutes and in events.** Minutes drive the conversation with the
business. Events drive the alert. An objective with neither is defect L-16.

**Worked spend.** A service targets 99.999% of queries per quarter. The budget is a 0.001%
failure rate. One problem fails 0.0002% of the expected queries. That problem spends 20% of
the quarterly budget (SRE Ch. 3, p. 59).

**Count every failure the user saw.** A network outage or a datacenter fault also spends the
budget. The number of pushes left in the quarter falls, even though no code change caused
the loss (SRE Ch. 3, p. 60). The defect code is L-19.

**A neutral party measures the spend.** The monitoring system measures the actual uptime
(SRE Ch. 3, p. 59). The deploy tool reads that number. It does not produce it. The defect
code is L-17.

---

## 4. The reset period and the tracking period

| Objective | Reset period | Reason | Source |
|---|---|---|---|
| At or below 99.99% | Month | The monthly allowance is large enough to manage | SRE App. B, p. 573 |
| Above 99.99% | Quarter | The monthly allowance is too small to manage | SRE App. B, p. 573 |

**Reset and tracking are different periods.** Google sets quarterly availability targets. It
tracks performance weekly, or even daily (SRE Ch. 3, p. 51). Track the budget daily or
weekly. Give management a monthly or quarterly assessment (SRE Ch. 4, p. 70).

**Signal for defect L-20.** The objective is above 99.99% and the policy resets monthly.
99.999% permits 25.9 seconds a month (SRE App. A, p. 570). One event exhausts the month, and
the policy then guides nothing.

---

## 5. Serial dependency multiplication

> `self-healing-apis`, in `self-healing-apis/references/09-vendor-slas-and-degradation.md`,
> owns the SLA-inversion antipattern in full — the dependency inventory, the three failure
> layers per dependency, the Frammitz worked case, and the design of the degraded mode. This
> section keeps only the arithmetic that converts that inventory into a number, because a
> budget cannot be written without it. Read the other file before you build the table.

**A serial chain multiplies. It does not average.** Nygard gives the formula. The
probability that the feature is up equals `(1 − P(internal failure))` multiplied by the
availability of every dependency (Release It! Sec. 4.10, p. 103). A single failure in one
dependency fails the whole feature.

**A chain does not keep the nines of its members.** The product falls with every dependency
you add.

| Dependencies, each at 99.9% | Product | Ceiling |
|---|---|---|
| 1 | 0.999 | 99.9% |
| 2 | 0.999² | 99.8% |
| 3 | 0.999³ | 99.7% |
| 5 | 0.999⁵ | **99.5%** |
| 9 | 0.999⁹ | 99.1% |
| 10 | 0.999¹⁰ | 99.0% |

The five-dependency row is the book's own worked number (Release It! Sec. 4.10, p. 103). The
other rows apply the same formula from that page. Nine dependencies at three nines give about
99.1%. That is two nines, not three.

**Mixed levels multiply the same way.** Two dependencies at 99.9% and one at 99% give
0.999 × 0.999 × 0.99, which is about 98.8%.

**The ceiling rule.** Nygard states it in one line: "The best you can possibly do is the SLA
of the worst of your service providers" (Release It! Sec. 4.10, p. 103).

**Count layers, not vendors.** Every dependency exposes three layers that fail
independently (Release It! Sec. 4.10, p. 103). A count of vendors understates the count of
failure points. `self-healing-apis/references/09-vendor-slas-and-degradation.md` names
the layers and holds the inventory method.

**A dependency with no stated availability blocks the arithmetic.** Project Frammitz depends
on two parties that offer no SLA. Strictly, Frammitz cannot offer an availability SLA at all
(Release It! Sec. 4.10, p. 103). Write "No SLA" in the cell. Do not write 100%. The defect
code is L-28.

**The lower bound and the upper bound.** A feature that is perfectly decoupled from every
external system carries only `P(internal failure)`. A feature that fails whenever any
dependency fails carries the full product. Most systems fall between the two
(Release It! Sec. 4.10, p. 104).

**Service levels only decrease when you call a third party** (Release It! Sec. 4.10, p. 104,
p. 105).

---

## 6. SLA inversion arithmetic

**SLA inversion is a stated objective above the computed ceiling.** Nygard defines it as a
system that must meet a high-availability SLA and depends on systems of lower availability
(Release It! Sec. 4.10, p. 104). The defect code is L-26. It blocks the merge.

Run the arithmetic in six steps. Steps 1 to 3 belong to
`self-healing-apis/references/09-vendor-slas-and-degradation.md`. Take the completed
table from that file and start at step 4.

1. List every dependency of the feature. Include the corporate DNS cluster, the mail
   transfer service, message queues and brokers, and the enterprise SAN
   (Release It! Sec. 4.10, p. 105).
2. Record the stated availability of each one. Write "No SLA" where none exists.
3. Note the three layers per dependency from Section 5.
4. Estimate `P(internal failure)` for your own code and your own hardware.
5. Multiply. Compare the product against the objective you stated.
6. If the objective is higher than the product, apply a remedy from the table below.

| Remedy | What the arithmetic gains | Source |
|---|---|---|
| Decouple with middleware, and degrade the feature | Removes the dependency from the serial product | Release It! Sec. 4.10, p. 104 |
| One Circuit Breaker per dependency | Limits the effect of a slow or refused dependency | Release It! Sec. 4.10, p. 104 |
| Restate the objective per feature | Moves the number to the level the arithmetic supports | Release It! Sec. 4.10, p. 105 |
| Write an exclusion for external systems | Removes the dependency from the agreement text | Release It! Ch. 15, p. 250 |

The first two remedies are mechanisms, and `self-healing-apis` owns both. Only the third and
the fourth belong to this skill. They change the number and the agreement text, not the code.

**A feature with no external dependency can carry your maximum objective.** A feature that
calls a third party can carry only the third party's level, reduced by your own failure
probability (Release It! Sec. 4.10, p. 105). This is why one objective for "the system" is
defect L-03.

**The certainty rule.** If the feature fails whenever a dependency fails, its availability is
always lower than that dependency's availability. Nygard calls that a mathematical certainty
(Release It! Sec. 4.10, p. 105).

`self-healing-apis` owns the Circuit Breaker and the degradation path. This file owns the
number that tells you the breaker is required.

---

## 7. From objective to alerting threshold

The books give a control loop, not a threshold formula. Follow the loop
(SRE Ch. 4, p. 72).

1. Measure the indicators.
2. Compare the indicators against the objective. Decide whether action is needed.
3. Determine the action that meets the target.
4. Take the action.

**The threshold is a projection, not the objective itself.** The book's own example alerts
because latency is rising and will miss the objective in a few hours unless somebody acts
(SRE Ch. 4, p. 72). A threshold set at the objective fires only after the loss.

Three values must exist before a threshold means anything.

| Value | Why the threshold needs it | Source |
|---|---|---|
| The six template dimensions | Interval, region, frequency, requests included, data source, and latency point | SRE Ch. 4, p. 69 |
| The timeout in force | The timeout censors the distribution at its own value. No success can exceed it. | SRE Ch. 4, p. 68 |
| The measured distribution | Do not assume the mean equals the median. Do not assume a normal distribution. | SRE Ch. 4, pp. 68–69 |

**An indicator with no aggregation window is not a threshold.** One system serves 200
requests a second in even seconds and none in odd seconds. A second system serves a constant
100 a second. Both report the same one-minute average. The instantaneous load of the first
system is double (SRE Ch. 4, p. 67).

**Use percentiles for latency.** The 99th and 99.9th percentiles show a plausible worst case.
The 50th shows the typical case. The mean hides both (SRE Ch. 4, p. 68). The defect code is
L-11.

**Decide the output type with the threshold.** Monitoring has three output types, and they are
a page, a ticket, and a log line (SRE App. B, pp. 573–574). `observability`, in
`observability/references/02-symptom-based-alerting.md`, owns the three outputs, the
routing, and the rule that email is not one of them. Choose the output at the moment you set
the threshold, because a threshold with no output is not an alert.

---

## 8. Burn rate — **Modern**

**Neither book describes a burn-rate alert.** SRE gives the budget and the freeze rule
(SRE Ch. 3, p. 60. SRE App. B, p. 573). Nygard gives the dependency ceiling. An alert on the
burn rate across two windows is later practice. The defect code is L-39, and it carries the
tag **Modern**.

**Definition.** The burn rate is the fraction of the budget spent divided by the fraction of
the window elapsed.

- A burn rate of 1 spends the whole budget exactly at the end of the window.
- A burn rate of 2 spends the whole budget at the halfway point.
- At a constant rate `r`, the budget reaches zero after `1 ÷ r` of the window.

**Read the burn rate against the book's spend example.** One problem spends 20% of a
quarterly budget (SRE Ch. 3, p. 59). The same 20% is routine in month three and severe on
day two. Only the rate separates the two cases.

| Window | What it catches | What it misses | Threshold |
|---|---|---|---|
| Fast | A short and severe spike | A slow drain below the trigger value | A local decision. Neither book states a number. |
| Slow | A slow drain across days | A spike that ends before the window fills | A local decision. Neither book states a number. |

**Use one fast window and one slow window.** A single window misses one of the two shapes.
**Modern**.

**The burn rate does not replace the budget.** It replaces the wait. The budget still decides
the release. The policy ladder sits in `01-risk-and-the-error-budget.md`.

---

## 9. The worked cases in the books

**Project Frammitz (Release It! Sec. 4.10, pp. 102–105).** A new website carried a 99.99%
availability requirement. The team built redundancy at every level and used a shared-nothing
architecture. The dependency levels then ran from 99.999% down to 98.5%, and two dependencies
offered no SLA at all. Nygard's verdict is that Frammitz met the target only through sheer
luck. Strictly, it could not offer an availability SLA. Careful engineering inside the box
does not raise a ceiling that sits outside it.
`self-healing-apis/references/09-vendor-slas-and-degradation.md` holds the full
dependency table.

**The 98% website (Release It! Sec. 13.1, pp. 229–230).** Nygard prices the decision in
money instead of nines. 98% availability is 864 minutes of downtime a month. At $1,500 an
hour of peak revenue, that is about $21,600 a month of loss. 99.99% reduces the loss to $108
a month, a gain of $21,492 a month. Each nine costs about ten times more to implement and
about twice as much to operate each year. In this case the added lifecycle cost over five
years was $98,700, against savings of $1,289,520. The arithmetic decided the target. Nobody
had to argue about the word "highly".

**The 99.999% quarter (SRE Ch. 3, p. 59).** Product management sets a 99.999% quarterly
objective. The budget is a 0.001% failure rate for the quarter. The monitoring system, and
not the shipping team, measures the actual number. A single problem fails 0.0002% of the
expected queries and spends 20% of the quarter's budget. The remaining budget then decides
how many releases the product team can push. The number replaces the negotiation.

**Google Apps for Work (SRE Ch. 3, pp. 52–53).** A typical service advertises a quarterly
availability target of 99.9% to customers. Behind that number the team holds a stronger
internal target. The contract attaches penalties to the external number. The gap between the two
numbers is the room the team needs to respond to a chronic problem before a customer sees it.
An internal target equal to the advertised target is defect L-37.

---

## 10. What this file does not own

| Question | Where it is answered |
|---|---|
| Which indicator, and where do we measure it? | `02-slis-slos-and-slas.md` |
| Who stops the release, and at which budget state? | `01-risk-and-the-error-budget.md` |
| How do we write the requirement and define "down"? | `05-gathering-availability-requirements.md` |
| How much operational work does the target create? | `04-the-toil-budget.md` |
| Which defect code applies, and how severe is it? | `slo-defect-catalog.md` |
| How many servers does the load need? | `system-design/references/02-estimation.md` |
| Which dependencies belong in the table, and how does the Circuit Breaker work? | `self-healing-apis/references/09-vendor-slas-and-degradation.md` |
| How does the revert or the staged rollout work? | The `reverse-branching` skill |
| Which output does the alert take, and where does it route? | `observability/references/02-symptom-based-alerting.md` |

**Do not compute a ceiling for a change with no dependency.** A change that adds no
user-visible path and no dependency needs no arithmetic from this file. Write one line:
`SLO gate: not applicable — [reason].` The availability target is a business decision, not a
technical one (SRE Ch. 1, pp. 26–27). Where the business has not made that decision, say so
and stop. Do not invent a number.
