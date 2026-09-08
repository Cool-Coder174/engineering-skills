---
name: incident-response
description: A method to run an incident with named roles and a written state, and to learn from it without blame. Use this skill when a service breaks now. Use it when a person declares an incident, opens a war room, or asks who the incident commander is. Use it for on-call design, pager load, alert noise, alert fatigue, escalation, severity, handoff, MTTR, and operational overload. Use it after the event for a blameless postmortem, root cause, action items, retrospective, and outage tracking. Trigger words include incident, outage, emergency, page, alert storm, runbook, playbook, on-call, toil, interrupts, and game day. The content comes from Site Reliability Engineering (Beyer, Jones, Petoff, Murphy) and Release It! (Nygard).
---

# INCIDENT RESPONSE

**ROLE:** You are the incident commander. You run the human response to a production fault.

**CORE FUNCTION:** You receive a broken service, or a question about on-call work. You produce
named roles, a written state, a measured exit, and a record that names no person as the cause.

A production fault holds two failures. The machine fails. The response can fail too. The second
failure is the one that people control. The book tells one story twice. In the first telling the
responders freelance, and the outage grows. In the second telling the same fault resolves
overnight (SRE Ch. 14, p. 200–205).

**This skill exists to stop the second failure.**

## How this skill relates to the other skills
| Skill | Question it answers | Relation to this skill |
|---|---|---|
| `incident-response` (this skill) | The service is broken now. Who leads, and what do we write? | Owner of the human response |
| `production-troubleshooting` | Why is production broken? | Owns the diagnosis method. This skill owns the command over it. |
| `observability` | What pages a person, and what only a store holds? | Owns alert design. This skill consumes the page. |
| `slo-engineering` | What is the reliability target and the error budget? | Owns the SLO. This skill uses it as an exit criterion. |
| `reverse-branching` | How do we undo a change safely and fast? | Supplies the mechanism that this skill calls for |
| `self-healing-apis` | Which integrated API is at fault, and what absorbs it? | Supplies the containment before a page |
| `system-design` | What must we build? | Owns the architecture behind the fault |
| `data-systems-design` | Does the design stay correct across machines? | Owns the data-loss hazards behind an incident |
| `security-engineering` | Can an attacker break it? | Owns a compromise. This skill supplies the command structure. |

## The method, in order

1. Ask the three declaration questions. Section 2.
2. Name a commander. Name an operations lead. Section 3.
3. Open the incident record. Name one command post. Section 4.
4. Stop the bleeding. Restore service. Preserve the evidence. Section 5.
5. Diagnose under the command of that commander. Section 6.
6. Transfer command at the end of each shift. Section 7.
7. Close on written exit criteria. Revert every divergence. Section 7.
8. Write the blameless postmortem. Assign every action item. Section 8.
9. Run the incident scan against the record and the runbook. Section 13.

---

# 1. WHEN TO USE THIS SKILL

Use this skill for these tasks:

- A service fails now. A page fires. A customer reports an outage.
- A person asks who leads, who answers stakeholders, or who may change production.
- A person declares an incident, or opens a war room.
- You design or repair an on-call rotation, a shift, or an escalation path.
- You measure pager load, alert noise, alert fatigue, or operational overload.
- You write, review, or close a postmortem. You ask why the same incident returns.
- You review a change to an alert rule, a runbook, a paging config, or a rotation.
- You plan a drill, a game day, or a disaster exercise.

Do not use this skill for these tasks:

- A design question with no live fault. Use `system-design` or `planner`.
- A code defect on one machine, with no user impact. Use `systems-programming`.
- The mechanics of a revert or a staged rollout. Use `reverse-branching`.
- The mechanics of a timeout, a retry, or a Circuit Breaker. Use `self-healing-apis`.
- The method that finds the cause of the fault. Use `production-troubleshooting`.
- The design of a metric, an alert rule, a log line, or a dashboard. Use `observability`.
- The value of an SLI, an SLO, or an error budget. Use `slo-engineering`.

**This skill covers the human response only.** The books state nothing about legal notification,
about regulator reporting, or about customer credits. This skill stays silent on all three.

---

# 2. THE DECLARATION GATE

Ask three questions. One answer of "yes" declares an incident.

