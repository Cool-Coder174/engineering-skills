---
name: slo-engineering
description: Reliability target engine. It converts a wish for reliability into an indicator, an objective, an error budget, and a release policy. Use it when a task names an SLI, an SLO, or an SLA. Use it for an error budget, a burn rate, an availability target, uptime, nines, or downtime. Use it for a latency percentile, p99, a release gate, a change freeze, toil, or on-call load. Use it for a vendor SLA or a dependency ceiling. Use it when a person writes "highly available", "five nines", or "it must never go down". The content comes from Site Reliability Engineering (Beyer, Jones, Petoff, Murphy) and Release It! (Nygard).
---

# SLO ENGINEERING

**ROLE:** You set the reliability target and the policy that enforces it.

**CORE FUNCTION:** You receive a wish. You produce an indicator, an objective, a budget in
minutes, and a rule that says when a release stops.

A reliability requirement fails in a predictable way. Someone writes a percentage. Nobody
writes the formula, the window, or the consequence. A year later an incident starts an
argument that no document can settle. **Write the number before the incident.**

---

# 1. WHEN TO USE THIS SKILL

Use this skill for these tasks:

- A person asks for an availability target, uptime, nines, or a downtime number.
- A document names an SLI, an SLO, or an SLA.
- You define an error budget, a burn rate, a release gate, or a change freeze.
- You choose a latency percentile, such as p50, p95, p99, or p99.9.
- You add a dependency on a vendor, and the vendor states an availability level.
- You measure toil, on-call load, or paging volume.
- A requirement says "highly available", "five nines", or "it must never go down".
- You write the exit criterion that ends an incident.

Do not use this skill for a change with no user-visible path and no new dependency. Do not
use it for the mechanism that produces reliability, which `self-healing-apis` owns. Do not
use it for the mechanism that reverts a change, which `reverse-branching` owns. Do not use it
to diagnose a live fault, which `production-troubleshooting` owns. Do not use it to write an
alert rule or to route a page, which `observability` owns.

**"Not applicable" is a valid result.** Section 12 gives the rule. An honest silence is
better than an invented number.

---

# 2. THE THREE TERMS

People use three words for three different objects. Separate them first.

## Table 1 — The three terms (SRE Ch. 4, pp. 62–65)

| Term | What it is | Who states it | The test |
|---|---|---|---|
| Indicator (SLI) | A measured quantity of one aspect of the service | The engineer | Can you plot it today, from real data? |
| Objective (SLO) | A target value or a range for that indicator | The product owner and the engineer together | Does a number and a window both exist? |
| Agreement (SLA) | A contract that attaches a consequence to the objective | The business team and the legal team | Ask "what happens if we miss it?" A named consequence must exist. |

**No explicit consequence means it is an objective, not an agreement** (SRE Ch. 4, p. 65).
Most people who report an "SLA violation" report a missed objective (SRE Ch. 4, p. 73,
footnote 16).

---

# 3. THE SIX RULES OF A RELIABILITY TARGET

These six rules cause most defects in a reliability requirement. Check every rule on every
target.

### Rule 1. One hundred percent is the wrong target.

The book writes that 100% is probably never the right target (SRE Ch. 3, p. 61). It names the
pacemaker and the anti-lock brake as the notable exceptions (SRE Ch. 1, p. 26). A user on a
99% reliable smartphone cannot tell 99.99% from 99.999% (SRE Ch. 3, p. 48). Avoid the
absolutes "infinitely" and "always" (SRE Ch. 4, p. 71).

### Rule 2. The business owns the number. The engineer owns the arithmetic.

The availability target is a product decision, not a technical one (SRE Ch. 1, pp. 26–27).
The product owner answers three questions. What availability do users expect? What
alternatives do dissatisfied users have? How does usage change at each level?

### Rule 3. Do not copy the target from present performance.

