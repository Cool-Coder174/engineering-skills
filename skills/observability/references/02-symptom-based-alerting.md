# Symptoms, Causes and What May Page

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 6 "Monitoring Distributed Systems", p. 82–96. Ch. 10 "Practical Alerting from Time-Series
Data", p. 141, p. 149–157. Ch. 11 "Being On-Call", p. 160–167. App. B "A Collection of Best
Practices for Production Services", p. 573–576. App. F "Example Production Meeting Minutes",
p. 589. Two war stories come from *Release It!* — Michael T. Nygard (Pragmatic Bookshelf,
2007), Sec. 16.8, p. 261, and Sec. 17.7, p. 304.

A monitoring system answers two questions. What is broken? Why? The answer to "what" is the
symptom. The answer to "why" is a cause, and that cause can be an intermediate one
(SRE Ch. 6, p. 86).

**Page on symptoms. Keep cause rules as aids to debugging.** Google states this as the shape
of a healthy alerting pipeline (SRE Ch. 6, p. 95). The distinction between "what" and "why"
is one of the most important distinctions in monitoring. It gives high signal and low noise
(SRE Ch. 6, p. 87).

---

## 1. Symptom and cause

Rows 1 to 4 come from SRE Table 6-1, Ch. 6, p. 86–87. Rows 5 and 6 are derived. They carry a
mark.

| Symptom — page on this | A cause — debug with this |
|---|---|
| I serve HTTP 500s or 404s | Database servers are refusing connections |
| My responses are slow | CPUs are overloaded by a bogosort, or a crimped Ethernet cable causes partial packet loss |
| Users in Antarctica do not receive animated cat GIFs | The content distribution network blacklisted some client IP addresses |
| Private content is world-readable | A new software push made the ACLs forgotten and allowed all requests |
| *Derived:* the checkout page shows "delivery unavailable" | A vendor connection pool has no connection free |
| *Derived:* the daily statement is missing a day | The batch job did not run |

**One person's symptom is another person's cause** (SRE Ch. 6, p. 87). Slow database reads
are a symptom to the database engineer. The same slow reads are a cause to the frontend
engineer. Name the reader of the alert before you classify the condition.

**The boundary moves, so state the layer.** White-box monitoring is sometimes
symptom-oriented and sometimes cause-oriented. The answer depends on how informative the
white-box view is (SRE Ch. 6, p. 87). Read `03-white-box-and-black-box.md` for the two
vantage points.

**Across a vendor boundary, your cause is the vendor's symptom.** You need the caller-side
view and the vendor-side view together. Without both, you cannot separate a slow database
from a slow network (SRE Ch. 6, p. 87).

---

## 2. The three outputs

A monitoring system has three output types only (SRE App. B, p. 573–574).

| Output | The book's meaning | The condition that earns it |
|---|---|---|
| **Page** | A person must do something now | A user-visible symptom breaks the SLO, or breaks it soon |
| **Ticket** | A person must do something within a few days | The condition is real, and the response can wait |
| **Log** | No person must look at this now | The record supports a later diagnosis |

**If a condition is important enough to disturb a person, it must page or become a bug.**
The bug goes into the bug-tracking system (SRE App. B, p. 574). There is no fourth option.

**Email is not an output.** Google nicknames email alerts "alert spam". People rarely read
them, and people rarely act on them (SRE Ch. 6, p. 96, fn. 22). App. B calls email alerting an
attractive nuisance. It relies on eternal human vigilance, it works for a while, and it makes
the inevitable outage more severe (SRE App. B, p. 574).

**Move the subcritical conditions to a dashboard.** Google favors a dashboard that monitors
all ongoing subcritical problems. Pair that dashboard with a log, so a person can analyze
historical correlations (SRE Ch. 6, p. 96).

**Route by class inside the alerting system.** Page-worthy alerts reach the on-call rotation.
Important but subcritical alerts reach ticket queues. Everything else stays as informational
data on status dashboards (SRE Ch. 10, p. 154).

---

## 3. The questions to ask before any alert

Ask these questions when you create a monitoring rule or an alerting rule. They exist to help
you avoid false positives and pager burnout (SRE Ch. 6, p. 92).

Questions 1 to 4 test the alert itself. Question 5 tests the rest of the organization.

| # | The question | A failed answer means |
|---|---|---|
| 1 | Does this rule detect an otherwise undetected condition that is urgent, actionable, and actively or imminently user-visible? | The rule is not a page. Send it to a ticket or a log |
| 2 | Will I ever be able to ignore this alert, knowing it is benign? When and why, and how can I avoid that case? | A known benign case exists. Exclude it, or delete the rule |
| 3 | Does this alert definitely indicate that users are affected? Which detectable cases must I exclude, such as drained traffic or a test deployment? | The rule fires when no user is affected |
| 4 | Can I take action? Is the action urgent? Could the action be safely automated? Is the action a long-term fix or a short-term workaround? | Nobody can act. The rule is a defect. See Section 6 |
| 5 | Are other people paged for this issue, which renders at least one of the pages unnecessary? | Delete one of the two rules |