## Table 1 — The declaration gate *(SRE Ch. 14, p. 205–206)*
| Question | Answer | Action |
|---|---|---|
| Do you need a second team to fix this? | Yes | Declare an incident |
| Is the outage visible to a customer? | Yes | Declare an incident |
| Is the problem unsolved after one hour of concentrated analysis? | Yes | Declare an incident |
| All three answers are no | — | Work the page. Ask the three questions again later. |

- **Declare early.** A framework that starts hours into a growing problem costs more (SRE Ch. 14, p. 205).
- **Agree these conditions before an event happens** (SRE Ch. 14, p. 205).
- The book gives these three as broad guidelines from one team (SRE Ch. 14, p. 205–206). The
  re-check interval carries no number in either book. It is a local decision.

## Table 2 — Response class by signal *(SRE Ch. 11, p. 161. SRE App. B, p. 573–574)*
| Signal | Class | Response time | Record |
|---|---|---|---|
| Page on a user-facing service | Page | 5 minutes | On-call log |
| Page on a less time-critical service | Page | 30 minutes | On-call log |
| A person must act within a few days | Ticket | Days | A bug |
| No person must act now | Log | None | The log only |

- **Monitoring has three outputs only.** They are pages, tickets, and logs (SRE App. B, p. 573).
- **`observability` owns the alert design.** Its Table 1 decides which output a condition gets.
  This skill consumes the page that the rule produces.
- **Data loss and a monitoring failure are postmortem triggers, not response classes.** Table 8
  holds them, and the book lists them there (SRE Ch. 15, p. 209).
- Severity names such as SEV1 and P1 appear in neither book. Map each one to a row of Table 2. **Modern**

---

# 3. THE FOUR ROLES AND THE COMMAND RULES

The framework comes from the Incident Command System (SRE Ch. 14, p. 202).

## Table 3 — The four roles *(SRE Ch. 14, p. 202–203. SRE App. C, p. 577)*
| Role | The role owns | The role must not do |
|---|---|---|
| Incident commander | High-level state, role assignment, the live document | Change the system |
| Operations lead | Every change to the system, with the delegates of that lead | Answer stakeholders |
| Communications lead | Periodic updates, the summary, the external message | Change the system |
| Planning lead | Bugs, staff, handoffs, the divergence ledger | Change the system |

- **The commander holds every role that the commander did not delegate** (SRE Ch. 14, p. 202).
- **A responder with too much load asks the planning lead for staff.** That responder then delegates (SRE Ch. 14, p. 202).
- A responder who cannot delegate the load creates a subincident (SRE Ch. 14, p. 202).

## Table 4 — Who may change production during an incident *(SRE Ch. 14, p. 201–203, p. 206)*
| Actor | May change production | Rule |
|---|---|---|
| The operations lead | Yes | The operations team is the only group that changes the system |
| A delegate of the operations lead | Yes | The delegate reports every action to that lead |
| The incident commander | No | Unless the commander has not delegated the operations role |
| Any other responder | No | Propose the change to the operations lead |
| A person outside the incident | No | Freelancing made the book's example far worse |
| An automated agent or a bot | No | The operations lead must start it, and the record must name it. **Modern** |

- **Give full autonomy inside the assigned role. Give none outside it** (SRE Ch. 14, p. 206).
- **Name one command post.** Interested parties must know where to reach the commander (SRE Ch. 14, p. 203).
- Log the alerts and the actions into that one channel or room (SRE Ch. 14, p. 203).

---

# 4. THE LIVE INCIDENT STATE DOCUMENT

**The first duty of the commander is the document, not the fix** (SRE Ch. 14, p. 203).

Five rules govern the record.

1. Keep the most important information at the top (SRE Ch. 14, p. 204).
2. Host it outside the failing system. A record on the broken service is unlikely to end well (SRE Ch. 14, p. 203).
3. Never include information that identifies an end user (SRE Ch. 15, p. 211).
4. Update the detailed status at least every four hours, and at every handoff (SRE App. C, p. 577).
5. Add the final record to the team repository of past incidents (SRE Ch. 15, p. 212).

The record holds three parts. Part 1 lives during the event. Part 2 lives from the first
emergency change to the return to norm. Part 3 lives after the event. Seed the Part 3 timeline
from the Part 1 timeline, then add the other entries (SRE App. D, p. 584, footnote 167).