A target that equals last month's measurement locks the team into heroic work. Start with a
loose target that you tighten later (SRE Ch. 4, p. 71).

### Rule 4. Write the objective per feature, not for "the system".

A Circuit Breaker can hold the system "up" while one feature is dead. "The system shall be
available 99.9% of the time" hides that state (Release It! Sec. 13.2, p. 230).

### Rule 5. The objective must sit at or below the dependency ceiling.

"The best you can possibly do is the SLA of the worst of your service providers"
(Release It! Sec. 4.10, p. 103). Section 7 gives the arithmetic.

### Rule 6. Keep as few objectives as possible, and keep each one simple.

An objective that you cannot quote to win a priority argument is not worth holding. A
complicated aggregation hides a change in performance (SRE Ch. 4, p. 71).

---

# 4. DECISION TABLES

Use these tables to make a choice quickly. Read the linked reference before you commit to a
number that a customer will see.

## Table 2 — The indicator, by system category (SRE Ch. 4, p. 66. SRE Ch. 6, p. 88)

| System category | Indicators to define |
|---|---|
| User-facing serving system | Availability, latency, throughput |
| Storage system | Latency, availability, durability |
| Data pipeline or batch job | Throughput, end-to-end latency, per-stage latency |
| Every system | Correctness — was the right answer returned? |

Correctness is a property of the data, not of the infrastructure (SRE Ch. 4, p. 66). Traffic
is not an objective, because users decide it (entry L-10). The `observability` skill owns the
four golden signals of monitoring and the alert routing. This skill owns the objective that
those signals are compared against.

## Table 3 — Where to measure (SRE Ch. 4, p. 63, p. 67. SRE App. B, p. 572. Release It! Sec. 13.2, p. 231)

| Measurement point | What it sees | What it cannot see | Use it when |
|---|---|---|---|
| The server | Server error rate and server latency | The network path, the client library, the browser | You need per-instance attribution |
| The client or the edge | What the user feels | Nothing the user feels | The objective is user-facing. Gmail moved to this point and its measured availability fell. |
| A synthetic transaction | One named feature, from named locations | The real traffic mix | The feature has low traffic, or a per-feature agreement exists |
| The vendor status page | The vendor's own server side | Your path to the vendor | Never as the indicator. Use it only to corroborate. |

The last row is this skill's rule, not a book rule. Neither book discusses a vendor status
page. The row applies the client-side measurement rule of SRE Ch. 4, p. 67 to a vendor call.

## Table 4 — The availability formula (SRE Ch. 3, pp. 50–51. SRE App. A, p. 570)

| Situation | Formula | Reason |
|---|---|---|
| One instance. It is wholly up or wholly down. | Time-based: uptime ÷ total time | The two states are the only states |
| Many replicas, or a load that varies by hour | Aggregate: successful requests ÷ all requests, over a rolling window | A partial outage is invisible to the time formula |
| A batch job, a pipeline, or a webhook consumer | Records processed successfully ÷ records received | There is no continuous request stream |

Worked number. A system serves 2.5M requests in a day. The daily target is 99.99%. The system
may serve up to 250 errors and still meet the target (SRE Ch. 3, p. 50).

## Table 5 — Window, budget, and reset period (SRE App. A, pp. 567–570. SRE App. B, p. 573)

| Objective | Allowed unavailability per month | Allowed per week | Allowed per day | Reset period |
|---|---|---|---|---|
| 99% | 7.2 hours | 1.68 hours | 14.4 minutes | Month |
| 99.5% | 3.6 hours | 50.4 minutes | 7.20 minutes | Month |
| 99.9% | 43.2 minutes | 10.1 minutes | 1.44 minutes | Month |
| 99.95% | 21.6 minutes | 5.04 minutes | 43.2 seconds | Month |
| 99.99% | 4.32 minutes | 60.5 seconds | 8.64 seconds | Month |
| 99.999% | 25.9 seconds | 6.05 seconds | 0.87 seconds | **Quarter** |

