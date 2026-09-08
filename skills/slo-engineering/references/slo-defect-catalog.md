# SLO Defect Catalog

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 1, pp. 25–29. Ch. 3, pp. 48–61. Ch. 4, pp. 62–73. Ch. 5, pp. 75–81. Ch. 6, pp. 88–89.
Ch. 33, p. 562. App. A, pp. 567–570. App. B, pp. 571–576. App. C, p. 577. App. D,
pp. 579–583. App. F, pp. 588–589. *Release It!* — Michael T. Nygard (Pragmatic Bookshelf,
2007). Sec. 4.10, pp. 102–105. Sec. 13.1, pp. 229–230. Sec. 13.2, pp. 230–232. Sec. 14.1,
p. 243. Sec. 14.2, p. 245.

A catalog of named defects in a reliability target, an indicator, an error budget, or a
release gate. Read it during every review of a target, a budget, or a gate.

Each entry has three parts:

- **Signature** — the text, the config, the design, or the process that you search for.
- **Consequence** — what goes wrong, and when.
- **Fix** — the required remedy.

**How the skills use it.** `slo-engineering` runs the applicable groups against the SLO
Record. `code-review` runs them against a difference. `verify` runs them against a phase.
`planner` and `detail-planning` run them against a design. `self-healing-apis` consumes
group F. Report only the defects that apply. "Not applicable. This change adds no
user-visible path and no dependency." is a valid result. An invented finding is worse.

**Severity.** 🔴 defeats the control itself, or hides a loss the user already suffered. It
blocks the merge. 🟡 causes an outage or a wrong decision under load. 🔵 is a risk to
operation or to maintenance.

**Every entry carries a chapter and a page, or the tag Modern.** A **Modern** entry names a
practice that neither book describes. Never attribute a modern practice to either book.

---

## A. Objective definition

### 🔵 L-01 — The hundred percent target
**Signature:** a requirement states `100% uptime`, `zero downtime`, or `it must never go down`.
Search the requirement files for `100%`, `always`, and `never fail`.
**Consequence:** the team cannot trade any risk against any feature. Cost rises without a
limit. A user on a 99% reliable smartphone cannot tell 99.99% from 99.999% (SRE Ch. 3, p. 48).
100% is probably never the right target (SRE Ch. 3, p. 61), and an absolute requirement is
unrealistic (SRE Ch. 4, p. 71). The book names the pacemaker and the anti-lock brake as the
notable exceptions (SRE Ch. 1, p. 26).
**Fix:** replace the absolute with a number, a window, and a formula.

### 🔵 L-02 — The target copied from present performance
**Signature:** the objective equals last month's measurement. The record names no other source.
**Consequence:** the team locks itself into heroic work to hold the number. The system cannot
improve without a redesign (SRE Ch. 4, p. 71).
**Fix:** derive the target from the user and from the cost test in `../SKILL.md`, Table 7.
Start with a loose target. Tighten it later (SRE Ch. 4, p. 71).

### 🟡 L-03 — One objective for the whole system
**Signature:** a single line such as `The system shall be available 99.9% of the time`. No
feature list follows it.
**Consequence:** every feature inherits the objective of the most expensive feature. A Circuit
Breaker can hold the system "up" while one feature is dead. The words "the system" also cover
every call to another system, in silence (Release It! Sec. 13.2, pp. 230–231).
**Fix:** write one objective per feature or per business process. Use `../SKILL.md`, Table 11.

### 🟡 L-04 — "Down" is not defined
**Signature:** the agreement answers none of the eight probe questions. It names no response
code list, no per-step time bound, no probe frequency, and no formula.
**Consequence:** a feature that answers in 27.5 minutes counts as available. A feature that
answers in 50 ms with an error for every user also counts as available. The first incident
becomes a blamestorm a year later (Release It! Sec. 13.2, pp. 230–232).
**Fix:** answer the eight questions before launch. Record the answers in the agreement
(Release It! Sec. 13.2, pp. 231–232).