```md
# Incident Record — INC-[number] — [one-line title]

## PART 1 — LIVE INCIDENT STATE
**Summary:** [one sentence. The failure mode and the suspected cause.]
**Status:** active | resolved
**Declared at:** [UTC] **Declared by:** [name]
**Trigger question:** [second team | customer visible | unsolved after one hour]
**Command post:** [one channel or room. It must not run on the failing system.]

### Command hierarchy
| Role | Person | Since (UTC) |
|---|---|---|
| Incident commander | | |
| Operations lead | | |
| Communications lead | | |
| Planning lead | | |
| Next incident commander | | |
> Only the operations lead and the delegates of that lead change the system.

### Detailed status
*Last updated [UTC] by [name]. The communications lead keeps the summary current.*
[What is broken. What is degraded. What is healthy. What we do now.]

### Exit criteria
- [ ] [measurable condition. Example: availability and latency inside the SLO for 30 minutes]
- [ ] [measurable condition]

### Impact so far
| Measure | Value |
|---|---|
| Users affected | |
| Failed operations | |
| Error budget consumed | |
| Revenue or settlement effect | |

### Timeline
*Most recent first. Times in UTC. Every entry names an actor.*
| Time (UTC) | Actor | Entry |
|---|---|---|
| | | |

## PART 2 — RETURN-TO-NORM LEDGER
*The planning lead owns this table. A responder reverts each row when the incident resolves.*
| # | Change made | Actor | Time (UTC) | Reverted? | Reverted at | Bug |
|---|---|---|---|---|---|---|
| 1 | [flag, drain, override, scale, silence, manual edit] | | | No | | |
**Close rule.** The incident does not close while a row says "No" with no owner and no date.

## PART 3 — BLAMELESS POSTMORTEM
**Date:** **Authors:** **Status:** draft | in review | final
**Trigger row:** [which row of Table 8 fired]
**Summary:** [what happened, in two sentences]
**Impact:** [users, duration, failed operations, error budget, money]
**Root causes:** [the contributing causes. Use the 5 Whys. Name the system fault, never a person.]
**Trigger:** [the event that started it]
**Detection:** [what alerted, and how long detection took. State it if a person found it.]
**Resolution:** [what restored the service]

### Action items
| Action item | Type | Owner | Bug | Due | State |
|---|---|---|---|---|---|
| | mitigate / prevent / process / other | | | | |
> An item with no owner and no date is not an action item.
> An item that only reverts the triggering change is not an action item.

### Lessons learned
**What went well:**
**What went wrong:**
**Where we got lucky:** [near misses. Treat each one as a preemptive postmortem.]

### Review
| Criterion | Verdict |
|---|---|
| Key incident data collected | |
| Impact assessment complete | |
| Root cause deep enough | |
| Action plan appropriate, bug priorities correct | |
| Outcome shared with stakeholders | |
**Reviewers:** **Review date:**

### Tracker entry
`cause:` `action:` `bug:` `customer:`

### Supporting information
[Links to logs, graphs, dumps, and the chat transcript. No end-user identifying data. Never.]

### Incident scan
[The table from Section 13.]
```

## Table 5 — Language discipline in the record *(SRE Ch. 12, p. 171. SRE Ch. 15, p. 211. Release It! Sec. 2.3, p. 27–28)*

These phrases are forbidden in the incident record. `production-troubleshooting` Table 11
holds the phrases that are forbidden during the diagnosis. Do not restate that table here.

| Forbidden phrase | Required replacement |
|---|---|
| "the deploy caused it" | The evidence, and the test that ruled out the other causes |
| "we rolled back" | Which artifact, which version, at which time, and whether the symptom changed |
| "root cause: human error" | The system fault that let the action produce the damage |
| "we will monitor it" | The alert name, the threshold, the owner, and the bug number |
| "the alert was noisy" | The count for the week, the disposition, and the tracked change |
| "we fixed it" | The action item, its type, its owner, and its date |
| "it is resolved" | The exit criteria, and the time for which they held |
| "no impact" | The measured user-facing error count, and the error budget consumed |
| "it happened again" | The count in the tracker, and the baseline for that count |

---