Above 99.99%, reset the budget per quarter. The monthly allowance is too small to manage
(SRE App. B, p. 573). Track the number weekly, or even daily. Report to management monthly or
quarterly (SRE Ch. 3, p. 51. SRE Ch. 4, p. 70). `system-design` owns this arithmetic for
capacity sizing. This skill owns it for budget accounting.

## Table 7 — The value of one more nine (SRE Ch. 3, p. 54. Release It! Sec. 13.1, pp. 229–230)

| Step | Question | Worked number |
|---|---|---|
| 1 | What revenue does the service carry over the period? | $1M |
| 2 | What availability does the next nine add? | 99.9% → 99.99% adds 0.09% |
| 3 | What is that increase worth? | $1M × 0.0009 = $900 |
| 4 | What does the next nine cost? | Nygard: about 10× implementation cost, about 2× yearly operation cost |
| 5 | Decide | Invest only when the cost is below the value |

Nygard gives a parallel case. At $1,500 an hour of peak revenue, 98% availability costs
$21,600 a month and 99.99% costs $108. The team spends $98,700 over five years to save
$1,289,520 (Release It! Sec. 13.1, pp. 229–230). Where reliability does not translate into
revenue, compare against the background error rate of internet providers, which falls between
0.01% and 1% (SRE Ch. 3, p. 54).

## Table 8 — Risk tolerance, by service type (SRE Ch. 3, pp. 52–57)

| Service type | Questions to answer before you set the target |
|---|---|
| Consumer service | What availability do users expect? Does the service tie to revenue? Is it paid or free? What do competitors offer? Consumer or enterprise? |
| Infrastructure with mixed clients | Does the client want an empty queue or a full queue? Can you partition into named service levels? What does each level cost? |
| Edge or frontend | Can a lost request be hidden? A request that never reaches the frontend is lost. Engineer the edge to a higher level. |

Success for a low-latency client is failure for a throughput client. Google separated the two
cluster types. The throughput cluster costs 10% to 50% of the low-latency one (pp. 55–56).

## Table 9 — The shape of the failure changes the response (SRE Ch. 3, p. 53)

| Failure shape | Example | Required response | Spends budget? |
|---|---|---|---|
| Partial and cosmetic | Profile pictures do not render | Remediate quickly | Yes |
| Private data exposure | One user sees another user's contacts | Stop the service during debugging and cleanup | Yes |
| Regular scheduled outage | The Ads Frontend outside business hours | Schedule it in advance | **No.** This is planned downtime. |

The same error count can carry a very different business impact. Put the shape of the failure
into the risk conversation, not only the count. The book states the choice as one question.
Which is worse for this service, a constant low rate of failures or an occasional full-site
outage? (SRE Ch. 3, p. 53)

The book grants the maintenance-window exemption to an outage that is occasional, regular and
scheduled (SRE Ch. 3, p. 53). It does not name an announcement. This skill requires the
announcement as well, so that a user who cannot read the schedule still gets notice.

---

# 5. THE ERROR BUDGET AND ITS POLICY

**A budget is one minus the objective** (SRE Ch. 1, p. 27. SRE App. B, p. 573). A 99.99%
objective leaves a 0.01% budget. Convert that fraction to minutes and to events with Table 5.

Follow these four steps (SRE Ch. 3, p. 59).

1. The product owner defines the objective for the period.
2. The monitoring system measures the actual number. It is the neutral third party.
3. The difference between the two numbers is the budget that remains.
4. While budget remains, the team releases.

Worked number: 99.999% of queries per quarter gives a 0.001% budget. A fault that fails
0.0002% of expected queries spends 20% of that budget (SRE Ch. 3, p. 59).

## Table 6 — Budget state to action (SRE Ch. 3, pp. 59–60. SRE App. B, pp. 572–573)

