# Interrupts and Operational Overload

**Sources:** *Site Reliability Engineering: How Google Runs Production Systems* — Beyer, Jones,
Petoff, Murphy (O'Reilly, 2016). SRE Ch. 29 "Dealing with Interrupts", p. 493–502. SRE Ch. 30
"Embedding an SRE to Recover from Operational Overload", p. 503–511. Supporting pages: SRE
Ch. 5, p. 75–81. SRE Ch. 11, p. 160–168. SRE Ch. 31, p. 514–516. SRE App. B, p. 575–576.

This file answers one question. The team answers every page, closes every ticket, and finishes
no engineering work. What do you change?

**The answer is structural.** The book locates the cause in team policy and team structure, not
in personal discipline (SRE Ch. 29, p. 497). A free calendar does not make an engineer
undistractible (SRE Ch. 29, p. 498).

Read `03-on-call.md` for shift design. Read `05-tracking-outages.md` for the data that proves
the load. This file changes how the team divides the work, and how it returns from overload.

---

## 1. Operational load has three classes

**Operational load is the work that keeps the system in a functional state** (SRE Ch. 29,
p. 493). The book divides it into three classes. Each class interrupts a person differently.

| Class | What it is | Response time | Source |
|---|---|---|---|
| Page | A production alert and its fallout | An SLO, sometimes in minutes | SRE Ch. 29, p. 493 |
| Ticket | A customer request that needs an action from you | An SLO in hours, days, or weeks | SRE Ch. 29, p. 493 |
| Ongoing responsibility | Code pushes, flag rollouts, ad hoc time-sensitive questions | No defined SLO | SRE Ch. 29, p. 493 |

**The third class carries no SLO and still interrupts.** The book also names it "toil" (SRE
Ch. 29, p. 493). Toil is manual, repetitive, automatable, tactical work with no enduring value,
and it grows as the service grows (SRE Ch. 5, p. 75–76).

**Interrupts are the first-ranked source of toil.** The book ranks three sources. Interrupts
come first. On-call response comes second. Releases and pushes come third (SRE Ch. 5, p. 77).

**Much of the load is unplanned.** It arrives at a nonspecific time. The receiver must then
decide whether the issue can wait (SRE Ch. 29, p. 493–494).

**Modern.** A chat mention, a bot alarm, and a coding agent that asks for approval belong to
the same classes. Count them, and route them like any other interrupt.

---

## 2. Why an interrupted engineer produces nothing

The book uses cognitive flow to explain the cost. Flow needs clear goals, immediate feedback, a
sense of control, and the time distortion that follows (SRE Ch. 29, p. 495–496).

**A disruptive interrupt removes a person from flow** (SRE Ch. 29, p. 495). The book states the
goal plainly. Maximize the time a person spends in that state.

| Flow state | How the engineer reaches it | What destroys it |
|---|---|---|
| Creative and engaged | Sustained attention on one problem the engineer can solve | A disruptive interrupt (SRE Ch. 29, p. 496) |
| "Angry Birds" | Full-time attention on interrupts, with clear goals such as closing bugs | Project work, which now acts as the distraction (SRE Ch. 29, p. 496–497) |

**The symmetry is the point.** When a person concentrates on interrupts full-time, interrupts
stop acting as interrupts. The project then becomes the distraction (SRE Ch. 29, p. 497).

**A 20-minute interruption costs about two hours.** The interruption entails two context
switches. The book states the realistic loss as a couple of hours of productive work (SRE
Ch. 29, p. 498). The book's instruction is short. "Assign a cost to context switches"
(SRE Ch. 29, p. 498).

**Keep the book's hedge.** The ideal balance of project time and interrupt time varies from
engineer to engineer. Some engineers do not know which balance motivates them (SRE Ch. 29,
p. 497).

---

## 3. Polarize the time

**Rule. A person arrives at work and knows one thing. That day holds only project work, or only
interrupts** (SRE Ch. 29, p. 498).

**Period.** The book prefers a week. It accepts a day or a half-day as more practical (SRE
Ch. 29, p. 498). The exact period is a local decision inside that range.

**Rule. A person is never on-call and also expected to make progress on projects** (SRE Ch. 29,
p. 499). The same rule covers any other work with a high context-switch cost.

**Rule. Cancel the project commitments for an on-call week** (SRE Ch. 29, p. 499). When a
project cannot slip by a week, that person does not take the shift. Escalate, and assign the
shift to someone else.

**Rule. Be on interrupts, or do not be** (SRE Ch. 29, p. 500). A person who works tickets while
unassigned appears busy and performs worse. That person also skews the numbers. The queue can
stay unmanageable while the team believes one handler is enough (SRE Ch. 29, p. 500).

This rule is the fix for **N-38** in `incident-failure-catalog.md`.

---

## 4. Assign each class of interrupt to a named role

| Class | Who holds it | Rule | Source |
|---|---|---|---|
| Pages | One primary on-call engineer per shift | A single engineer per shift limits team interruption and avoids the bystander effect | SRE Ch. 29, p. 494 |
| Page backup | A secondary on-call engineer | Duties vary by rotation. The secondary may sit on another team. | SRE Ch. 29, p. 494 |
| Tickets | A dedicated ticket handler, for a bounded period | Never assign tickets at random | SRE Ch. 29, p. 499–500 |
| Ongoing responsibilities | A named role such as push manager | Anyone on the team can hold the role. Formalize the handover. | SRE Ch. 29, p. 500 |

**Rule. Scale by adding one person to a class, not by spreading the class over the team**
(SRE Ch. 29, p. 499). The on-call engineer can also transfer work to the secondary, or
downgrade a page to a ticket.

**Rule. Decide the secondary duties from their weight** (SRE Ch. 29, p. 499). A secondary whose
only duty is to contact the primary for an unanswered page can still do project work. A
secondary who helps the primary under high pager volume does interrupt work.

**Rule. Never assign tickets at random.** The book calls random assignment disrespectful of the
team's time. It causes the context switches that destroy flow time (SRE Ch. 29, p. 500).

**Rule. Use at most two ticket handlers.** The primary and the secondary may not close the
queue. Then put exactly two people on tickets at any time. Do not spread the load over the
whole team (SRE Ch. 29, p. 500).

**Rule. A quiet pager does not release the on-call engineer.** Give that engineer cleanup work
that a page can interrupt at any moment. Documentation and configuration cleanup never end
(SRE Ch. 29, p. 499).

**Rule. No person shepherds one change for its whole lifetime.** Define the procedure for a
push or a flag flip. Then define a push manager role for the duration of one interrupt period.
Formalize the handover (SRE Ch. 29, p. 500).

---

## 5. Reduce the number of interrupts

**Signal to act.** The interrupt load needs too many people staffing interrupts at the same time
(SRE Ch. 29, p. 500–501). The book names the antipattern. A rotation runs like a gauntlet. The
handler survives the shift, returns to regular duties, and never finds a root cause (p. 501).

| Observation about a recurring interrupt | Required action | Source |
|---|---|---|
| The team runs on-call handoffs but no ticket handoff | Create a ticket handoff that carries shared state between handlers | SRE Ch. 29, p. 501 |
| Nobody has examined the class of interrupt | Run a regular scrub of tickets and pages. Find the root cause. | SRE Ch. 29, p. 501 |
| The root cause is fixable in a reasonable time | Silence the interrupt until the expected fix date, and record the date | SRE Ch. 29, p. 501 |
| The step is slow but needs no privilege of yours | Use policy. Return the work to the requester for your review. | SRE Ch. 29, p. 502 |
| Neither the developer team nor the operations team has diagnosed the alert | Investigate the alert fully, or fix the alerting rule. There is no third option. | SRE Ch. 30, p. 505 |
| The on-call engineer took no action on the page | Remove the page, or change how it pages | SRE Ch. 31, p. 515 |
| Nobody will give attention to the root causes | Return the pager, deprecate the component, or replace it | SRE Ch. 29, p. 502 |

**Rule. A silence carries a date.** The silence relieves the interrupt handler. The date creates
a deadline for the person who fixes the cause (SRE Ch. 29, p. 501). A silence with no date and
no owner is **N-29** in `incident-failure-catalog.md`.

**Rule. Policy is a tool of the same class as code** (SRE Ch. 29, p. 502). Use policy to move
effort that needs no privilege of yours back to the customer.

**The customer contract that a policy must respect** (SRE Ch. 29, p. 502). The request must be
meaningful. The request must be rational. The request must carry the information and the
legwork that you need. In return, your response must be helpful and timely.

**Rule. A chronically flaky component with no developer attention is a candidate for return.**
Measure the value of the time you spend on its interrupts. Then consider the pager, the
deprecation, or the replacement (SRE Ch. 29, p. 502).

---

## 6. The signals of operational overload

| Measure | Limit | Source |
|---|---|---|
| Operational work per engineer | 50 percent maximum | SRE Ch. 11, p. 160. SRE Ch. 5, p. 77. |
| On-call time per engineer | 25 percent maximum | SRE Ch. 11, p. 162 |
| Incidents per 12-hour shift | 2 maximum | SRE Ch. 11, p. 163. SRE App. B, p. 576. |
| Cost of one incident, end to end | About 6 hours | SRE Ch. 11, p. 163 |
| Median paging events per day | 0 | SRE Ch. 11, p. 163 |
| Daily tickets | Fewer than 5 | SRE Ch. 11, p. 166 |
| On-call team, one site | 8 engineers minimum | SRE Ch. 11, p. 162. SRE App. B, p. 576. |
| On-call team, two sites | 6 engineers per site | SRE Ch. 11, p. 162. SRE App. B, p. 576. |
| Project work during an on-call week | None | SRE Ch. 29, p. 499 |

**The rotation sets a floor under the toil number.** Assume one week primary and one week
secondary per cycle. A 6-person rotation then floors at 33 percent. An 8-person rotation floors
at 25 percent (SRE Ch. 5, p. 77). Compare the measurement against that floor, not against zero.

**A published baseline exists.** Google quarterly surveys measured an average near 33 percent
toil, with outliers from 0 percent to 80 percent (SRE Ch. 5, p. 77–78). An average hides the
individual who carries 80 percent.

**Rule. A breach of a limit is a signal, not a verdict.** When the incident limit fails for a
quarter, install corrective measures that return the load to a sustainable state (SRE Ch. 11,
p. 163). More than two events per shift also points at the system design, at the monitoring
sensitivity, or at unclosed postmortem bugs (SRE App. B, p. 576).

These measures are the signatures of **N-37** and **N-40** in `incident-failure-catalog.md`.

---

## 7. Embed exactly one engineer

**Signal to act.** The daily ticket volume rises. The team gives a disproportionate share of
its time to tickets instead of to the service. Scalability and reliability then suffer (SRE
Ch. 30, p. 503).

**Rule. Transfer exactly one engineer.** Two engineers do not necessarily produce a better
result. Two engineers can make the receiving team defensive (SRE Ch. 30, p. 503).

**Rule. The embedded engineer improves the team's practices.** That engineer does not empty the
ticket queue (SRE Ch. 30, p. 503).

**Rule. More tickets must not require more engineers.** The book states the model directly.
"more tickets should not require more SREs" (SRE Ch. 30, p. 504). Add people for complexity,
never for volume.

### Phase 1 — Learn the service and get context (SRE Ch. 30, p. 503–506)

1. Articulate why each habit helps or harms the scalability of the service.
2. Test the claim "my service is tiny". Shadow an on-call session, because scale changes the
   strategy.
3. Prepare a new service for growth. A 100 request per second service can reach 10,000 requests
   per second in a year.
4. Rank the outages by their effect on the team's stress. A very small outage can produce
   disproportionate stress.
5. Identify the kindling. Section 8 lists the sources.

### Phase 2 — Share context (SRE Ch. 30, p. 506–507)

6. Do not review the postmortem archive and leave comments. That exercise makes the team
   defensive.
7. Own the next postmortem instead. An outage will happen during the embedding. Write the
   blameless record with the on-call engineer.
8. Reject the Bad Apple Theory out loud. Evidence from several disciplines, airline safety
   included, shows the theory to be false (SRE Ch. 30, p. 507).
9. Sort every fire into toil or not-toil. Present the list to the team. Explain each entry as
   work to automate, or as acceptable overhead.

### Phase 3 — Drive change (SRE Ch. 30, p. 507–510)

10. **Write the SLO first.** Without that agreement, no other step in the chapter helps (SRE
    Ch. 30, p. 508). The SLO gives a quantitative measure of outage impact.
11. **Do not fix the issues yourself.** A personal fix reinforces the belief that other people
    make the changes (SRE Ch. 30, p. 508).
12. Run the delegation loop. Find useful work for one team member. Explain how the work fixes a
    postmortem issue permanently. Review the code and the document changes. Repeat for two or
    three issues (SRE Ch. 30, p. 508).
13. Record every further issue in a bug report or a document for the team.
14. Explain every decision, whether or not a person asks. A team that sees weak reasoning copies
    that reasoning (SRE Ch. 30, p. 509).
15. Ask leading questions, not loaded questions. A team in ops mode rejects first-principles
    reasoning from its own members, so model it (SRE Ch. 30, p. 510).

### Close-out (SRE Ch. 30, p. 511)

16. Write an after-action report. Restate the perspective, the examples, and the explanation.
    Add action items. Organize it as a *postvitam*, which explains the critical decisions that
    produced the success.
17. Stay available for design reviews and code reviews.
18. Watch the team for the next few months. Confirm improvement in capacity planning, in
    emergency response, and in rollout processes.

---

## 8. Kindling — the emergencies that wait

**Kindling is an emergency that has not happened yet** (SRE Ch. 30, p. 505). Find the kindling
after you rank the team's existing fires. A new subsystem that cannot manage itself is one
common source. The book names these others.

| Kindling source | Why it burns | Source |
|---|---|---|
| Knowledge gaps from overspecialization | The specialist lacks the breadth for on-call. Teammates then ignore critical parts. | SRE Ch. 30, p. 505 |
| A service that the operations team built and that quietly grows in importance | It escapes the scrutiny that a feature launch receives | SRE Ch. 30, p. 505 |
| Strong dependence on "the next big thing" | The team ignores a problem for months and never applies the temporary fix | SRE Ch. 30, p. 505 |
| A common alert that nobody has diagnosed | Responders label it transient. It distracts them from real problems. | SRE Ch. 30, p. 505 |
| A complained-about service with no SLI, SLO, or SLA | No quantitative basis exists to judge impact or to set priority | SRE Ch. 30, p. 505 |
| A capacity plan that says "add more servers" | A load test that passes at 1.99 GB against a 2 GB model proves nothing | SRE Ch. 30, p. 505–506 |
| A postmortem whose only action items revert the change | The change goes away. The defect stays. | SRE Ch. 30, p. 506 |
| "We do not know anything about that. The developers own it." for a serving-critical component | The team cannot give acceptable on-call support | SRE Ch. 30, p. 506 |

**The on-call knowledge floor.** For every serving-critical component, the responder must know
two facts. Know the consequence when it breaks. Know the urgency that a fix needs (SRE Ch. 30,
p. 506).

The undiagnosed alert is **N-28**. The revert-only action item is **N-33**. Both appear in
`incident-failure-catalog.md`.

---

## 9. The overload ladder

Take the steps in order. Take the next step only when the signal repeats.

This ladder is the same one that `../SKILL.md` Table 13 and `03-on-call.md` Section 5 hold. The
step numbers match across all three.

| Step | Action | Signal to take the next step | Source |
|---|---|---|---|
| 1 | Write the SLO first | No SLO exists for the service | SRE Ch. 30, p. 508 |
| 2 | Quantify the symptoms. Set quarterly objectives. | The load stays above the cap | SRE Ch. 11, p. 166 |
| 3 | Fix the monitoring. Remove the unactionable pages. | The load stays above the cap | SRE Ch. 11, p. 166–167 |
| 4 | Make tickets a full-time role for one or two people | The load stays above the cap | SRE Ch. 29, p. 499–500 |
| 5 | Scrub the interrupt classes. Silence each interrupt until its fix date. | No owner accepts the fix | SRE Ch. 29, p. 501 |
| 6 | Transfer exactly one experienced engineer into the team | The team's practice does not change | SRE Ch. 30, p. 503 |
| 7 | Set common goals with the application developers | The developers keep adding noise | SRE Ch. 11, p. 167 |
| 8 | Route some paging alerts to the developer on-call | The service still misses the standard | SRE Ch. 11, p. 167 |
| 9 | Return the pager to the developer team | — | SRE Ch. 11, p. 167 |

**Rule. Steps 8 and 9 are temporary.** They hold while both teams return the service to the
standard that permits a handover again (SRE Ch. 11, p. 167).

**Rule. Step 6 fails without step 1.** Write the SLO before the embedded engineer starts phase 3
(SRE Ch. 30, p. 508). `slo-engineering` owns that SLO.

---

## 10. War stories

**Fred's Monday (SRE Ch. 29, p. 497–498).** Fred is neither on-call nor on interrupts. He wants
project work. He gets a coffee, wears headphones, and sits at his desk. Then five things can
happen. An automated system assigns him a ticket that is due today. A colleague on-call
interrupts him about a component he knows well. A user raises the priority of a ticket that Fred
received last week during his on-call shift. A flag rollout that runs over three or four weeks
goes wrong, and Fred must examine it and revert the change. A user contacts Fred directly,
because Fred is helpful. Fred has a free calendar and remains extremely distractible. Team
policy and assumptions about ongoing responsibilities cause this result, so the fix is
structural.

**Running the gauntlet (SRE Ch. 29, p. 501).** On a large team a person may hold interrupts only
once every couple of months. That person survives the shift, feels relief, and returns to regular
duties. The successor then does the same. Nobody investigates the root causes of the tickets.
The team makes no forward movement, and a succession of people gets annoyed by the same issues.
The book's remedy has two parts. Create a ticket handoff that carries shared state. Then run a
regular scrub of tickets and pages to find the root cause of each class.

**The streaming timeout action item (SRE Ch. 30, p. 506).** The book gives a one-line example of
a defective postmortem outcome. The action item reads "Change the streaming timeout back to 60
seconds". The correct item asks why the first megabyte of a promotional video sometimes takes 60
seconds. A postmortem whose only output reverts the triggering change is kindling. The change
goes away and the behavior stays, so the same class of outage returns.

**The Bad Apple Theory postmortem (SRE Ch. 30, p. 506–507).** A team that treats postmortems as
punishment answers a postmortem request with "Why me?". That attitude follows from the Bad Apple
Theory, which holds that the system works and that removing the people who make mistakes keeps
it working. The book calls the theory demonstrably false and cites evidence from several
disciplines, airline safety included. The embedded engineer names that falsity, then asks the
on-call engineer to write what they were thinking at each point in time. The purpose is to find
where the system misled the responder, and where the cognitive demands were too high.

**The routing config explanation (SRE Ch. 30, p. 509).** An embedded engineer objects to every
server generating its own routing configuration. The weak reason is "we cannot see it". The
teachable reason names two mechanisms. A bug in that code can cause a correlated failure across
the service. The extra code is a source of bugs that can slow a rollback. A team that hears weak
reasoning copies weak reasoning after the embedded engineer leaves.

---

## 11. Proportionality

| The question in front of you | What this file owes you |
|---|---|
| A service breaks now | Nothing. Read `01-incident-command.md` and `02-emergency-response.md`. |
| One page arrived and a runbook closed it | Nothing |
| The team asks who handles tickets this week | Sections 3 and 4 |
| The pager fires more than the limits in Section 6 permit | Sections 5, 6, and 9 |
| An engineer reports that they finish no project work | Sections 2, 3, and 4 |
| The team has stayed above the 50 percent cap for a quarter | Sections 6 through 9 |
| A single noisy alert | `05-tracking-outages.md` for the count, then `observability` for the disposition |

**This file stays silent for a live fault.** Operational load is a quarter-scale measurement. Do
not open this file during an incident.

---

## 12. Ownership of the overlaps

| Overlap | Owner | What this file keeps |
|---|---|---|
| Shift length, rotation size, and stress safety | `03-on-call.md` | The numeric limits as overload signals |
| Alert disposition and the page-or-ticket decision | `observability` | Step 3 of the ladder, and the silence-with-a-date rule |
| Counting the interrupts and finding the trend | `05-tracking-outages.md` | The interpretation of the count against a baseline |
| The blameless record and its action items | `04-postmortem-culture.md` | The revert-only action item as kindling |
| Writing the SLO and the error budget | `slo-engineering` | The consumption of the SLO as step 1 of the ladder |
| Turning a postmortem action item into a plan | `planner` | The delegation loop of phase 3 |

Seven codes in `incident-failure-catalog.md` cover this file. They are **N-24**, **N-28**,
**N-29**, **N-33**, **N-37**, **N-38**, and **N-40**. Run them against any change to a
rotation, to a ticket policy, or to an alert rule.
