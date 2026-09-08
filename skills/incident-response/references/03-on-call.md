# Being On-Call

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Chapter 11, "Being On-Call", p. 160–168. Supporting pages: Ch. 14, p. 206. Ch. 15, p. 209.
Ch. 28, p. 486. Ch. 29, p. 493–502. Ch. 30, p. 503–511. App. B, p. 571–576. App. D, p. 581.

This file answers one question. **What makes a rotation that a person can hold for years?**
Read it in four cases. You design a rotation. A team reports pager load. A responder decides
badly under stress. A team produces no engineering work. For the roles inside a live
incident, read `01-incident-command.md`. Every number below carries its page. A number that
no page supports is a local decision, and this file says so.

---

## 1. The life of an on-call engineer

The on-call engineer holds two duties (SRE Ch. 11, p. 161). The engineer manages the outages
that affect the team. The engineer also performs or checks production changes. **A page takes
priority over almost every other task, including project work** (SRE Ch. 11, p. 161). Lower
priority alerts and software releases are not pages, and the engineer handles those during
business hours.

### 1.1 Response time and the paging path

| Service class | Response time | Source |
|---|---|---|
| User-facing, or otherwise highly time-critical | 5 minutes | SRE Ch. 11, p. 161 |
| Less time-sensitive | 30 minutes | SRE Ch. 11, p. 161 |

**The response time follows from the availability target. It is not a preference.** A system
at 99.99% availability has about 13 minutes of allowed downtime per quarter. The reaction
time must then be on the order of minutes. For a more relaxed SLO, tens of minutes are enough
(SRE Ch. 11, p. 161).

**Signal that your response time is wrong.** The team agreed a 30-minute response for a
service whose SLO permits 13 minutes of quarterly downtime. Derive the number again from the
SLO, which `system-design` owns.

Google supplies the page-receiving device, and dispatches a page by email, SMS, robot call,
or app (SRE Ch. 11, p. 161). **Modern.** A paging vendor with schedules and an acknowledgment
timer performs the same function. Severity names such as SEV1 and P1 appear in neither book,
so a severity scheme also carries the tag **Modern**.

### 1.2 Primary and secondary rotations

| Design | Duty of the secondary | Cost | Source |
|---|---|---|---|
| Fall-through | Answer the pages that the primary misses | Needs a second rotation | SRE Ch. 11, p. 161–162 |
| Split duties | Handle every non-urgent production activity | Needs a second rotation | SRE Ch. 11, p. 162 |
| Paired teams | Two related teams cover each other | Each team learns two services | SRE Ch. 11, p. 162 |

**Staff each shift with a single engineer** (SRE Ch. 29, p. 494). One engineer per shift
limits the interruption to the team. It also prevents the bystander effect.

---

## 2. Balanced on-call

An SRE manager owns two axes, and must keep both sustainable (SRE Ch. 11, p. 162). Quantity
is the percent of engineer time that on-call duty consumes. Quality is the number of
incidents that occur during one shift.

### 2.1 Balance in quantity

**Google caps purely operational work at 50 percent of an engineer's time**
(SRE Ch. 11, p. 160). The remainder splits again (SRE Ch. 11, p. 162).

| Slice | Share | Source |
|---|---|---|
| Engineering projects | At least 50 percent | SRE Ch. 11, p. 162 |
| On-call duty | No more than 25 percent | SRE Ch. 11, p. 162 |
| Other operational, nonproject work | Up to 25 percent | SRE Ch. 11, p. 162 |

**The team size follows from the 25 percent rule. It is arithmetic, not preference.**

| Team shape | Minimum engineers | Source |
|---|---|---|
| One site | 8 | SRE Ch. 11, p. 162 |
| Two sites | 6 per site, well separated | SRE Ch. 11, p. 162 and App. B, p. 576 |