**Answer all five in writing before the rule ships.** Record the answers beside the rule.
The Observability Record in `../SKILL.md`, Section 12, carries the fields.

**Question 4 also selects the automation candidates.** The book asks whether the action could
be safely automated. That question is the gate that `self-healing-apis` must pass before it
adds any automatic remediation.

---

## 4. The four pager rules

These four statements are the philosophy behind the five questions (SRE Ch. 6, p. 92–93).

1. **A person can react with urgency only a few times a day before fatigue.**
   Treat the pager as a fixed daily budget, not as a channel.
2. **Every page must be actionable.** The on-call engineer must have an action to take.
3. **Every page response must require intelligence.** If a page merely merits a robotic
   response, it should not be a page.
4. **Pages should be about a novel problem, or an event that nobody has seen before.**
   A repeat page means the last incident produced no fix.

**A page that satisfies all four is valid, whatever produced it.** When a page meets these
four rules, it is irrelevant whether white-box monitoring or black-box monitoring triggered
it (SRE Ch. 6, p. 93).

---

## 5. When a cause may page

**Spend much more effort on catching symptoms than causes.** For causes, worry only about
very definite and very imminent causes (SRE Ch. 6, p. 93).

| The condition | Output | The book's reason |
|---|---|---|
| A user-visible symptom breaks the SLO now | Page | Urgent, actionable, user-visible (SRE Ch. 6, p. 92) |
| Zero redundancy, an N + 0 state | Page | Zero redundancy counts as imminent (SRE Ch. 6, p. 96, fn. 25) |
| A part of the service is nearly full | Page | A nearly full part counts as imminent (SRE Ch. 6, p. 96, fn. 25) |
| Saturation is nearly problematic | Page | Saturation is the one signal that pages early (SRE Ch. 6, p. 89) |
| A user-visible symptom breaks the SLO in days | Ticket | A person must act within a few days (SRE App. B, p. 573) |
| A cause that is possible but not imminent | Log | Spend the effort on symptoms instead (SRE Ch. 6, p. 93) |
| One machine of many fails | Log | Single-machine data is too noisy to be actionable (SRE Ch. 10, p. 141) |
| The response is one fixed command | Ticket, then automate the response | A rote response is a red flag (SRE Ch. 6, p. 95) |
| Another team is already paged for it | Nothing | Question 5 of the alert review (SRE Ch. 6, p. 92) |

**Do not page on a single-machine failure.** At Google's scale such data is too noisy to be
actionable. Build a system that survives a failure in the systems it depends on. Then alert
on the high-level service objective (SRE Ch. 10, p. 141).

Saturation and its early signal belong to `01-the-four-golden-signals.md`.

---

## 6. An alert that a person cannot act on is a defect

**Never trigger an alert simply because something seems a bit weird.** The book allows one
exception, which is a security audit on very narrowly scoped components (SRE Ch. 6, p. 84).

**All paging alerts must be actionable** (SRE Ch. 11, p. 166).

**Every paging alert must align with a symptom that threatens the SLO** (SRE Ch. 11, p. 166).
Misconfigured monitoring is the common cause of operational overload. Non-actionable alerts
and low-priority alerts that disturb the on-call every hour disrupt productivity
(SRE Ch. 11, p. 166).

**Pages with rote, algorithmic responses are a red flag** (SRE Ch. 6, p. 95). The page
consumes a person and returns no judgment.

**When an alert becomes frequent, find and eliminate the root cause.** If that resolution is
not possible, the alert response deserves full automation (SRE Ch. 6, p. 93). Route the
automation work to `self-healing-apis`.

**A workaround script is technical debt, not a fix.** It is easy to build layers of
unmaintainable technical debt when it patches problems instead of making real fixes. An
unwillingness to automate such pages implies that the team lacks confidence in its ability to
clear the debt. The book calls that a major problem worth escalation (SRE Ch. 6, p. 95).

**The gap catalog names three matching codes.** M-14 is a page on a cause. M-15 is a page
with a rote response. M-20 is an alert nobody can act on. Read
`observability-gap-catalog.md`.

---

## 7. Alert fatigue is a reliability risk

**Paging a human is an expensive use of an employee's time** (SRE Ch. 6, p. 84). A page at
work interrupts the workflow. A page at home interrupts personal time, and perhaps sleep.