# 5. THE RESPONSE ORDER

## Table 6 — The response order *(SRE Ch. 14, p. 206. SRE Ch. 12, p. 173)*
| Order | Action | Signal that the step is complete |
|---|---|---|
| 1 | Stop the bleeding | The error rate stops rising |
| 2 | Restore service | The service meets its SLO |
| 3 | Preserve the evidence | Logs, dumps, and graphs are copied out of the failing system |
| 4 | Find the cause | The postmortem states the contributing causes |

- **Step 4 never precedes step 1.** A responder wants the cause first. "Ignore that instinct" (SRE Ch. 12, p. 173).
- **Rapid triage does not cancel step 3.** Both happen (SRE Ch. 12, p. 173).
- **Restoring service takes precedence over investigation** (Release It! Sec. 2.1, p. 24).
- Collect data only when the collection does not make the outage longer (Release It! Sec. 2.1, p. 24).

## Table 7 — Choose the mitigation move *(SRE Ch. 12, p. 173. SRE Ch. 13, p. 195. SRE App. B, p. 572, p. 575. SRE App. D, p. 582–583. Release It! Sec. 16.9–16.10, p. 262–263)*
| Signal | Move |
|---|---|
| One location fails. Other locations are healthy. | Divert traffic away from that location |
| Total load is above total capacity | Drop a fraction of traffic upstream, retries included |
| One subsystem drives the load | Disable that subsystem |
| A change sits inside the fault window | Revert the change first. Diagnose after. |
| A bug can corrupt data that you cannot recover | Freeze the system |
| Your own automation causes the damage | Disable all team automation |
| One integration point exhausts a resource pool | Set the pool maximum low. Recycle the component. |
| Every cluster fails, and one cluster can absorb the load | Sacrifice one cluster. Drain the rest. |

- **Restore traffic in a ladder.** Use 1, 10, 30, 50, then 100 percent (SRE App. D, p. 583).
- Check the SLO between the steps of that ladder (SRE App. D, p. 583).
- **Restart the component, not the whole server**, when the platform permits it (Release It! Sec. 16.10, p. 263).
- `self-healing-apis` owns the implementation of these moves. This skill owns the decision.
- `production-troubleshooting` Table 4 states which evidence each move destroys. Read it before
  you pick a move.

---

# 6. DIAGNOSIS UNDER COMMAND

**`production-troubleshooting` owns the diagnosis method.** It holds the six steps of SRE
Ch. 12, the technique table, the test-design rules, and the trap catalog. This skill does not
restate them. Read that skill for the method.

This skill owns four rules that govern the method while an incident is open.

1. **The operations lead runs every test.** A test changes production, so Table 4 governs a
   test in the same way that it governs a fix (SRE Ch. 14, p. 202–203).
2. **Every attempt and every result enters the record.** A remedy that did not help is data,
   not an ending (SRE Ch. 14, p. 204–205).
3. **A test that changes the system opens a Part 2 row.** Undo every such change (SRE Ch. 12,
   p. 180).
4. **The commander orders the diagnosis. The commander does not perform it** (SRE Ch. 14,
   p. 203).

One rule from *Release It!* sits at the boundary, so this skill states it and stops. Treat the
last change as a good place to start and a bad place to stop (Release It! Sec. 2.3, p. 27).

---

# 7. HANDOFF, EXIT, AND THE RETURN TO NORM

**The handoff script** (SRE Ch. 14, p. 204).

1. Brief the incoming commander in full. Use a call when that person is remote.
2. State the words: "You are now the incident commander."
3. Wait for firm acknowledgment. Do not leave the call before it.
4. Broadcast the handoff to everyone who works the incident.

- **A handoff without acknowledgment is not a handoff.** Nobody leads in that gap.
- **Write the exit criteria as a checklist inside Part 1.** Each item states a measure and a duration.
- The book's example holds the availability and latency SLOs for 30 or more minutes (SRE App. C, p. 577).
- **An incident does not close on a feeling.** It closes when every item holds for the stated time.
- **The planning lead tracks how the system diverged from the norm** (SRE Ch. 14, p. 203).
- Record one Part 2 row for each emergency flag, override, drain, scale change, silence, and manual edit.
- Each row carries an actor, a time, a revert state, and a bug number.
- The incident stays open while any row says "No" with no owner and no date.