The single-site number assumes two people on-call and week-long shifts
(SRE Ch. 11, p. 162). Each engineer then holds the pager one week per month. **Choose one
site or many sites from the trade-off, not from habit** (SRE Ch. 11, p. 163).

| Option | Gain | Cost |
|---|---|---|
| Multi-site, "follow the sun" | Removes night shifts, which harm health [Dur05] | Communication and coordination overhead |
| Single site | No cross-site overhead | Night shifts remain |

### 2.2 Balance in quality

The book defines an **incident** as a sequence of events and alerts with the same root cause,
discussed in one postmortem (SRE Ch. 11, p. 163).

**One incident costs about 6 hours, end to end** (SRE Ch. 11, p. 163). That figure covers
root-cause analysis, remediation, the postmortem, and the bug fixes. **A 12-hour shift
therefore carries at most 2 incidents** (SRE Ch. 11, p. 163). App. B states the same ceiling
(SRE App. B, p. 576).

**The distribution of paging events must be flat, with a likely median of 0**
(SRE Ch. 11, p. 163). A component that pages every day pushes the median above 1. The book
predicts the consequence. Something else will break, and the shift cannot absorb it.

**More than two events per shift points at three suspects** (SRE App. B, p. 576). They are
the design of the system, the sensitivity of the monitoring, and the response to postmortem
bugs. **If a quarter exceeds the limit, establish corrective measures for the next quarter**
(SRE Ch. 11, p. 163). Use the ladder in Section 5. Code: `N-37`.

### 2.3 Compensation

Google offers time off in lieu, or straight cash, capped at some proportion of salary
(SRE Ch. 11, p. 163–164). **In practice the cap limits how much on-call work one person will
accept** (SRE Ch. 11, p. 164). The proportion is a local decision, with no number in the book.

---

## 3. Feeling safe

Research names two ways of thinking that a person may choose [Kah11] (SRE Ch. 11, p. 164).

| Mode | Description | Result under a complex outage |
|---|---|---|
| Intuitive | Automatic and rapid action | Less likely to produce a good result |
| Deliberate | Rational, focused cognition | More likely to produce well-planned handling |

**The second mode is more likely to produce better results** (SRE Ch. 11, p. 164). So the
team must reduce the stress of being on-call, because stress selects the first mode. Stress
hormones such as cortisol and CRH cause fear (SRE Ch. 11, p. 164). Fear impairs cognitive
function and causes suboptimal decisions [Chr09]. Under those hormones the deliberate
approach yields to immediate action, and the responder abuses heuristics.

### 3.1 The heuristic trap and the two costs of intuition

The book gives one worked example. The same alert pages for the fourth time in a week, and an
external infrastructure system caused the previous three. **It is extremely tempting to
attribute the fourth page to the same cause** (SRE Ch. 11, p. 165). That is confirmation bias.

**Intuition can be wrong, and obvious data often does not support it**
(SRE Ch. 11, p. 165). The responder then loses time on a wrong line of reasoning. **Quick
reactions grow from habit, and a habitual response is unconsidered** (SRE Ch. 11, p. 165).
The book states that such a response can be disastrous.

The book keeps the hedge. Intuition and quick reaction can seem desirable during incident
management. The ideal method balances two acts. The responder moves at the desired pace once
enough data supports a decision, and examines the assumptions at the same time
(SRE Ch. 11, p. 165). Code: `N-20`.

### 3.2 The written procedure is the remedy

The book names the three most important on-call resources (SRE Ch. 11, p. 165).

| Resource | What it removes | Owner file |
|---|---|---|
| Clear escalation paths | The fear of asking for help | Section 3.3 below |
| Well-defined incident-management procedures | The need to invent structure under stress | `01-incident-command.md` |
| A blameless postmortem culture | The fear of punishment | `04-postmortem-culture.md` |

