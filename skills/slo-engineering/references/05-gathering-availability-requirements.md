# Gathering and Writing Availability Requirements

**Sources:** *Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007). Sec. 13.1,
pp. 229–230. Sec. 13.2, pp. 230–232. Sec. 13.4, p. 238.
Sec. 4.10, pp. 102–105. Ch. 15, p. 250. *Site Reliability Engineering* — Beyer, Jones,
Petoff, Murphy (O'Reilly, 2016). Ch. 3, pp. 52–54. Ch. 4, pp. 62–73. App. B, p. 572.

This file holds the method. It gathers one availability requirement for one feature. It then
writes that requirement so an operator can act on it. `../SKILL.md`, Section 6 holds the
summary row and the eight questions. This file holds the interview, the cost arithmetic, and
the writing rules.

**Read this file when a sponsor states a target. Read it when you must define the word
"down".**

---

## 1. The unit is the feature

**A want with no price attached is always unlimited.** Nygard opens the chapter with children
and ice cream. The child knows no cost, so the child asks for all of it (Release It!
Sec. 13.1, p. 229). A sponsor answers the same way. The inexperienced sponsor answers "100%".
The knowledgeable sponsor answers "five nines", because the phrase sounds technical
(Release It! Sec. 13.1, p. 229).

**"The system" is the wrong unit.** Nygard rejects the sentence "The system shall be
available 99.9% of the time" as a requirement. "Vagueness lurks behind every word of that
sentence" (Release It! Sec. 13.2, p. 230).

### Table 1 — Three faults of a system-wide number

| Fault | Mechanism | Source |
|---|---|---|
| The number covers calls you do not control | "The system" includes calls to other systems inside and outside the enterprise. You accept accountability for all of them. | Release It! Sec. 13.2, p. 230 |
| A Circuit Breaker hides a dead feature | The system answers requests and serves pages. One feature behind an open breaker returns nothing. The aggregate number stays green. | Release It! Sec. 13.2, p. 230 |
| Every feature inherits the most expensive target | One number forces the cheapest feature to the level of the costliest one. The cost of a nine is not the same for each feature. | Release It! Sec. 13.1, p. 230. Sec. 13.2, p. 231 |

**Write the requirement per feature or per business process.** A hotel chain names the
property locator, the online reservation, the loyalty club subscription, and the event
booking. Each one carries a different importance to the business. Event booking and online
reservation both produce revenue, so they hold the highest targets (Release It! Sec. 13.2,
p. 231).

**Signal that the unit is wrong:** the requirement document names no feature. Report defect
`L-03` from `slo-defect-catalog.md`.

---

## 2. Price the requirement before you agree to it

**Frame the decision as actual cost against avoided loss.** That framing is the one Nygard
prescribes (Release It! Sec. 13.1, p. 229). Run these five steps with the sponsor.

1. Convert the percentage into downtime minutes per month.
2. Multiply those minutes by the peak revenue rate. This gives the worst-case loss.
3. Subtract the loss at the higher target from the loss at the lower target.
4. Price the added implementation cost and the added operation cost across the life span.
5. Compare the avoided loss against the added cost. Decide with the sponsor.

### Table 2 — Nygard's worked comparison

**Sources:** Release It! Sec. 13.1, pp. 229–230.

| Item | 98% | 99.99% |
|---|---|---|
| Downtime per month | 864 minutes | 4 minutes |
| Cost of downtime per month, at $1,500 an hour of peak revenue | $21,600 | $108 |
| Added cost | $0 | $98,700 |
| Net saving across five years | $0 | $1,289,520 |

**Each nine costs about ten times the implementation and about twice the yearly operation.**
Nygard states both multipliers (Release It! Sec. 13.1, p. 230). Use them for the first
estimate. Replace them with your own measured numbers when you have them.

**The gain in this case is $21,492 a month.** The team spends $98,700 to save $1,289,520.
Nygard calls that a sound financial choice (Release It! Sec. 13.1, p. 230).

**The same test exists in the other book.** Multiply the added availability by the revenue
that the service carries. A move from 99.9% to 99.99% adds 0.09%. On $1M of revenue that is
$900. Invest only when the next nine costs less than $900 (SRE Ch. 3, p. 54).

**Where reliability does not convert into revenue, use the noise floor.** The background error
rate of internet providers falls between 0.01% and 1%. Below that rate your errors sit inside
the noise of the user's own connection (SRE Ch. 3, p. 54).

**Signal to run this step:** a sponsor states a target and no money appears in the same
document.

---

## 3. The interview, one feature at a time

Ask these questions for each feature. Record each answer in the row that `../SKILL.md`,
Table 11 defines.

### Table 3 — The questions and where each answer goes

| Question | Why it changes the number | Where the answer goes | Source |
|---|---|---|---|
| Which business process is this? | It fixes the unit of measurement. | Feature | Release It! Sec. 13.2, p. 231 |
| Does the feature produce revenue directly? | Revenue sets the value of the next nine. | Revenue link | Release It! Sec. 13.1, p. 229 |
| What level of service do users expect? | Expectation, not present performance, sets the target. | Objective | SRE Ch. 3, p. 52 |
| Is the feature paid or free? | A paid feature carries a different tolerance. | Objective | SRE Ch. 3, p. 52 |
| What do competitors offer? | The alternative sets the user's patience. | Objective | SRE Ch. 3, p. 52 |
| Is the audience consumer or enterprise? | An enterprise outage becomes an outage for every dependent business. | Objective | SRE Ch. 3, pp. 52–53 |
| Which external services does the feature call? | Each one lowers the ceiling. | Dependencies | Release It! Sec. 4.10, p. 103 |
| What does the feature do when a dependency fails? | A fallback path raises the achievable number. | Fallback | Release It! Sec. 4.10, p. 104 |

**Ask the dependency question about infrastructure too.** Nygard names the corporate DNS
cluster, the mail transfer service, the message queues and brokers, and the enterprise storage
network (Release It! Sec. 4.10, p. 105). An unlisted dependency still sets the true ceiling.
Report defect `L-27`.

**The ceiling caps the answer.** A feature cannot hold a better agreement than the worst
external dependency inside it (Release It! Sec. 13.2, p. 231). Five external services at 99.9%
each cap the feature at 99.5% (Release It! Sec. 4.10, p. 103). A third party handles the
loyalty club, so that agreement can only pass through the vendor's own agreement, at best
(Release It! Sec. 13.2, p. 231). `03-availability-math.md` holds the arithmetic.

**Signal to stop the interview and reject the target:** the stated objective sits above the
computed ceiling. Report defect `L-26`.

---

## 4. Write the definition of "down"

An agreement that does not define "down" cannot be defended after an incident. Nygard gives
three failures that a loose definition permits (Release It! Sec. 13.2, p. 231).

### Table 4 — Three states that a loose definition calls "available"

| Observed behavior | Why the loose definition passes it | Variable that catches it |
|---|---|---|
| The feature answers after 27.5 minutes | The definition names no time bound | Maximum response time per step |
| The feature answers in 50 ms and returns an error to every user | The definition names no success pattern | Response codes that mean success |
| The feature wobbles, and it looks healthy at each check | The definition names no check frequency | Frequency, and number of locations |

**Answer all eight variables before launch** (Release It! Sec. 13.2, pp. 231–232).

1. How often does the monitoring device run the synthetic transaction?
2. What is the maximum acceptable response time for each step of the transaction?
3. Which response codes or text patterns mean success?
4. Which response codes or text patterns mean failure?
5. How frequently does the device execute the transaction?
6. From how many locations?
7. Where does the device record the data?
8. Which formula computes the percentage — time, or number of samples?

Three more answers make the document usable by an operator.

- **Name the device that monitors the feature, and name how it reports a problem.** Without
  both, nobody knows who receives the signal (Release It! Sec. 13.2, p. 231).
- **Give the synthetic transaction a designated monitoring user id.** Without one, the probe
  pollutes production data (Release It! Sec. 13.2, p. 231). Report defect `L-09`.
- **Write the exclusions for a loss of availability that an external system caused**
  (Release It! Ch. 15, p. 250).

**A person clicking a mouse is not a measurement.** Neither is a count of help desk tickets.
Nygard rejects both. An automated system must run the synthetic transaction (Release It!
Sec. 13.2, p. 231). Report defect `L-08`.

**Signal that the definition is incomplete:** the document answers fewer than eight of the
questions above. Report defect `L-04`.

---

## 5. The requirement must name a delivery mechanism

Load balancing and clustering are the two mechanisms that deliver an availability number.
Nygard names both as prerequisites for high availability (Release It! Ch. 15, p. 250). This
skill does not select between them. It only records that the requirement forces a choice, and
that the choice must happen early.

**Define the high-availability architecture early.** Nygard states that the early decision
makes development and deployment much easier (Release It! Ch. 15, p. 250). A requirement
written after the architecture is fixed can only describe what the architecture already does.

**The one distinction the requirement writer needs.** Load balancing needs no collaboration
between the separate servers. The servers form a cluster when they are aware of each other and
they actively participate in distributing the load (Release It! Sec. 13.4, p. 238). A
load-balanced farm scales close to linearly. A cluster scales less than linearly, so its
capacity can flatten severely as servers are added (Release It! Sec. 13.4, p. 238). Write the
requirement so that the cheaper mechanism can meet it, where the business allows that.

**One trap belongs in the requirement, not in the design.** A load balancer with no health
check distributes traffic, not availability. Name the health check as part of the definition
of "down" in Section 4. Otherwise the probe and the balancer disagree about the same server.

| Question | Owner |
|---|---|
| Which balancing method, and which algorithm? | `capacity-engineering/references/06-load-balancing-and-utilization.md` |
| Where does a load balancer sit in the growth path? | `system-design/references/03-scaling-ladder.md` |
| What does the feature return when a member is dead? | `self-healing-apis/SKILL.md` |

**Signal to revisit this section:** the requirement demands a number that the current
mechanism cannot reach. Stop, and send the number back to the sponsor with the cost.

---

## 6. Join the feature frame to the objective frame

Nygard writes an agreement per feature. The other book writes an indicator, an objective, and
an agreement per service. The two frames merge without loss.

### Table 5 — The merge

| Nygard's item | The matching item | Merge rule |
|---|---|---|
| The feature or business process (Sec. 13.2, p. 231) | The user journey that the indicator represents (SRE Ch. 4, p. 66) | One feature gives one journey. Name it in the Scope block. |
| The eight variables (Sec. 13.2, pp. 231–232) | The six template dimensions (SRE Ch. 4, p. 69) | Both describe the same measurement. Record all eight and all six. Add the timeout. |
| The synthetic transaction (Sec. 13.2, p. 231) | The client measurement point (SRE App. B, p. 572) | Run the probe where the user sits, not on the server. |
| The per-feature agreement (Sec. 13.2, p. 231) | The objective, plus a stated consequence (SRE Ch. 4, p. 65) | Ask what happens when you miss it. No consequence means it is an objective. |
| The dependency ceiling (Sec. 4.10, p. 103) | The source of the number (SRE Ch. 4, p. 71) | The ceiling caps the objective. Record it as the source. |

**Record the timeout beside the latency number.** A timeout truncates the distribution at its
own value. No success can exceed it (SRE Ch. 4, p. 68). Report defect `L-13` when the two
numbers sit in separate files.

**Measure where the user sits.** Server-side collection misses faults that the user feels
(SRE Ch. 4, p. 67). Report defect `L-06`.

**Keep as few objectives as possible.** An objective that you cannot quote to win a priority
argument is not worth holding (SRE Ch. 4, p. 71). This rule and Nygard's per-feature rule do
not conflict. Write one objective for each feature that a person will argue about. Delete the
rest. Report defect `L-05` for sprawl and `L-03` for a single system-wide number.

**Do not copy the target from last month's measurement.** That choice locks the team into
heroic work. The system then cannot improve without a redesign (SRE Ch. 4, p. 71). Report
defect `L-02`.

---

## 7. War stories

Two other stories bear on this file and live elsewhere. `03-availability-math.md`, Section 9,
holds Project Frammitz, and `02-slis-slos-and-slas.md`, Section 8, holds the Global Chubby
planned outage. Read the first before you accept a target. Read the second before you publish
one.

**The blamestorm one year later (Release It! Sec. 13.2, p. 230).** Nygard describes the
sequence. A team takes a word with several fuzzy meanings. It forces an agreement about that
word. It attaches a large amount of money to the word. People then argue about the word a year
or two later. A discussion about the definition one year after launch is almost always a
heated discussion after an incident. By then it is too late. The story proves that the eight
variables belong in the document before launch, not after the first outage.

**The loyalty club pass-through (Release It! Sec. 13.2, p. 231).** A hotel chain runs the
loyalty club subscription through a third party. The chain cannot promise more availability
for that feature than the vendor promises for the service behind it. The best available
agreement is a pass-through of the vendor's own agreement. The story proves that a feature
inherits the number of its worst external dependency. That inheritance becomes visible only
after you split the system into features.

**Gmail moved to the client (SRE App. B, p. 572).** Google measured Gmail error rates and
latency at the client rather than at the server. The measured availability fell at once. The
new number caused changes to both the client code and the server code. Gmail then moved from
about 99.0% available to more than 99.9% available across a few years. The story proves that
the measurement point is part of the requirement. A truthful indicator is the input to the
repair, not the reward for it.

---

## 8. What this file does not decide

| Question | Owner |
|---|---|
| The budget in minutes, and the release policy | `01-risk-and-the-error-budget.md` |
| The indicator, the percentiles, and the six dimensions | `02-slis-slos-and-slas.md` |
| The nines table and the dependency product | `03-availability-math.md` |
| The operational load that the target creates | `04-the-toil-budget.md` |
| The named defects and the index by symptom | `slo-defect-catalog.md` |
| The Circuit Breaker, the timeout, and the fallback path | `self-healing-apis/SKILL.md` |
| The dependency inventory and the SLA-inversion remedy | `self-healing-apis/references/09-vendor-slas-and-degradation.md` |
| The staged rollout and the revert | `reverse-branching/SKILL.md` |
| The load balancer algorithm and the health check design | `capacity-engineering/references/06-load-balancing-and-utilization.md` |
| Where a load balancer sits in the growth path | `system-design/references/03-scaling-ladder.md` |

---

## 9. Proportionality

This file applies when a change adds a user-visible feature, a new external dependency, or a
new availability promise. It does not apply to a change with none of those.

**Write one line and stop when the file does not apply:** `SLO gate: not applicable —
[reason].`

**Do not invent the number.** The availability target is a business decision, not a technical
one (SRE Ch. 1, pp. 26–27). Where the sponsor has not made that decision, record the gap and
stop. A missing number is a finding. A fabricated number is a defect.