### 🔵 L-05 — Objective sprawl
**Signature:** more than a handful of objectives cover one service. Nobody can name the
objective that stops a release.
**Consequence:** nobody watches any of them. An objective that you cannot quote to win a
priority argument is not worth holding (SRE Ch. 4, p. 71).
**Fix:** keep as few objectives as possible. Delete the rest (SRE Ch. 4, p. 71).

### 🔵 L-41 — The number chosen for its sound
**Signature:** the requirement says `five nines`. No cost calculation appears beside it.
**Consequence:** the sponsor picked a number that sounds technical. Each further nine raises
the implementation cost by about a factor of ten. It raises the yearly operation cost by about
a factor of two (Release It! Sec. 13.1, pp. 229–230).
**Fix:** run the cost test before you accept the number. Compare the avoided loss against the
added lifecycle cost (Release It! Sec. 13.1, pp. 229–230. SRE Ch. 3, p. 54).

---

## B. Indicator choice and measurement point

### 🟡 L-06 — Server-side measurement only
**Signature:** the indicator reads a server counter. No client measurement and no edge
measurement exists in the dashboard or in the code.
**Consequence:** the indicator misses faults that the user feels. The page JavaScript, the
network path, and the client library stay invisible. Google moved the Gmail measurement to the
client. The measured availability fell at once (SRE Ch. 4, p. 63, p. 67. SRE App. B, p. 572).
**Fix:** measure at the client or at the edge. State the measurement point in the indicator
specification (SRE Ch. 4, p. 69).

### 🟡 L-07 — No correctness indicator
**Signature:** the indicator set holds availability and latency only. No check tests the answer.
**Consequence:** the service returns HTTP 200 with a wrong answer and stays inside the
objective. Correctness is a property of the data, not of the infrastructure. Only an end-to-end
test detects wrong content (SRE Ch. 4, p. 66. SRE Ch. 6, p. 88).
**Fix:** add a correctness indicator. Ask whether the service returned the right data and did
the right analysis (SRE Ch. 4, p. 66).

### 🟡 L-08 — Availability judged by a person
**Signature:** the availability number comes from a help desk ticket count, or from a person
who clicks through the feature.
**Consequence:** the number is neither measurable nor timely. Nobody can defend it after an
incident (Release It! Sec. 13.2, p. 231).
**Fix:** run an automated synthetic transaction against the feature. Name the device that runs
it. Name the way that device reports a problem (Release It! Sec. 13.2, p. 231).

### 🔵 L-09 — A probe without a monitoring identity
**Signature:** the probe uses a real customer account, or it writes records that the business
reads. Read the probe configuration and search for a designated user id.
**Consequence:** the probe pollutes the production data (Release It! Sec. 13.2, p. 231).
**Fix:** give the probe a designated monitoring user id. Exclude its records from the business
reports (Release It! Sec. 13.2, p. 231).

### 🔵 L-10 — An objective on the incoming request rate
**Signature:** a document names a `QPS SLO`, or a target on requests per second from external
users.
**Consequence:** the number measures user desire, not service behavior. The team cannot control
it and cannot act on a breach (SRE Ch. 4, p. 63).
**Fix:** state throughput as a capacity number. Keep the objective on latency, on availability,
or on correctness (SRE Ch. 4, p. 63, p. 66).

### 🟡 L-42 — A pipeline with no end-to-end indicator
**Signature:** a batch job, a data pipeline, or a webhook consumer carries an uptime number
only. No indicator counts the records that the pipeline processed.
**Consequence:** the process stays "up" while records wait or fail. No continuous request
stream exists, so the uptime number describes nothing that the user receives (SRE Ch. 3,
pp. 50–51. SRE Ch. 4, p. 66).
**Fix:** define throughput and end-to-end latency for the pipeline. Divide the records
processed successfully by the records received (SRE Ch. 4, p. 66. SRE Ch. 3, p. 51).