**A written procedure replaces a decision that a stressed person would otherwise improvise.**
The protocol offers an easy-to-follow and well-defined set of steps (SRE Ch. 11, p. 165).
Google supports it with a web-based tool that automates role transfer and status updates. The
incident manager then spends no effort on formatting email (SRE Ch. 11, p. 165–166).
**Proficiency atrophies quickly when the framework is not in constant use**
(SRE Ch. 14, p. 206). A procedure that nobody practices is not a capability. Read Section 6.

### 3.3 When to escalate, and when to adopt the protocol

| Signal | Action | Source |
|---|---|---|
| The outage is serious and has significant unknown dimensions | Escalate to the developer on-call | SRE Ch. 11, p. 165 |
| The issue is complex enough to involve multiple teams | Adopt the formal incident-management protocol | SRE Ch. 11, p. 165 |
| After some investigation you cannot estimate an upper bound for the time span | Adopt the formal incident-management protocol | SRE Ch. 11, p. 165 |

The developer teams of SRE-supported systems usually hold a 24/7 rotation, so escalation is
always possible. **Appropriate escalation is a principled reaction, not a failure**
(SRE Ch. 11, p. 165).

### 3.4 After the incident

**The team writes a postmortem after a significant incident, with a full timeline**
(SRE Ch. 11, p. 166). The postmortem focuses on the events, not on the people. For the
triggers and the template, read `04-postmortem-culture.md`. The book adds one design
instruction here. **Recognizing automation opportunities is one of the best ways to prevent
human errors** (SRE Ch. 11, p. 166).

---

## 4. Avoiding inappropriate operational load

Operational load falls into three categories (SRE Ch. 29, p. 493). Pages are production
alerts and their consequences, and they carry an SLO of minutes. Tickets are customer
requests that need an action within hours, days, or weeks. Ongoing responsibilities, such as
a rollout, carry no defined SLO and still interrupt.

### 4.1 Alert disposition

Monitoring has three outputs only. They are pages, tickets, and logs (SRE App. B, p. 573).

**`observability` owns the disposition of an alert rule.** Its Table 1 chooses the output. Its
Table 13 chooses which rule to remove. Do not restate those tables here. This file states only
the two dispositions that a rotation review needs.

| Observation | Disposition | Source |
|---|---|---|
| It pages, and the on-call takes no action | Remove the unactionable page | SRE Ch. 31, p. 515 |
| It fires often, and nobody has diagnosed it | Investigate it fully, or fix the rule | SRE Ch. 30, p. 505 |

**Never route alerts to email.** The book calls that practice the moral equivalent of
`/dev/null` (SRE App. B, p. 574). The strategy works for a while, and it relies on eternal
human vigilance. The inevitable outage is then more severe. Codes: `N-26`, `N-28`.

### 4.2 The two rules for a paging alert

**A paging alert must align with a symptom that threatens an SLO** (SRE Ch. 11, p. 166).
**Every paging alert must be actionable** (SRE Ch. 11, p. 166).

Misconfigured monitoring is a common cause of operational overload (SRE Ch. 11, p. 166). A
low-priority alert that interrupts the on-call engineer every hour disrupts productivity. The
fatigue that such an alert induces makes the team treat a serious alert with less attention
than it needs (SRE Ch. 11, p. 166). Code: `N-24`. For the monitoring design itself, and for
the four golden signals, read `observability`.

**Make the symptoms of overload measurable, so the goals can be quantified**
(SRE Ch. 11, p. 166). Daily tickets stay below 5, and paging events per shift stay below 2.

### 4.3 Alert fan-out

One abnormal condition can generate several alerts (SRE Ch. 11, p. 167). Three moves control
the fan-out.

1. Group the related alerts in the monitoring system (SRE Ch. 11, p. 167).
2. Silence a duplicate or uninformative alert during the incident (SRE Ch. 11, p. 167).
3. Tune a noisy alert toward a **1:1 alert-to-incident ratio** (SRE Ch. 11, p. 167).