---

# 8. THE BLAMELESS POSTMORTEM

A postmortem is a written record of an incident, its impact, the actions that mitigated it, the
root causes, and the follow-up actions (SRE Ch. 15, p. 208).

## Table 8 — Postmortem triggers *(SRE Ch. 15, p. 209)*
| Trigger | Threshold |
|---|---|
| User-visible downtime or degradation | Above the agreed threshold |
| Data loss | Any amount. No threshold. |
| On-call intervention | Any. A release rollback counts. Traffic rerouting counts. |
| Resolution time | Above the agreed threshold |
| Monitoring failure | Any. A person found the incident, and the monitor did not. |
| A stakeholder asks for one | Always |

- **Agree these criteria before an incident happens** (SRE Ch. 15, p. 209).
- **The blameless test.** Name the contributing causes. Indict no individual and no team (SRE Ch. 15, p. 209).
- Assume that every person held good intentions, and acted correctly on the information available (SRE Ch. 15, p. 209).
- Ask the on-call engineer to write what they thought at each point in time (SRE Ch. 30, p. 507).
- The team then finds where the system misled that person (SRE Ch. 30, p. 507).
- The Bad Apple Theory is demonstrably false (SRE Ch. 30, p. 507).
- **Blameless does not mean toothless.** The record states where the service must improve (SRE Ch. 15, p. 210).

## Table 9 — Action item types *(SRE App. D, p. 579–581)*
| Type | Meaning | Book example |
|---|---|---|
| `mitigate` | Reduces the impact of the next occurrence | Update the playbook with instructions for a cascading failure |
| `prevent` | Removes the cause | Plug the file descriptor leak in the search ranking subsystem |
| `process` | Changes how people work | Schedule a cascading failure test during the next company drill |
| `other` | Everything else | Freeze production until the error budget recovers, or ask for an exception |

- **Every action item carries an owner, a bug number, and a state** (SRE App. D, p. 579–581).
- **Reject an action item that only reverts the triggering change** (SRE Ch. 30, p. 506).
- Replace "Change the streaming timeout back to 60 seconds" with an item that explains the delay (SRE Ch. 30, p. 506).
- **Rescope an item that is too extreme or too costly.** The book names these "knee-jerk" items (SRE App. D, p. 583).

**The review.** Senior engineers assess the draft against five criteria (SRE Ch. 15, p. 211).
Did the team collect the key data. Are the impact assessments complete. Was the root cause deep
enough. Is the action plan appropriate. Did the team share the outcome.

- **An unreviewed postmortem might as well never have existed** (SRE Ch. 15, p. 211).
- After the review, add the record to the repository of past incidents (SRE Ch. 15, p. 212).

---

# 9. ON-CALL LOAD AND ALERT HYGIENE

**`observability` owns the alert rule.** Its Table 1 decides which output a condition gets. Its
Table 13 decides which rule to remove. The production meeting asks two questions of every
paging event. Should it have paged in that way, and should it have paged at all (SRE Ch. 31,
p. 515). Take the disposition from `observability`, and record it as a tracked bug.

## Table 10 — Alert handling inside an open incident *(SRE Ch. 11, p. 166–167. SRE Ch. 29, p. 501)*
| Observation during the incident | Action |
|---|---|
| One event produces several alerts | Group the related alerts. Target one alert per incident. |
| An alert is duplicate or uninformative | Silence it for the incident, so the on-call can focus |
| A silence outlives the incident | Record it with a named owner and a fix date |
| An alert paged, and the on-call took no action | Open a bug against the rule. `observability` decides the disposition. |

- **Every paging alert must be actionable** (SRE Ch. 11, p. 166).
- **Every paging alert must align with a symptom that threatens the SLO** (SRE Ch. 11, p. 166).
- **Never route alerts to email.** Email is the moral equivalent of `/dev/null` (SRE App. B, p. 574).
- False positives train the team to ignore the alarm (Release It! Sec. 16.8, p. 261).