**Too many pages destroy the detection capability that you purchased.** When pages occur too
frequently, employees second-guess incoming alerts, skim them, or ignore them. They sometimes
ignore a real page that the noise masks. Outages can be prolonged, because the noise
interferes with a rapid diagnosis and fix (SRE Ch. 6, p. 85).

**Effective alerting systems have good signal and very low noise** (SRE Ch. 6, p. 85).

**Alert fatigue causes serious alerts to receive less attention than they need**
(SRE Ch. 11, p. 166). The fatigue is a property of the on-call load, not of the person.

### The three limits that gate a new rule

`incident-response` owns the on-call load budget, the team size, and the toil ceiling. Read
`incident-response/references/06-interrupts-and-overload.md`. Check a new page against
these three limits only.

| Measure | Limit | Source |
|---|---|---|
| Incidents per 12-hour on-call shift | 2 maximum | SRE Ch. 11, p. 163 |
| Alerts per incident | Approach 1 to 1 | SRE Ch. 11, p. 167 |
| Page frequency review | Quarterly, with management | SRE Ch. 6, p. 95 |

**The budget is a capacity calculation.** One incident costs about six hours end to end, so a
12-hour shift absorbs two. The distribution of paging events should be very flat. The likely
median is zero per day. One component that pages every day leaves no room for the next break
(SRE Ch. 11, p. 163).

**An incident is one root cause, not one alert.** The book defines an incident as a sequence
of events and alerts with the same root cause. One postmortem discusses that sequence
(SRE Ch. 11, p. 163).

**Tune a noisy alert toward one alert per incident.** Regulate alert fan-out by grouping
related alerts. Silence duplicate or uninformative alerts during an incident
(SRE Ch. 11, p. 167). The alerting system provides the machinery. It can inhibit one alert
while another is active. It can deduplicate alerts that carry the same labelset. It can
combine or split alerts by labelset (SRE Ch. 10, p. 154).

**Review the page frequency with management every quarter.** Google expresses the statistic
as incidents per shift. One incident might be a few related pages (SRE Ch. 6, p. 95).

---

## 8. The shape a paging rule must have

Ch. 10 gives the mechanical form. `04-time-series-and-rules.md` covers the full rule
language.

| Part | The rule | The book's worked example |
|---|---|---|
| Ratio | Compare the error rate to the request rate | `dc:http_errors:ratio_rate10m > 0.01` |
| Volume floor | Add an absolute rate, so low traffic cannot page | `and dc:http_errors:rate10m > 1` |
| Window | Compute the rate over a history range, not one point | `[10m]` |
| Duration | Hold the condition before the alert fires | `for 2m` |
| Severity | Route by a label, not by the rule name | `labels { severity=page }` |
| Objective | Compare the result to the SLO | Fire when the SLO is missed or in danger |

**Set the duration to at least two rule evaluation cycles.** That duration makes sure no
missed collection sends a false page (SRE Ch. 10, p. 153). It also stops the alert from
flapping.

**Pair a ratio with an absolute rate.** A ratio alone pages on two failures out of ten
requests. The volume floor removes that class of false page (SRE Ch. 10, p. 153).

**Alert when the SLO is missed, or when it is in danger of being missed** (SRE Ch. 10,
p. 151).

---

## 9. Rules that must not exist

| The test | The action | Source |
|---|---|---|
| The rule tries to learn its own threshold, or to detect causality | Remove it. Avoid magic systems | SRE Ch. 6, p. 85 |
| The rule depends on a chain of other conditions | Remove it, unless every part of the chain is very stable | SRE Ch. 6, p. 85–86 |
| The rule is exercised less than once a quarter | Candidate for removal | SRE Ch. 6, p. 91 |
| The signal appears on no prebaked dashboard and in no alert | Candidate for removal | SRE Ch. 6, p. 91 |
| The rule pages on one machine of many | Move it to a log. Alert on the service objective | SRE Ch. 10, p. 141 |

**The one endorsed dependency rule.** "If a datacenter is drained, then do not alert me on
its latency" is acceptable. Traffic draining is a very stable part of the system
(SRE Ch. 6, p. 85–86).

**The rules that catch real incidents most often must be as simple, predictable and reliable
as possible** (SRE Ch. 6, p. 91). Unused collection and alerting configuration is pure
maintenance cost.

---

## 10. War stories

**Bigtable and the tale of over-alerting (SRE Ch. 6, p. 93–94).** The Bigtable SLO rested on
the mean performance of a synthetic well-behaved client. Problems in Bigtable and in the
lower storage layers drove the mean with a large tail. The worst 5% of requests were often
much slower than the rest. Email alerts fired as the SLO approached, and pages fired
when the service exceeded it. Both types fired voluminously. The team spent large amounts of
engineering time on triage to find the few actionable alerts, and often missed the problems
that actually affected users. Many pages were non-urgent, well-understood infrastructure
problems with a rote response or no response. The remedy had three parts. The team improved
Bigtable performance, temporarily reduced the SLO target to the 75th percentile request
latency, and disabled email alerts. **This proves that a mean-based SLO plus alerts on both
approach and breach produces noise that hides real user impact.**