---

## C. Aggregation and statistics

### 🟡 L-11 — Mean latency as the indicator
**Signature:** the dashboard query or the alert rule contains `avg(`, `mean`, or `average
response time` for a latency indicator.
**Consequence:** 5% of requests can run 20 times slower with no change in the mean. The 99th
percentile moves across the day while the average line stays flat (SRE Ch. 4, pp. 67–68).
**Fix:** treat the metric as a distribution. Publish the 50th, 95th, 99th, and 99.9th
percentiles (SRE Ch. 4, p. 68).

### 🟡 L-12 — No aggregation window on the indicator
**Signature:** the indicator says `99% of requests`. It states no averaging interval, no
region, and no measurement frequency.
**Consequence:** one system serves 200 requests a second in even seconds and 0 in odd seconds.
It carries the same one-minute average as a constant 100 a second. The instantaneous load is
double (SRE Ch. 4, p. 67).
**Fix:** state the six template dimensions. They are the interval, the region, the frequency,
the requests included, the data source, and the latency point (SRE Ch. 4, p. 69).

### 🔵 L-13 — The timeout is not recorded with the latency indicator
**Signature:** the latency objective sits in one file and the client timeout sits in another.
The indicator specification does not name the timeout value.
**Consequence:** the timeout truncates the distribution at its own value. No success can exceed
it. A censored 99.9th percentile looks healthy (SRE Ch. 4, p. 68).
**Fix:** record the timeout beside every latency indicator. Report a timeout as a failure. The
book counts a request above the committed time as an error (SRE Ch. 4, p. 68. SRE Ch. 6, p. 88).

### 🟡 L-14 — Time-based availability for a partly available service
**Signature:** the formula divides uptime minutes by total minutes. The service runs many
replicas, or its load varies across the day.
**Consequence:** the number hides a partial outage. Some replicas serve while others fail
(SRE Ch. 3, p. 50. SRE App. A, p. 570).
**Fix:** use the aggregate formula. Divide successful requests by all requests over a rolling
window (SRE Ch. 3, p. 50).

### 🟡 L-15 — An automated action keyed to an assumed distribution
**Signature:** automation restarts a server, sheds load, or reverts a release when a latency
outlier crosses a static threshold. No test of the distribution exists.
**Consequence:** the action fires too often, or not often enough. The book gives the restart of
a server with high request latencies as the example (SRE Ch. 4, p. 69).
**Fix:** verify the distribution before you connect an automatic action to a threshold. Do not
assume that the mean equals the median (SRE Ch. 4, pp. 68–69).

### 🟡 L-43 — Failed requests inside the latency indicator
**Signature:** the latency query counts every response. It applies no filter on the response
code.
**Consequence:** an HTTP 500 from a lost backend connection returns very fast. The fast errors
pull the latency number down during an outage. The dashboard improves while the service fails
(SRE Ch. 6, p. 88).
**Fix:** measure the latency of successful requests apart from the latency of failed requests
(SRE Ch. 6, p. 88). `observability` owns the four golden signals and the wrong measurement for
each one. Read `observability/references/01-the-four-golden-signals.md` before you write
the query.

---

## D. The error budget

### 🟡 L-16 — An objective with no budget
**Signature:** the document states a target percentage. It states no allowed failure amount in
minutes and none in events.
**Consequence:** nobody can say how much risk remains this month. Every outage becomes an
argument about blame (SRE Ch. 3, p. 59, p. 61).
**Fix:** compute budget = 1 − objective. Convert it to minutes and to events for the window.
Use `../SKILL.md`, Table 5 (SRE App. B, p. 573. SRE App. A, pp. 567–570).