## Table 11 — On-call load limits *(SRE Ch. 11, p. 160–168. SRE Ch. 29, p. 499. SRE App. C, p. 577)*
| Limit | Value | Source |
|---|---|---|
| Operational work per engineer | 50 percent maximum | SRE Ch. 11, p. 160 |
| On-call time per engineer | 25 percent maximum | SRE Ch. 11, p. 162 |
| Incidents per 12-hour shift | 2 maximum | SRE Ch. 11, p. 163 |
| Cost of one incident, end to end | About 6 hours | SRE Ch. 11, p. 163 |
| Median paging events per day | 0 | SRE Ch. 11, p. 163 |
| Daily tickets | Fewer than 5 | SRE Ch. 11, p. 166 |
| Alerts per incident | 1 | SRE Ch. 11, p. 167 |
| On-call team, one site | 8 engineers minimum | SRE Ch. 11, p. 162 |
| On-call team, two sites | 6 engineers per site | SRE Ch. 11, p. 162 |
| On-call rotation exposure | Once or twice per quarter per engineer | SRE Ch. 11, p. 168 |
| Project work during an on-call week | None | SRE Ch. 29, p. 499 |
| Status update cadence during an incident | At least every 4 hours | SRE App. C, p. 577 |

A local team can set a different number. State the local number. State that the book gives
another one. Do not present a local number as a rule from the book.

`observability` Table 14 states the alert subset of these numbers as an alert load budget.
`slo-engineering` owns the error budget and the release policy that the budget enforces.

---

# 10. OUTAGE TRACKING AND TRENDS

Postmortems cover the large incidents only. Events that are individually small, frequent, and
widespread stay outside their scope (SRE Ch. 16, p. 216). An outage tracker receives every
alert, and closes that gap.

## Table 12 — Outage tracker tags *(SRE Ch. 16, p. 219)*
| Namespace | Use | The book's example |
|---|---|---|
| `cause:` | What produced the incident | `cause:network`, `cause:network:switch` |
| `action:` | What a responder did | A suggested prefix. The book prints no example. |
| `bug:` | The tracked follow-up | `bug:76543` |
| `customer:` | The affected counterparty | `customer:132456` |
| `bogus` | A false positive | `bogus` |

- **A colon separates the levels of the namespace** (SRE Ch. 16, p. 219).
- **Do not fix the tag list in advance.** Teams that choose their own tags produce better data (SRE Ch. 16, p. 219).

Analyze in three layers (SRE Ch. 16, p. 220).

1. Count. Report incidents per week, per month, and per quarter. Report alerts per incident.
2. Compare across teams, across services, and across time.
3. Read the meaning. Name the component that causes the most incidents.

- **Count incidents per day and alerts per day as two separate numbers** (SRE Ch. 16, p. 218).
- **A raw count means nothing without a baseline** (SRE Ch. 16, p. 220).
- **Alert the owner of a dependency when their own alerting stays silent** (SRE Ch. 16, p. 221).

---

# 11. INTERRUPTS AND OPERATIONAL OVERLOAD

Operational load holds three forms. They are pages, tickets, and ongoing responsibilities
(SRE Ch. 29, p. 493). Toil is manual, repetitive, automatable work. It holds no enduring value,
and it scales with the service (SRE Ch. 5, p. 75).

- **A person is on interrupts, or on projects. Never both** (SRE Ch. 29, p. 499).
- A 20-minute interruption costs about two hours of productive work (SRE Ch. 29, p. 498).

## Table 13 — The overload ladder *(SRE Ch. 11, p. 166–167. SRE Ch. 29, p. 499–501. SRE Ch. 30, p. 503–508)*
| Step | Action | Signal to take the next step | Source |
|---|---|---|---|
| 1 | Write the SLO first | No SLO exists for the service | SRE Ch. 30, p. 508 |
| 2 | Quantify the symptoms. Set quarterly objectives. | Load stays above the cap | SRE Ch. 11, p. 166 |
| 3 | Fix the monitoring. Remove the unactionable pages. | Load stays above the cap | SRE Ch. 11, p. 166–167 |
| 4 | Make tickets a full-time role for one or two people | Load stays above the cap | SRE Ch. 29, p. 499–500 |
| 5 | Scrub the interrupt classes. Silence each one until its fix date. | No owner accepts the fix | SRE Ch. 29, p. 501 |
| 6 | Transfer exactly one experienced engineer into the team | The practice of the team does not change | SRE Ch. 30, p. 503 |
| 7 | Set common goals with the application developers | The developers keep adding noise | SRE Ch. 11, p. 167 |
| 8 | Route some paging alerts to the developer on-call | The service still misses the standard | SRE Ch. 11, p. 167 |
| 9 | Return the pager to the developer team | — | SRE Ch. 11, p. 167 |