**Record every silence as a tracked item with an owner and a fix date** (SRE Ch. 29, p. 501).
Silence the interrupt only until the root cause is expected to be fixed. That rule relieves
the handler and sets a deadline for the fixer. Codes: `N-25`, `N-29`.

### 4.4 One person is on interrupts, or on projects, and never both

**A person should never be expected to be on-call and also make progress on projects**
(SRE Ch. 29, p. 499). Do not plan project work for an on-call week. If a project cannot slip
by one week, that engineer must not hold the pager. A 20-minute interruption entails two
context switches, and costs about two hours of productive work. **Assign a cost to context
switches** (SRE Ch. 29, p. 498).

**Polarize the time.** Each engineer must know on arrival whether the day holds only project
work or only interrupts (SRE Ch. 29, p. 498). One week is the ideal period, and a day or a
half day may be more practical (SRE Ch. 29, p. 498).

**Do not assign tickets at random across the team** (SRE Ch. 29, p. 499–500). Make tickets a
full-time role for a manageable period. If the load exceeds one person, use two people.
Code: `N-38`. Read `06-interrupts-and-overload.md`.

---

## 5. Operational overload and the error-budget lever

Overload starts when operational activities exceed the 50 percent cap (SRE Ch. 11, p. 166).
Code: `N-40`. Apply the steps in order. Each row states the signal that sends you to the next
row.

| Step | Action | Signal to take the next step | Source |
|---|---|---|---|
| 1 | Write the SLO first | No SLO exists for the service | SRE Ch. 30, p. 508 |
| 2 | Quantify the symptoms. Set quarterly objectives. | Load stays above the cap | SRE Ch. 11, p. 166 |
| 3 | Fix the monitoring. Remove the unactionable pages. | Load stays above the cap | SRE Ch. 11, p. 166–167 |
| 4 | Make tickets a full-time role for one or two people | Load stays above the cap | SRE Ch. 29, p. 499–500 |
| 5 | Scrub the interrupt classes. Silence each one until its fix date. | No owner accepts the fix | SRE Ch. 29, p. 501 |
| 6 | Transfer exactly one experienced engineer into the team | The practice of the team does not change | SRE Ch. 11, p. 166 and Ch. 30, p. 503 |
| 7 | Set common goals with the application developers | The developers keep adding noise | SRE Ch. 11, p. 167 |
| 8 | Route some paging alerts to the developer on-call | The service still misses the standard | SRE Ch. 11, p. 167 |
| 9 | Return the pager to the developer team | — | SRE Ch. 11, p. 167 |

**Step 1 is not optional. Without an SLO, no later step helps** (SRE Ch. 30, p. 508). The SLO
gives a quantitative measure of the impact of an outage.

**Transfer one engineer at step 6, not two.** Two engineers do not necessarily produce a
better result, and the team can react defensively (SRE Ch. 30, p. 503). **That engineer does
not fix the issues.** Fixing them reinforces the idea that making changes is for other people.
The engineer finds one-person work, then explains how that work fixes a postmortem issue
permanently. The engineer reviews the change, and repeats the loop two or three times
(SRE Ch. 30, p. 508).

**More tickets must not require more engineers.** A team that answers growth by adding
administrators is in **ops mode** (SRE Ch. 30, p. 504). The SRE model adds humans only when
the system adds complexity (SRE Ch. 30, p. 504).

**Step 9 is the book's extreme case, and the book names it "give back the pager"**
(SRE Ch. 11, p. 167). The developer team then holds the pager alone until the service meets
the standard of the SRE team. This happens rarely (SRE Ch. 11, p. 167). Steps 8 and 9 are
temporary measures while both teams prepare the service for onboarding again.

### 5.1 The error budget

**`slo-engineering` owns the error budget, the burn rate, and the release policy.** Read it for
the definition, the reset period, and the freeze rule. This file states only what the ladder
consumes.

The error budget makes step 7 a data question rather than an argument.