### 🔴 L-17 — The shipping party measures its own budget
**Signature:** the burn number comes from the deploy tool, from a release dashboard, or from
the team that ships the change.
**Consequence:** the gate holds no independent evidence. The book requires a neutral third
party. It names the monitoring system as that party (SRE Ch. 3, p. 59).
**Fix:** the monitoring system measures the budget. The deploy tool reads that number. The
deploy tool does not produce it (SRE Ch. 3, p. 59).

### 🔵 L-18 — On/off control only
**Signature:** the policy holds two states. They are ship and freeze. No intermediate action
exists.
**Consequence:** the service swings between full speed and full stop. The book names this
bang/bang control. It calls the graduated forms more effective (SRE Ch. 3, p. 60, footnote 15).
**Fix:** add the intermediate states of `../SKILL.md`, Table 6. Slow the release rate. Then
revert the release. Then freeze (SRE Ch. 3, p. 60).

### 🟡 L-19 — Dependency and infrastructure faults excluded from the budget
**Signature:** the policy text says the budget counts only the failures that our own code
caused.
**Consequence:** the budget stops representing what the user experienced. A network outage or a
datacenter fault also spends the budget. It does reduce the number of pushes that remain in the
quarter (SRE Ch. 3, p. 60).
**Fix:** count every failure that the user sees. Attribute the cause apart from the count
(SRE Ch. 3, p. 60).

### 🔵 L-20 — The wrong reset period for the target
**Signature:** a monthly budget reset for an objective above 99.99%.
**Consequence:** 99.999% permits 25.9 seconds a month. One event exhausts the month. The policy
then loses its ability to guide anything (SRE App. A, p. 570. SRE App. B, p. 573).
**Fix:** reset the budget quarterly above 99.99%. Keep the monthly reset below it (SRE App. B,
p. 573).

### 🔴 L-21 — A gate with no authority
**Signature:** the policy says releases "should" stop. No named person and no system can stop
them. The pipeline holds no blocking step.
**Consequence:** the budget becomes a report, not a control. The scheme works only when the
reliability team can stop launches (SRE Ch. 3, p. 60).
**Fix:** name the party that can stop a release. Keep that party separate from the party whose
objective is to ship. A trading firm gives this authority to a separate enforcement team
(SRE Ch. 3, p. 60. SRE Ch. 33, p. 562).

### 🔴 L-44 — Downtime labeled planned after the event
**Signature:** an outage appears in the record as planned downtime. No schedule exists from
before the event.
**Consequence:** the label removes a real loss from the budget. The book grants the exemption
to an outage that is occasional, regular and scheduled, and it counts only that outage as
planned downtime (SRE Ch. 3, p. 53). An outage that becomes "scheduled" after it ends is
unplanned downtime, and it spends the budget.
**Fix:** define each maintenance window in advance. Charge every other loss to the budget as
unplanned downtime (SRE Ch. 3, p. 53, p. 60). Announce the window as well. The announcement is
this skill's requirement, not the book's, because a schedule the user cannot read gives the
user no notice.

---

## E. Release policy and rollout

### 🟡 L-22 — A budget gate with no progressive rollout
**Signature:** the pipeline deploys to 100% of traffic in one step. The gate reads the budget
after the deploy completes.
**Consequence:** the budget is spent before the gate can act. A non-emergency rollout must
proceed in stages, across small fractions of traffic and across different geographies
(SRE Ch. 1, p. 29. SRE App. B, p. 572).
**Fix:** stage the rollout. Detect during each stage. Then revert. The three automations work
only as a set. `reverse-branching` owns the mechanism (SRE Ch. 1, p. 29).

### 🟡 L-23 — The engineer who ships also supervises the stage
**Signature:** the runbook tells the engineer to watch the graph during the rollout. No
automated check gates the next stage.
**Consequence:** the observer and the actor are the same party. The book prefers a monitoring
system that is demonstrably reliable (SRE App. B, p. 572).
**Fix:** a monitoring system supervises every stage. That system gates the next stage
(SRE App. B, p. 572).