**Gmail and the scriptable human (SRE Ch. 6, p. 94–95).** Early Gmail ran on Workqueue, a
distributed process management system built for batch processing of search-index pieces and
retrofitted to long-lived processes. Scheduler bugs in a relatively opaque codebase proved
hard to beat. Alerts fired whenever Workqueue de-scheduled an individual task, and Gmail had
many thousands of tasks. Each task represented a fraction of a percent of users. SRE built a
tool that poked the scheduler in the right way to minimize user impact. The team then debated
whether to automate the whole loop. Some engineers worried that the workaround would delay a
real fix. **This proves that a rote page is a red flag, and that a team must protect the
long-term fix against the convenience of the script.**

**The alert that did not fire (SRE Ch. 10, p. 149–153).** In the worked example five
webserver tasks produce an error ratio of 0.15. That value is fifteen times the 0.01
threshold. The `ErrorRatioTooHigh` rule still does not fire, because the absolute error rate
is not above 1 per second. Once the error rate passes 1 per second, the alert goes pending
for two minutes, and then fires. **This proves two things. A ratio without a volume floor pages on
noise at low traffic. A duration separates a real condition from a transient.**

**The fourth page in a week (SRE Ch. 11, p. 164–165).** The same alert pages for the fourth time
in a week. An external infrastructure system caused the previous three. The book states that
it is extremely tempting to exercise confirmation bias and to associate this occurrence with
the previous cause. Stress hormones make the problem worse, because they push the engineer
from deliberate cognition toward fast, unreflective action. **This proves that a recurring
alert corrodes the quality of diagnosis, not only the mood of the on-call engineer.**

**The threshold raised on the record (SRE App. F, p. 589).** In the example production
meeting the alert `AnnotationConsistencyTooEventual` paged five times in one week. The
suspected cause was cross-regional replication delay between two Bigtables, and no fix was
near. The team raised the acceptable delay threshold from 60 seconds to 180 seconds to reduce
unactionable alerts. The change carries a bug number and a named owner. **This proves that
alert tuning is a tracked change with an owner, and not a silent act of silencing.**

**The engineer who did not answer the page (Release It! Sec. 16.8, p. 261).** During a
retail outage the home-delivery scheduling system sat at 100% CPU on its one remaining
server. Its on-call engineer had received several pages about the high CPU condition and had
not responded. That group is paged routinely for transient CPU spikes that are false alarms. Nygard states that all the false positives had trained them to ignore high CPU
conditions. **This proves that a threshold set above reality destroys the detection
capability that the alarm was purchased to give.**

**The warning chime that nobody heard (Release It! Sec. 17.7, p. 304).** A plant supervisor
watched a control-room operator silence a warning chime by reflex. The operator then denied
the act. He denied it because he had no conscious memory of the act, and not
because he feared a consequence. The system had trained him to disregard the chime so
completely that he could silence it without awareness. **This proves that alert fatigue can
remove the alarm from conscious attention entirely, which is a failure the alert count alone
will never show.**

---

## 11. The review procedure

Run these steps for every new alert rule and every changed alert rule.

1. Name the reader. State the layer at which this condition is a symptom.
2. Classify the condition as a symptom or a cause. Use the table in Section 1.
3. If the condition is a cause, apply Section 5. Only a very definite and very imminent cause
   may page.
4. Answer the five questions in Section 3 in writing.
5. Test the rule against the four pager rules in Section 4.
6. Select one output from Section 2. Reject any route to an email alias.
7. Give the rule a ratio, a volume floor, a window, a duration and a severity label.
8. Write the runbook action. If the action is one fixed command, convert the page to a ticket
   and automate the action.
9. Add the new page to the load budget in Section 7. Check the two-per-shift limit.
10. Record the answers in the Observability Record. Read `../SKILL.md`, Section 12.

**A rule that fails any step does not ship as a page.** It ships as a ticket, as a log, or
not at all.

---

## 12. Where to read next

| File | Read it for |
|---|---|
| `01-the-four-golden-signals.md` | The measurements a symptom alert reads. Saturation as the early signal |
| `03-white-box-and-black-box.md` | The two vantage points, and which one detects a symptom |
| `04-time-series-and-rules.md` | The rule language, aggregation, labels and rule evaluation |
| `06-the-operations-database.md` | Where a threshold comes from, and how an expectation matures |
| `observability-gap-catalog.md` | Group C, the alerts that must not page. Codes M-14 to M-21 |
| `../SKILL.md` | The Observability Record and the proportionality table |