| Budget state | Required action | Who acts | Source |
|---|---|---|---|
| Budget remains | Release. The gate is open. | The product team | SRE Ch. 3, p. 59 |
| Budget close to spent | Slow the release rate. | The product team | SRE Ch. 3, p. 60 |
| A stage shows unexpected behavior | Revert first. Diagnose afterwards. | The release owner | SRE App. B, p. 572 |
| Budget close to spent, and a release is live | Revert the release. | The release owner | SRE Ch. 3, p. 60 |
| Budget spent | Freeze changes. Permit urgent security fixes and fixes for the cause of the errors. | The reliability owner | SRE App. B, p. 573 |
| Budget spent, and the team cannot ship at all | Change the objective, deliberately and in writing. | The product owner | SRE Ch. 3, p. 60 |

1. The two-state form is bang/bang control. The book names the graduated forms as better
   (SRE Ch. 3, p. 60, footnote 15).
2. A fault in the network or in a datacenter also spends the budget. The gate tightens even
   when no code change caused the loss (SRE Ch. 3, p. 60).
3. **The gate needs authority.** The scheme works only when the reliability team can stop a
   launch (SRE Ch. 3, p. 60). A trading firm gives that stop authority to a separate
   enforcement team (SRE Ch. 33, p. 562). Keep the party that stops a release separate from
   the party whose objective is to ship.
4. **Modern.** Neither book describes an alert on the rate of spend. Multi-window burn-rate
   alerting is a later practice. Entry L-39 states the gap.

---

# 6. PER-FEATURE AVAILABILITY REQUIREMENTS

Nygard defines availability per feature, because a Circuit Breaker can leave the system "up"
with one feature dead (Release It! Sec. 13.2, p. 230).

## Table 11 — The per-feature availability row (Release It! Sec. 13.1–13.2, pp. 229–232. Release It! Sec. 4.10, pp. 102–105)

| Column | Content | Example |
|---|---|---|
| Feature | One business process. Never "the system". | Online reservation |
| Revenue link | Direct, indirect, or none | Direct |
| Dependencies | Every one, including DNS, mail transfer, the message broker, and the storage network | Payment vendor 99.9%, corporate DNS 99.9%, message broker 99% |
| Ceiling | (1 − P(internal failure)) × the product of the dependency availabilities | 0.999 × 0.999 × 0.99 ≈ 98.8% |
| Objective | At or below the ceiling | 98.5% |
| Probe | The synthetic transaction, with its eight variables | See below |
| Fallback | What the feature does when the dependency fails | Queue the request. Show a delayed confirmation. |

## The eight questions that define "down"

Answer all eight before launch (Release It! Sec. 13.2, pp. 231–232).

1. How often does the monitoring device run the synthetic transaction?
2. What is the maximum acceptable response time for each step?
3. Which response codes or text patterns mean success?
4. Which response codes or text patterns mean failure?
5. How frequently does the device execute the transaction?
6. From how many locations?
7. Where does the device record the data?
8. Which formula computes the percentage — time or samples?

Three answers protect the probe. Name the device that monitors the feature, and the way it
reports a problem. Give the probe a designated monitoring user id, so that it does not pollute
production data (Release It! Sec. 13.2, p. 231). Write the exclusions for a loss of
availability that an external system caused (Release It! Ch. 15, p. 250).

Three anti-signatures show an incomplete definition (p. 231). A feature answers in 27.5
minutes and counts as available. A feature answers in 50 ms with an error for every user and
counts as available. A feature wobbles, and it looks healthy at every check.

`self-healing-apis` uses the same probe as a fault localization instrument. This skill owns
the definition of "available" that the probe measures.

---

# 7. DEPENDENCY CEILING ARITHMETIC

Compute the ceiling before you write the objective.

```
P(feature up) = (1 − P(internal failure)) × P(dep 1 up) × P(dep 2 up) × …
```

A single failure in any dependency fails the feature (Release It! Sec. 4.10, p. 103). Three
rules decide the number.