**Use the budget as the reason, and state it.** The book's model sentence pushes back on a
release because the error budget for releases is exhausted. It does not push back because the
tests are bad (SRE Ch. 30, p. 509). The budget removes the structural tension between the SRE team and the
product development team (SRE App. B, p. 573). The same balance appears in Ch. 11 as the
ability to renegotiate on-call responsibility (SRE Ch. 11, p. 167).

The worked example freezes production pushes for one month after an incident exceeded the
budget by several orders of magnitude (SRE App. D, p. 581). The record files an owned action
item to seek an exception. **A named person approves an exception. Nobody assumes one.**

---

## 6. Operational underload

**A too-quiet system is also a defect.** The book calls operational underload "a treacherous
enemy" (SRE Ch. 11, p. 168). Long periods away from production produce two failures. The
engineer becomes overconfident or underconfident. The knowledge gaps appear only when an
incident occurs (SRE Ch. 11, p. 168).

| Countermeasure | Cadence | Source |
|---|---|---|
| Size the team so every engineer is on-call | Once or twice per quarter | SRE Ch. 11, p. 168 |
| Wheel of Misfortune role play | Weekly, 30 to 60 minutes | SRE Ch. 11, p. 168 and Ch. 28, p. 486 |
| DiRT, the company disaster recovery event | Annual, over several days | SRE Ch. 11, p. 168 |

Code: `N-39`. Read `02-emergency-response.md` for the drill formats.

---

## 7. War stories

**The fourth page in a week (SRE Ch. 11, p. 165).** The same alert pages a fourth time in one
week, and an external infrastructure system caused the previous three pages. The book states
that it is extremely tempting to associate the fourth occurrence with the previous cause. The
responder then follows a line of reasoning that was wrong from the start. It proves that a
recurring alert corrodes diagnosis quality.

**The eight-engineer minimum (SRE Ch. 11, p. 162).** The book applies the 25 percent on-call
rule with two people on-call and week-long shifts. A single-site team then needs at least
eight engineers, and each holds the pager one week per month. A dual-site team needs at least
six per site. It proves that rotation size is a derived number, not a staffing preference.

**"Give back the pager" (SRE Ch. 11, p. 167).** An SRE team may ask the developer team to
hold the pager alone until the system meets the SRE standard. The book reports that this
happens rarely, because it is almost always possible to reduce the load with the developer
team. It also states when the move is correct. Complex or architectural changes across
multiple quarters can be necessary. It proves that on-call ownership depends on a standard.

**Out of practice (SRE App. B, p. 576, and App. D, p. 581).** A team that implements these
best practices makes incidents rare, and the team then loses fluency. The named consequence
is "making a long outage out of a short one". The worked postmortem records the same defect
in its own "What went wrong" section. It proves that drills, not incidents, must supply the
practice.

---

## 8. Cross-references, and when this file stays silent

| Question | File |
|---|---|
| Who leads a live incident, and what do we write? | `01-incident-command.md` |
| What do the three emergency classes teach? | `02-emergency-response.md` |
| When do we write a postmortem, and how do we review it? | `04-postmortem-culture.md` |
| How do we see trends instead of one story? | `05-tracking-outages.md` |
| How do we return a team from overload? | `06-interrupts-and-overload.md` |
| Which named failure applies to this rotation? | `incident-failure-catalog.md` |
| Where do the SLO and the error budget come from? | `slo-engineering/SKILL.md` |
| Which output does an alert rule get, and which rule must go? | `observability/SKILL.md` |

Codes that a rotation review runs: `N-20`, `N-24`, `N-25`, `N-26`, `N-28`, `N-29`, `N-37`,
`N-38`, `N-39`, `N-40`.

**State "not applicable" and stop in three cases.** A fault is live now. The question
concerns a written postmortem. The question concerns the design of the system. The owner file
above answers each one. **Do not apply a rotation number to a team that runs no rotation.**
These numbers describe a 24/7 on-call team. A one-person project with no pager needs none.