- **Step 1 is not optional.** Without an SLO, no later step helps (SRE Ch. 30, p. 508).
- **Transfer one engineer at step 6.** Two engineers can make the team defensive (SRE Ch. 30, p. 503).
- **The embedded engineer does not fix the issues.** That engineer finds the work and explains the permanent fix (SRE Ch. 30, p. 508).
- That engineer then reviews the change, and repeats the loop two or three times (SRE Ch. 30, p. 508).
- **Steps 8 and 9 are temporary.** They hold while both teams return the service to the standard (SRE Ch. 11, p. 167).
- `references/03-on-call.md` and `references/06-interrupts-and-overload.md` carry the same nine steps.

---

# 12. PRACTICE AND DRILLS

**Incident management proficiency atrophies quickly when the framework is not in constant use**
(SRE Ch. 14, p. 206). A team that is out of practice makes a long outage out of a short one
(SRE App. B, p. 576).

| Drill | Cadence | Source |
|---|---|---|
| Run the incident framework during routine change management | Every change | SRE Ch. 14, p. 206 |
| Rotate the roles between incidents | Every incident | SRE Ch. 14, p. 207 |
| Disaster role playing, 30 to 60 minutes | Weekly | SRE Ch. 28, p. 486 |
| Wheel of Misfortune. Reenact a past postmortem. | Regular | SRE Ch. 15, p. 212–213 |
| Test the rollback path in a test environment | Before a large-scale change | SRE Ch. 13, p. 191 |
| Company disaster and recovery testing | Yearly | SRE Ch. 33, p. 553 |

- **A response path that nobody exercises is not a response capability.**
- Prefer a controlled failure with the best people present over a failure at 2 a.m. on a Saturday (SRE Ch. 13, p. 198).

---

# 13. THE INCIDENT SCAN (MANDATORY GATE)

Run `references/incident-failure-catalog.md` against the artifact before you accept it. The
catalog holds 52 named failure modes, `N-01` to `N-52`. Each entry gives a signature, a
consequence, and a fix. Run the scan against four artifacts. They are a closed incident record,
a runbook, an on-call rotation design, and a change to an alert rule or a paging config.

Report in this form:

```md
### Incident Scan
| Code | Present | Evidence | Required fix |
|---|---|---|---|
| N-09 No divergence ledger | Yes | 3 flags set at 14:02, no revert list | Open Part 2 and list each flag |
| N-15 Untested rollback path | No | Rollback test recorded 2026-08-14 | — |
```

List only the codes that apply. **"Not applicable" is a valid and preferred result.** An honest
"not applicable. This page closed in four minutes with no production change." beats an invented
finding.

---

# 14. PROPORTIONALITY

Rigor scales with the size of the event. A full record for a routine page is itself a failure
mode.

## Table 14 — Proportionality *(the classes are **Modern**. The thresholds inside them carry the citations above)*
| Class of event | Required output |
|---|---|
| A page that a runbook closes in minutes, with no user impact | One on-call log line |
| A page that needs a production change | The log line, plus a divergence ledger entry |
| A declared incident under Table 1 | Incident Record, Parts 1 and 2 |
| Any trigger in Table 8 | Incident Record, Parts 1, 2, and 3 |
| A repeat of an incident class already in the tracker | Part 3, plus a trend query over the tracker |
| A question about on-call load, alert noise, or interrupts | Sections 9 and 11 only |
| A design question with no live fault | Nothing. Use `system-design` or `planner`. |

**This skill stays silent** in four cases. No page fired. No user saw a fault. No responder
changed production. No person asks about on-call work. It also stays silent on legal
notification, on regulator reporting, and on customer credits. Neither book covers those three.

---

# 15. GLOSSARY

This skill uses one word for one meaning. Use these words in your output.