### 🟡 L-24 — Diagnosis before the revert
**Signature:** the runbook orders the responder to find the cause first, then to decide whether
to revert.
**Consequence:** mean time to recovery rises. The book gives the opposite order. Revert first
and diagnose afterwards (SRE App. B, p. 572).
**Fix:** revert on the signal. Diagnose after the service returns (SRE App. B, p. 572).

### 🔵 L-25 — A silent exception to the freeze
**Signature:** a release lands during a freeze. The tracker holds no exception record and no
owner.
**Consequence:** the policy loses force. Nobody can audit the decision. The worked postmortem
records the exception request as an owned, tracked action item (SRE App. D, p. 581. SRE App. F,
p. 588).
**Fix:** file every freeze exception as a tracked item. Give it a named owner and a reason
(SRE App. D, p. 581).

### 🔵 L-45 — The incident ends by judgment
**Signature:** the incident document holds no exit criterion. The responder declares the
incident over when the graph looks normal.
**Consequence:** the team ends the incident during a temporary recovery. The service then fails
again with no incident open (SRE App. C, p. 577. SRE App. D, p. 583).
**Fix:** write the exit criterion before the response starts. State it as the objective held
for a continuous period. The book's example uses 30 minutes (SRE App. C, p. 577).

---

## F. Dependency arithmetic

### 🔴 L-26 — SLA inversion
**Signature:** a stated objective sits above the availability of a dependency inside the same
feature. Compare the objective against the dependency table.
**Consequence:** a single failure in any dependency fails the feature. The ceiling is the
product of the dependency availabilities, capped by the worst provider. Five dependencies at
99.9% cap the feature at 99.5% (Release It! Sec. 4.10, pp. 102–104).
**Fix:** decouple from the lower-availability system and degrade the feature. Add a Circuit
Breaker per dependency. Then restate the objective per feature (Release It! Sec. 4.10, p. 104).

### 🟡 L-27 — An incomplete dependency inventory
**Signature:** the dependency table lists application vendors only. It omits the corporate DNS
cluster, the mail transfer service, the message broker, and the storage network.
**Consequence:** an unlisted dependency sets the true ceiling. Every dependency also exposes
three layers that fail on their own. They are the transport, the naming service, and the
application protocol (Release It! Sec. 4.10, p. 103, p. 105).
**Fix:** list every dependency, including the infrastructure. Record the availability of each
one. The firewall rule set indexes the integration points (Release It! Sec. 14.1, p. 243).

### 🔴 L-28 — A no-SLA dependency counted as available
**Signature:** the dependency table shows a blank cell or `N/A`. The feature still claims an
availability number.
**Consequence:** that feature cannot offer an availability figure at all. Project Frammitz met
its 99.99% target only through luck (Release It! Sec. 4.10, pp. 102–104).
**Fix:** mark the cell "No SLA". Then decouple the feature from that dependency, or remove the
number from the agreement (Release It! Sec. 4.10, p. 103).

### 🟡 L-29 — The vendor dashboard used as the indicator
**Signature:** the availability report cites the vendor status page, or the vendor's own uptime
figure.
**Consequence:** the vendor measures its own server side. Server-side collection misses a range
of problems that affect the caller and leave the server metrics unchanged (SRE Ch. 4, p. 67).
Neither book discusses a vendor status page. This entry applies the book's measurement-point
rule to a vendor call. The parts a vendor page cannot show you are the network path, the
serialization, and your own client library.
**Fix:** measure the dependency at your caller edge. Use the vendor page only to corroborate
(SRE Ch. 4, p. 67. SRE App. B, p. 572).

### 🔵 L-30 — An objective built on observed vendor behavior
**Signature:** the design assumes the availability that the vendor has delivered so far, not the
level that the vendor contracted. No fallback path exists for that dependency.
**Consequence:** users build on the reality you offer, not on the promise you publish. A
dependency that has never failed is the one your code has no fallback for. Global Chubby
produced user-visible outages for this reason (SRE Ch. 4, pp. 64–65, p. 73).
**Fix:** design to the contracted level. Take a planned outage of the dependency in a drill. The
drill exposes the missing fallback path (SRE Ch. 4, p. 65).