1. **Five external services at 99.9% each cap the feature at 99.5%** (p. 103).
2. **A dependency with no stated availability blocks any number.** Project Frammitz met its
   99.99% target "only through sheer luck" (pp. 102–104).
3. **The inventory must include infrastructure** — the corporate DNS cluster, the mail
   transfer service, the message broker, and the storage network (p. 105).

When the objective exceeds the ceiling, apply two responses in this order (p. 104). Decouple
the feature from the lower-availability system and degrade it. Add a Circuit Breaker for each
dependency (Release It! Sec. 5.2, p. 115). Then restate the objective per feature.

`self-healing-apis` owns the full dependency inventory, the three failure layers per
dependency, and the design of the degraded mode. Its file is
`self-healing-apis/references/09-vendor-slas-and-degradation.md`. Read it before you build
the table. This skill owns only the number the table produces, and the objective that number
caps.

**Design to the contracted level, not to the observed level.** Users build on the reality you
offer, not on the promise you publish (SRE Ch. 4, p. 73). Global Chubby produced user-visible
outages because service owners assumed it would never fail (pp. 64–65). Take a planned outage
of a dependency in a drill to find the missing fallback path (p. 65).

---

# 8. THE TOIL BUDGET

Toil is manual, repetitive, automatable, and tactical work. It has no enduring value, and it
grows with the service (SRE Ch. 5, pp. 75–76). It is the operating cost of your objective.

## Table 10 — Toil classification (SRE Ch. 5, pp. 75–78)

| Bucket | Test | Counts toward the 50% cap? |
|---|---|---|
| Software engineering | The person writes or changes code | No |
| Systems engineering | The person configures production and leaves a permanent improvement | No |
| Toil | Manual, repetitive, automatable, tactical, no enduring value, and it grows with the service | **Yes** |
| Overhead | Meetings, hiring, goal setting, training, paperwork | No. Account for it separately. |

1. **Cap operational work at 50% of the team's time**, averaged over a few quarters (p. 77,
   p. 79). Redirect the overflow to the product development team (SRE Ch. 1, p. 25).
2. The first time or the second time you do a task, it is not toil. Unpleasant work with a
   permanent result is grungy work, not toil (pp. 75–76).
3. The on-call rotation sets a floor. A 6-person rotation gives 2 ÷ 6 = 33%. An 8-person
   rotation gives 25% (p. 77).
4. **Expect no more than two paging events per 8-hour to 12-hour shift** (SRE Ch. 1, p. 25.
   SRE App. B, p. 576). A higher rate means a defect in the design, in the monitoring
   sensitivity, or in the response to postmortem items.
5. "It needs human judgment" is not a toil exemption. A service that pages several times a
   day is poorly designed and carries unnecessary complexity (p. 81, footnote 21).

`incident-response` owns the rotation, the escalation path, and the procedure for an
overloaded team. Its file is `incident-response/references/03-on-call.md`. `observability`
owns the alert threshold and the routing. This skill counts the hours those two skills
produce, and charges them against the target.

---

# 9. THE DEFECT GATE (MANDATORY)

Run `references/slo-defect-catalog.md` against every record, every target, and every change to
a release gate. The catalog holds 47 named defects in nine groups. The codes run from L-01 to
L-47 with no gap. Codes L-41 to L-47 arrived after the first eight groups were numbered. Each
one therefore sits at the end of the group it belongs to.

| Group | Content | Codes | Group | Content | Codes |
|---|---|---|---|---|---|
| A | Objective definition | L-01 to L-05, L-41 | F | Dependency arithmetic | L-26 to L-30, L-46 |
| B | Indicator and measurement point | L-06 to L-10, L-42 | G | Toil and operating load | L-31 to L-34, L-47 |
| C | Aggregation and statistics | L-11 to L-15, L-43 | H | Expectation and agreement | L-35 to L-38 |
| D | The error budget | L-16 to L-21, L-44 | I | Modern practice gaps | L-39, L-40 |
| E | Release policy and rollout | L-22 to L-25, L-45 | | | |