| Word | Meaning in this skill |
|---|---|
| **incident** | An event that Table 1 declares |
| **page** | A notification that demands action now |
| **ticket** | A notification that demands action within a few days |
| **revert** | The verb. To return an artifact to a previous version. |
| **rollback** | The noun. The act of reverting, as a recorded event. |
| **transfer command** | The verb. To move the commander role to a named person. |
| **handoff** | The noun. The transfer of a role at a shift change. |
| **stop** | To end a process or a job. This skill never writes "shut down". |
| **record** | The written incident document, in three parts |
| **divergence** | A change that moves the system away from its normal state |
| **toil** | Manual, repetitive, automatable work with no enduring value |
| **error budget** | 1 minus the SLO, over the agreed period |
| **postmortem** | Part 3 of the record. This skill never writes "retrospective". |

---

# 16. REFERENCE INDEX AND RELATED SKILLS

| Reference | Content | Read it when |
|---|---|---|
| `references/01-incident-command.md` | Roles, command post, live state document, handoff, the declaration gate | A fault needs more than one person |
| `references/02-emergency-response.md` | The three emergency classes and what each one teaches | A system breaks now |
| `references/03-on-call.md` | Shift design, the numeric limits, safety under stress, overload | You design or fix a rotation |
| `references/04-postmortem-culture.md` | Triggers, the blameless rule, the template, the review, adoption | You write or review a postmortem |
| `references/05-tracking-outages.md` | Escalator, Outalator, grouping, tagging, analysis, reporting | You need trends, not one story |
| `references/06-interrupts-and-overload.md` | Pages, tickets, ongoing work, polarized time, embedding | The team produces nothing |
| `references/incident-failure-catalog.md` | 52 named failure modes, `N-01` to `N-52`, with an index by symptom | Every scan and every review |

## Who owns each overlap
| Overlap | Owner | What this skill keeps |
|---|---|---|
| The six-step diagnosis method, the technique table, the test design | `production-troubleshooting` | The command over the method, and the record of each test |
| Metric design, alert rule shape, log contract, dashboards | `observability` | The page as an input, and the alert handling during an incident |
| SLI, SLO, error budget, burn rate, release freeze | `slo-engineering` | The SLO, used as an exit criterion |
| How to revert a commit, a config, or a release | `reverse-branching` | The decision to revert now, and the record of the attempt |
| Canary, staged rollout, automatic rollback | `reverse-branching` | Nothing. Table 7 links to it. |
| Timeouts, retries, backoff, jitter, Circuit Breaker, bulkheads | `self-healing-apis` | Nothing. Table 7 links to it. |
| Load shedding and graceful degradation mechanics | `self-healing-apis` | The move, not the implementation |
| Isolation, replication, idempotency, dual writes | `data-systems-design` | Nothing. A data-loss incident cites its hazard catalog. |
| Threat modelling, compromise, and privilege | `security-engineering` | The command roles and the record |
| Capacity estimation and the load test | `capacity-engineering` | Nothing. A saturation incident cites it. |
| Architecture and the component boundaries | `system-design` | Nothing |

## Invocation rules

1. `engineer-workflow` does not schedule this skill. A page or a declaration starts it. **Modern**
2. `code-review` runs Section 13 against a change to an alert rule, a runbook, or a rotation.
3. `verify` runs Section 13 against a closed record, before that record reaches the repository.
4. `planner` reads the Part 3 action items. Each item becomes a phase or a bug.
5. `detail-planning` must produce a tested rollback for any phase that this skill can page on.
6. When a fault localizes to an integrated API, transfer the technical work to `self-healing-apis`.
7. When the incident is a compromise, transfer the technical work to `security-engineering`.
8. When the cause is unknown, transfer the technical work to `production-troubleshooting`.
9. In rules 6, 7, and 8, keep the command structure and the record in this skill.

---

**Attribution:** the rules, the numbers, and the page references in this skill and in its
references come from two books. The first is *Site Reliability Engineering: How Google Runs
Production Systems*. Its authors are Betsy Beyer, Chris Jones, Jennifer Petoff, and Niall
Richard Murphy. O'Reilly Media published it in 2016. The second is *Release It! Design and
Deploy Production-Ready Software*, by Michael T. Nygard. The Pragmatic Bookshelf published it
in 2007. Page numbers refer to those editions. Content that both books predate carries the tag
**Modern**, and this skill never attributes such content to a book.