### 🟡 L-46 — The managed platform is absent from the ceiling — **Modern**
**Signature:** the dependency table lists application vendors only. It omits the managed control
plane, the managed database, the identity provider, and the content delivery network.
**Consequence:** each platform publishes its own objective, and the ceiling rule applies to it
without change. The best you can do is the level of the worst provider (Release It! Sec. 4.10,
p. 103). Neither book names these platform layers.
**Fix:** add every managed platform to the dependency table. Record its published objective.
Compute the ceiling again (Release It! Sec. 4.10, p. 103).

---

## G. Toil and operating load

### 🔵 L-31 — Toil never measured
**Signature:** no survey, no ticket category, and no time record separates the operational work
from the engineering work.
**Consequence:** toil expands and can fill 100% of everyone's time. The reliability team becomes
an operations team (SRE Ch. 5, p. 77).
**Fix:** measure toil each quarter. Classify each task with `../SKILL.md`, Table 10 (SRE Ch. 5,
pp. 77–78).

### 🟡 L-32 — Operational work above the cap with no valve
**Signature:** measured toil stays above 50% across several quarters. No written rule redirects
the overflow.
**Consequence:** no engineering time remains. The named harms are career stagnation, low
morale, slower delivery, attrition, and a breach of faith with new hires (SRE Ch. 5, pp. 77–80).
**Fix:** cap the operational work at 50% of the team's time. Redirect the overflow to the
product development team until the load falls back (SRE Ch. 1, p. 25. SRE App. B, p. 575).

### 🔵 L-33 — "It needs human judgment" used as a toil exemption
**Signature:** a service pages several times a day. A written argument says that each page needs
complex human judgment and is therefore not toil.
**Consequence:** the service is poorly designed and carries unnecessary complexity. The work
stays toil until the redesign ships (SRE Ch. 5, p. 81, footnote 21).
**Fix:** remove the failure condition, or handle it automatically. Count the response work as
toil until then (SRE Ch. 5, p. 81, footnote 21).

### 🟡 L-34 — Alert volume above the shift ceiling
**Signature:** more than two paging events per 8-hour to 12-hour shift, as a pattern.
**Consequence:** the responder cannot investigate thoroughly and cannot learn from the events.
This does not improve with scale (SRE Ch. 1, p. 25. SRE App. B, p. 576).
**Fix:** treat the volume as a defect in the design, in the monitoring sensitivity, or in the
postmortem response. Change a threshold only as a tracked item (SRE App. B, p. 576. App. F,
p. 589).

### 🟡 L-47 — The budget signal has no output type
**Signature:** the alert that reports the budget ends at a mailbox. A person reads the message
and decides whether to act. **Modern:** the same defect now appears as a chat channel that
nobody owns. Neither book names a chat channel.
**Consequence:** monitoring must never require a human to interpret the alerting domain. The
book calls email alerting an attractive nuisance. The practice works for a while, and it makes
the inevitable outage more severe (SRE Ch. 1, p. 27. SRE App. B, p. 574).
**Fix:** give the budget signal one of the three outputs, which are a page, a ticket, or a log
line (SRE App. B, pp. 573–574). `observability` owns the routing and the rule set. This entry
exists so that a budget review cannot end without naming the output.

---

## H. Expectation and agreement

### 🟡 L-35 — Chronic overachievement
**Signature:** measured availability stays far above the objective across several quarters.
Dependent teams hold no fallback path for this service.
**Consequence:** other services add dependencies and assume that the service never fails. High
reliability gives a false sense of security. Global Chubby is the named case (SRE Ch. 4,
pp. 64–65).
**Fix:** meet the objective, but do not greatly exceed it. Take a planned outage, throttle some
requests, or design the service so it is not faster under light load (SRE Ch. 4, p. 73).