Each entry has a **Signature**, a **Consequence**, and a **Fix**. Severity 🔴 defeats the
control itself, or hides a loss the user already suffered. It blocks the merge. 🟡 causes an
outage or a wrong decision under load. 🔵 is a risk to operation or to maintenance.

```md
### Defect Scan
| Code | Present | Evidence | Required fix |
|---|---|---|---|
| L-26 SLA inversion | Yes | broker at 99% under a 99.9% objective | decouple, or restate to 98.9% |
| L-06 Server-side only | No | probe runs at the edge from 3 locations | — |
```

**Report only the defects that apply.** "Not applicable. This change adds no user-visible path
and no dependency." is a valid result, and it is better than an invented finding.
`code-review` calls this catalog for an `L-` scan. The catalog is the source of truth.

---

# 10. THE SLO RECORD

Produce this record when the skill activates. One record covers one feature or one service.
Every line is a commitment that `verify` and `code-review` can check later. The record belongs
in `plan.md` under `## Reliability`, beside the Design Record that `data-systems-design`
writes.

```md
## SLO Record: [feature or service]

### Scope
- Feature: [one business process. Never "the system".] / Not covered: [ ]

### Indicator
| Dimension | Value |
|---|---|
| Quantity | [good events ÷ valid events] |
| Aggregation interval | [averaged over 1 minute] |
| Aggregation region | [all tasks in one cluster] |
| Measurement frequency | [every 10 seconds] |
| Requests included | [HTTP GET from the probe] |
| Data source | [the monitoring system, measured at the client] |
| Latency point | [time to last byte] |
| Timeout in force | [ms] |

> The first six dimensions are the standard template (SRE Ch. 4, p. 69). The last two are
> additions. The timeout censors the distribution (SRE Ch. 4, p. 68).

### Objective
- Target: [99.9% of valid requests succeed] / Window: [rolling 30 days]
- Formula: [aggregate / time-based] — because [ ]
- Internal target: [tighter number] / Advertised target: [ ]
- Source of the number: [the cost test / the user / the dependency ceiling]

### Error budget
- Budget: 1 − objective = [0.1%] / In minutes: [43.2 a month] / In events: [x of y requests]
- Reset period: [month / quarter] / Spent so far: [ ]
- Measured by: [the monitoring system. Not the deploy tool.]

### Budget policy
| Budget state | Action | Who acts |
|---|---|---|
| Remaining | Release | [product team] |
| Close to spent | Slow the release rate | [product team] |
| Burn rate high in a fast window | Revert the last release | [release owner] |
| Spent | Freeze. Permit urgent security fixes and fixes for the cause. | [reliability owner] |
| Exception granted | Tracked item [id], owner [name], reason [ ] |

### Dependency ceiling
| Dependency | Layer that can fail | Stated availability | Fallback | Effect |
|---|---|---|---|---|
| [payment vendor] | transport / naming / protocol | 99.9% | [queue and retry] | ×0.999 |
| [message broker] | transport | No SLA | [none] | **blocks any number** |
- Internal failure probability: [ ] / Computed ceiling: [ ]
- The objective is at or below the ceiling: [yes / no]

### Probe
- Answers to the eight questions of Section 6: [1] [2] [3] [4] [5] [6] [7] [8]
- Monitoring user id: [ ] / Device that monitors, and how it reports: [ ]

### Agreement
- Consequence when the objective is missed: [ ] — if none, this is an objective, not an agreement
- Exclusions: [loss of availability caused by an external system]
- Audience: [internal / named customer / the public]

### Incident exit criterion (form from SRE App. C, p. 577)
- [inside the objective and inside the latency objective for 30 continuous minutes]

### Toil accounting (only when this change adds operational work)
- New manual tasks: [ ] / Hours per week: [ ] / Automation item filed: [id]

### Defect scan
- The table from Section 9, with one row per applicable code.
```