### 🔵 L-36 — No published objective
**Signature:** no objective document exists for a service that other teams call.
**Consequence:** users invent their own beliefs. Over-reliance follows, or under-reliance
follows. A prospective user avoids a system that the user believes is flaky (SRE Ch. 4, p. 64).
**Fix:** choose an objective and publish it (SRE Ch. 4, p. 64).

### 🔵 L-37 — No safety margin
**Signature:** the internal objective equals the advertised objective, to the digit.
**Consequence:** a chronic problem becomes visible outside before the team can respond. No room
remains for a reimplementation that trades performance for cost (SRE Ch. 4, pp. 72–73).
**Fix:** set the internal objective tighter than the advertised one. Google Apps for Work
advertises a 99.9% external quarterly target. It backs that target with a stronger internal
target and with a contract that stipulates penalties (SRE Ch. 3, pp. 52–53).

### 🟡 L-38 — The word "SLA" with no consequence
**Signature:** a document uses the word SLA. Ask what happens if the team misses it. No answer
exists in the document.
**Consequence:** the team believes it holds a contract and plans no response to a breach. Be
conservative in what you advertise. A broad constituency is hard to change (SRE Ch. 4, p. 65,
p. 73).
**Fix:** apply the test in `../SKILL.md`, Table 1. No explicit consequence means it is an
objective, not an agreement. Write the consequence, or rename the document (SRE Ch. 4, p. 65).

---

## I. Modern practice gaps

Neither book describes the two items in this group. Both carry the tag **Modern**. Entry L-46
in group F carries the same tag.

### 🔵 L-39 — No burn rate alert — **Modern**
**Signature:** the only budget signal is a monthly report or a quarterly report. No alert fires
while the budget drains inside the window.
**Consequence:** the team learns of exhaustion after the fact. It cannot slow the release rate
in time. The books give the budget and the freeze rule (SRE Ch. 3, p. 60. SRE App. B, p. 573).
Neither book describes an alert on the rate of spend.
**Fix:** alert on the fraction of the budget that the service spends per unit of time. Use one
fast window and one slow window. A short spike and a slow drain then both raise an alert.

### 🔵 L-40 — The objective exists only in a dashboard — **Modern**
**Signature:** the objective is a saved query or a panel threshold. No file in version control
holds it. Search the repository for the target value and find nothing.
**Consequence:** the number changes with no review and with no history. Nobody can say what the
objective was during last month's incident. Nygard requires version control, a link to change
control, and an automated drift audit for the operational configuration (Release It! Sec. 14.2,
p. 245).
**Fix:** store the indicator and the objective in version control. Generate the dashboard and
the alert from that file. Audit for drift (Release It! Sec. 14.2, p. 245).

---

## Index by symptom

| Symptom | Check these entries |
|---|---|
| The requirement is a wish, not a number | L-01, L-02, L-04, L-41 |
| The dashboard is green and the users complain | L-06, L-07, L-11, L-13, L-14, L-43 |
| The alert did not fire during the incident | L-11, L-12, L-13, L-39 |
| Nobody acted on the alert | L-34, L-47 |
| A release shipped during a freeze | L-21, L-25 |
| The budget was spent before anyone noticed | L-16, L-17, L-19, L-22, L-39 |
| The budget hides an outage the users saw | L-19, L-44 |
| The target cannot be met, ever | L-26, L-27, L-28, L-46 |
| A vendor failed and the feature had no fallback | L-28, L-30, L-35 |
| The pipeline is "up" and the records are late | L-10, L-42 |
| The team has no time to build anything | L-31, L-32, L-33, L-34 |
| The post-incident meeting became an argument | L-04, L-16, L-38 |
| The incident closed too early | L-04, L-45 |
| Automation acted at the wrong moment | L-15, L-24 |