---

# 11. LANGUAGE DISCIPLINE

These phrases carry no mechanism. Each one is forbidden in a record, in a plan, or in a
requirement. Replace it with the required content.

| Forbidden | Must be replaced with |
|---|---|
| "highly available" | The formula, the number, the window, and the measurement point |
| "five nines" | The minutes per month from Table 5, and the cost test from Table 7 |
| "100% uptime", "zero downtime", "it must never go down" | A number below 100%, a window, and a formula (L-01) |
| "the system shall be available X%" | One objective per feature, with the ceiling (L-03) |
| "we have an SLA" | The named consequence when the objective is missed (L-38) |
| "average response time" | The 50th, 95th, 99th, and 99.9th percentiles, and the timeout in force (L-11, L-13) |
| "the vendor is 99.9%" | Your own measurement at the caller edge, and the contracted level (L-29) |
| "we monitor it" | The measurement point, the aggregation interval, and the alert action (L-12) |
| "we will slow down if needed" | The budget state, the actor, and the blocking step in the pipeline (L-21) |
| "the error budget is fine" | The minutes spent, the minutes that remain, and the measuring system (L-17, L-19) |
| "the dependency has never failed" | The contracted level, the fallback path, and the drill that tested it (L-30) |
| "we will automate that later" | The hours per week, the toil bucket, and the tracked automation item (L-31) |

---

# 12. PROPORTIONALITY

Rigor must scale with the blast radius. A full record for a copy change is itself a defect.

## Table 12 — Proportionality

| Change class | Required output from this skill |
|---|---|
| Copy change, refactor, no user-visible path | Nothing. Write one line: `SLO gate: not applicable — [reason].` |
| A new endpoint on a feature that already has an objective | Confirm the objective covers it. Run the defect scan. |
| A new user-facing feature | One feature row (Table 11) plus an indicator, an objective, and a budget |
| A new dependency on an external vendor | A dependency ceiling row, a measurement point, and group F of the catalog |
| A change to a release gate, a freeze rule, or an alert threshold | Sections 5 and 9, plus the budget policy table |
| A new service | Full SLO Record |
| Anything with money, or with an external agreement | Full SLO Record plus the agreement section |

**When this skill stays silent.** This skill stays silent for a change that adds no
user-visible path, no dependency, and no operational work. Write one line:
`SLO gate: not applicable — [reason].` That line is a valid and preferred result.

One hundred percent is the wrong target. So is one hundred percent ceremony. A service with
one feature and one dependency needs one row in Table 11, not a full record. The target is a
business decision, not a technical one (SRE Ch. 1, pp. 26–27). Where the business has not made
that decision, say so and stop. Do not invent a number.

---

# 13. REFERENCE INDEX AND RELATED SKILLS

| Reference | Content | Read it when |
|---|---|---|
| `references/01-risk-and-the-error-budget.md` | Risk, unplanned downtime, the two availability formulas, risk tolerance, the budget and its control loop. SRE Ch. 3, pp. 48–61 | You set a target, or you argue about release velocity |
| `references/02-slis-slos-and-slas.md` | Indicators, objectives, agreements, the measurement point, aggregation, percentiles, the six template dimensions. SRE Ch. 4, pp. 62–73 | You choose an indicator, or you write an objective |
| `references/03-availability-math.md` | The availability table, the aggregate formula, the dependency product, the three failure layers. SRE App. A, pp. 567–570. Release It! Sec. 4.10, pp. 102–105 | You convert a percentage into minutes, or you compute a ceiling |
| `references/04-the-toil-budget.md` | The six attributes, the four buckets, the 50% cap, the safety valve, the on-call floor. SRE Ch. 5, pp. 75–81 | The team has no time to build, or the pager is loud |
| `references/05-gathering-availability-requirements.md` | The cost method, the per-feature agreement, the eight variables, the synthetic transaction. Release It! Sec. 13.1–13.2, pp. 229–232 | You write a requirement per feature, or you define "down" |
| `references/slo-defect-catalog.md` | 47 named defects with a signature, a consequence, and a fix. It also has an index by symptom. | Every review of a target, a budget, or a gate |

**Related skills.** This skill owns the number and the policy. It does not own the mechanism
that produces reliability, and it does not own the mechanism that reverts a change.

| Skill | Who owns the overlap | What this skill does instead |
|---|---|---|
| `engineer-workflow` | It owns routing and the state in `plan.md` | Supplies the gate content. Adds one `## Reliability` block. |
| `planner` | It owns the requirements section | Turns the target into a budget, a window, and a policy |
| `system-design` | It owns the nines table for capacity sizing | Owns the same arithmetic for budget accounting |
| `data-systems-design` | It owns "will the design stay correct" | Owns "what number do we promise, and who stops the release" |
| `security-engineering` | It owns the defense against an attack | Counts attacker-driven unavailability against the budget |
| `detail-planning` | It owns the per-phase spec in `executor.md` | Supplies the indicator, the objective, and the exit criterion |
| `implement` | It owns timeouts, retries, and bounded queues | Supplies the latency and availability numbers those must meet |
| `verify` | It owns the evidence format | Supplies the exit criterion form (SRE App. C, p. 577) |
| `code-review` | It owns the review modes and the report | Supplies the catalog for an `L-` scan |
| `reverse-branching` | It owns how to revert and how to stage a release | Owns the trigger condition — the budget state and the freeze state |
| `self-healing-apis` | It owns the dependency inventory, the SLA-inversion arithmetic, the Circuit Breaker, and the degraded mode | Owns the objective that the computed ceiling caps, and the definition of "available" |
| `observability` | It owns the four golden signals, the alert rule, the threshold, and the routing | Owns the objective those signals are compared against |
| `incident-response` | It owns the rotation, the escalation, the postmortem, and the overloaded team | Owns the exit criterion and the toil accounting |
| `capacity-engineering` | It owns load balancing, the load-balancer algorithm, and the capacity number | Owns the availability number that capacity must deliver |
| `production-troubleshooting` | It owns the diagnosis of a live fault | Owns the number that says the fault matters |
| `systems-programming` | It owns the machine-level cost of a latency number | Consumes that cost as the floor under a latency objective |

---

# 14. GLOSSARY

This skill uses one word for one meaning. Use these words in your output.

| Word | Meaning in this skill |
|---|---|
| **indicator** | A measured quantity of one aspect of the service |
| **objective** | A target value or a range for an indicator, with a window |
| **agreement** | A contract that attaches a named consequence to an objective |
| **budget** | One minus the objective, stated in minutes and in events |
| **window** | The period over which the formula computes the indicator |
| **reset period** | The period after which the budget returns to full |
| **ceiling** | The highest availability that the dependencies permit |
| **probe** | The synthetic transaction that measures one feature |
| **toil** | Manual, repetitive, automatable operational work with no enduring value |
| **revert** | To return the service to the last known-good version |
| **freeze** | A state in which the gate blocks every non-urgent change |
| **measure** | To read a value from the monitoring system. "Estimate" is never a substitute. |

---

**Attribution:** the rules, the terminology, the thresholds, and the page references in this
skill and in its references come from two books. The first is *Site Reliability Engineering:
How Google Runs Production Systems*, by Betsy Beyer, Chris Jones, Jennifer Petoff and Niall
Richard Murphy (O'Reilly Media, 2016). The second is *Release It! Design and Deploy
Production-Ready Software*, by Michael T. Nygard (Pragmatic Bookshelf, 2007). Page numbers are
the PDF page numbers of the extracted source. Content that both books predate carries the tag
**Modern**.
