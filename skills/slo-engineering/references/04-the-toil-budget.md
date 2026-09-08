# Toil and the Fifty Percent Rule

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 5 "Eliminating Toil", p. 75–81. Ch. 1 "Introduction", p. 23–26. App. B "A Collection of
Best Practices for Production Services", p. 575–576. Part II introduction, p. 47.
*Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007). Sec. 4.11, p. 106–108.
Sec. 14.4, p. 248. Ch. 15, p. 250.

Read this file when the team reports no time to build anything. Read it when the pager is
loud. Read it when a change adds a manual step to production.
`01-risk-and-the-error-budget.md` governs the release. This file governs the people. The two
budgets are separate. A team can hold its error budget and still fail this one.

---

## 1. Three words that people confuse

**Toil is not "work I do not like to do".** The book separates three kinds of unwelcome work
(SRE Ch. 5, p. 75).

| Term | Test | Counts as toil? |
|---|---|---|
| Toil | Work tied to running a production service. It matches the attributes in Section 2. | **Yes** |
| Overhead | Administrative work not tied to running a production service. Team meetings, goal setting and goal grading, snippets, HR paperwork. | No. Record it in a separate bucket. |
| Grungy work | Unpleasant work that leaves a permanent improvement. The book's example is a cleanup of the whole alerting configuration. | No. |

**Personal taste does not decide the class.** Some people enjoy manual repetitive work
(SRE Ch. 5, p. 75). The attributes of the work decide the class. The feelings of the person
do not. **Signal that starts this scan:** a person says "operational work" without a number.
The book replaced that phrase with the word *toil* for exactly this reason (p. 75).

---

## 2. The six attributes of toil

Toil is work tied to running a production service. It tends to be manual, repetitive,
automatable, tactical, and without enduring value. It also grows in proportion to the
service (SRE Ch. 5, p. 75).

| # | Attribute | The test | The trap it closes |
|---|---|---|---|
| 1 | Manual | A human spends hands-on time on the task. | A person runs a script by hand. The script does not remove the toil. Count the hands-on time, not the elapsed time (p. 75–76). |
| 2 | Repetitive | The person does the task again and again. | The first time, and even the second time, is not toil (p. 76). |
| 3 | Automatable | A machine could do the task as well as the human, or a design could remove the need. | When human judgment is essential, there is a good chance the task is not toil (p. 76). Section 10 tests that claim. |
| 4 | Tactical | The work is interrupt-driven and reactive, not strategy-driven. | Handling pager alerts is toil. The book does not promise full removal. It requires continuous work toward a minimum (p. 76). |
| 5 | No enduring value | The service is in the same state after the task ends. | Grunt work inside a permanent improvement is not toil, even when the grunt work is large (p. 76). |
| 6 | O(n) with service growth | The work grows in proportion to service size, traffic volume, or user count. | A well designed service grows by at least one order of magnitude with no extra work, except one-time work to add resources (p. 76). |

**Not every task that you call toil has all six attributes.** The more attributes the work
matches, the more likely it is toil (SRE Ch. 5, p. 75). Keep the hedge. Score the task
against all six. Do not argue about one attribute alone.

**Attribute 6 decides the budget.** Attributes 1 to 5 describe a task. Attribute 6 describes
a trend. A task that grows with the user count converts every sales win into more load.

---

## 3. What qualifies as engineering

**Engineering work is novel and intrinsically requires human judgment.** It produces a
permanent improvement in the service, and a strategy guides it (SRE Ch. 5, p. 78).

| Category | What the person does | Counts toward the engineering floor? |
|---|---|---|
| Software engineering | Writes or changes code, with the design and the documentation. Automation scripts, tools, frameworks, features for scale and reliability. | Yes |
| Systems engineering | Configures production, or documents it, so that one-time work leaves a lasting improvement. Monitoring configuration, load balancer configuration, OS tuning, architecture consulting for a developer team. | Yes |
| Toil | Work tied to running the service that is repetitive and manual. | No. It counts against the 50% cap. |
| Overhead | Hiring, HR paperwork, meetings, bug queue hygiene, peer reviews, self-assessments, training. | No. Record it separately. |

**The test in one line: does the work let the same staff hold a larger service?** Engineering
work helps a team handle more services at the same staffing level (SRE Ch. 5, p. 78).

---

## 4. Why less toil is better

**Toil expands when nobody checks it.** Left unchecked, it can quickly fill 100% of the time
of everyone on the team (SRE Ch. 5, p. 77). This is the reason for a hard number.

**An operations team grows in proportion to the service.** When the product succeeds, the
operational load grows with the traffic. The company then hires more people to do the same
tasks again and again (SRE Ch. 1, p. 23).

**Engineering work is what lets the team grow more slowly than the service.** Engineering is
the reason the organization scales sublinearly with service size (SRE Ch. 5, p. 77).

**The 50% goal is also a promise to new hires.** Google quotes the rule during hiring, and
must then keep it (SRE Ch. 5, p. 77). A broken promise is a named harm. Section 11 lists it.

**Removing the human from the release can reduce toil and raise reliability at the same
time** (SRE Part II, p. 47). `reverse-branching/SKILL.md` owns that machinery.

---

## 5. The fifty percent rule

**Cap operational work at 50% of the time of each engineer.** The named cap covers aggregate
ops work — tickets, on-call, and manual tasks (SRE Ch. 1, p. 23). App. B repeats it. A team
spends no more than 50% of its time on operational work (SRE App. B, p. 575).

**The cap is an upper bound, not a target** (SRE Ch. 1, p. 23). The stated end state is a
service that basically runs and repairs itself. The book wants systems that are automatic,
not only automated. Scale and new features keep the load above zero in practice (p. 23).

**Spend at least 50% of the time on engineering project work** (SRE Ch. 5, p. 77). That
project work either reduces future toil or adds service features (p. 77).

**Average the number over a few quarters or a year** (SRE Ch. 5, p. 79). Toil is spiky. A
steady 50% is not realistic for every team, and a team can fall below the target in some
quarters.

| Window | What it tells you | Correct action |
|---|---|---|
| One week | Nothing that you can act on. Toil is spiky (p. 79). | Record it. Do not argue about it. |
| A few quarters or a year | The number the rule applies to (p. 79). | Enforce the rule. |

**Signal that requires action:** the fraction of time on projects averages significantly
below 50% over the long term. The book requires the team to stop and to find what is wrong
(SRE Ch. 5, p. 79).

---

## 6. The safety valve

**The cap needs a mechanism, or it is only a slogan.** The book names the mechanism. It
redirects excess operational work to the product development team (SRE Ch. 1, p. 25).

**The two named actions:** reassign bugs and tickets to development managers, and integrate
developers into the on-call pager rotations (SRE Ch. 1, p. 25). App. B states the same rule
in one line. Operational overflow goes to the product development team (SRE App. B, p. 575).

**The redirection ends when the operational load returns to 50% or lower** (SRE Ch. 1,
p. 25). Write the exit condition with the valve. A valve with no exit condition becomes a
permanent transfer of work.

**The valve is also a feedback mechanism.** It guides developers to build systems that need
no manual intervention (SRE Ch. 1, p. 25).

**The valve works only when the whole organization understands why it exists.** The goal state
has no overflow events at all, because the product does not generate enough operational load
to need them (SRE Ch. 1, p. 25).

| Signal | Response | Source |
|---|---|---|
| One person reports high toil, and the team average is inside the cap | The manager spreads the toil load more evenly across the team | SRE Ch. 5, p. 78 |
| Project time averages well below 50% over the long term | The team stops and finds the cause | SRE Ch. 5, p. 79 |
| Operational load is above 50% now | Open the valve. Reassign tickets and pager duty to the development team. | SRE Ch. 1, p. 25 |
| The overflow does not stop | Add staff to the team, and assign that team no new operational responsibility | SRE Ch. 1, p. 23 |
| No overflow at all | Goal state. Keep developers in the rotation anyway. | SRE Ch. 1, p. 25. SRE App. B, p. 575 |

**Keep developers in the rotation even with no overflow.** Many services include the product
developers in the on-call rotation and in ticket handling at all times. That gives an
incentive to design systems that minimize toil (SRE App. B, p. 575).

---

## 7. The floor that on-call sets

> `incident-response`, in `incident-response/references/03-on-call.md`, owns the
> rotation itself — the response time, the escalation, the compensation, the handoff, and the
> nine-step procedure for an overloaded team. This section keeps only the two numbers that
> enter the toil arithmetic. They are the floor the rotation size sets and the ceiling on
> paging volume.

**On-call puts a floor under toil.** A typical engineer takes one week as primary and one
week as secondary in each cycle (SRE Ch. 5, p. 77). The arithmetic follows from the rotation
size.

| Rotation size | Weeks on-call per cycle | Lower bound on toil |
|---|---|---|
| 6 people | 2 of 6 | 2 ÷ 6 = 33% (SRE Ch. 5, p. 77) |
| 8 people | 2 of 8 | 2 ÷ 8 = 25% (SRE Ch. 5, p. 77) |

**Read the floor before you argue about the cap.** A 6-person rotation starts at 33% toil
before any other operational work. Team size is part of the toil number, and no amount of
effort inside the team changes it.

**App. B gives the staffing minimum.** At least eight people belong in the on-call team, to
avoid fatigue and to keep turnover low. Two well-separated locations are better, and six
people at each site is then the minimum (SRE App. B, p. 576).

**Alert volume has a ceiling of two events per shift.** The engineer on operations duty
should receive at most two events per 8-hour to 12-hour shift (SRE Ch. 1, p. 25. SRE App. B,
p. 576). That volume leaves time to handle the event, to restore service, and to write the
postmortem (SRE Ch. 1, p. 25).

**Signal above the ceiling:** more than two events per shift as a pattern. The responder
cannot investigate thoroughly and cannot learn from the events (SRE Ch. 1, p. 25). Pager
fatigue does not improve with scale (p. 25). App. B names the three suspects. The fault sits
in the system design, in the monitoring sensitivity, or in the response to postmortem bugs
(SRE App. B, p. 576). Entry L-34 in `slo-defect-catalog.md` covers the scan.

**Signal below the ceiling:** fewer than one event per shift, consistently. That wastes the
time of the on-call engineer (SRE Ch. 1, p. 25). The team can also lose practice and turn a
short outage into a long one. `incident-response` owns the drill formats that fix this.

---

## 8. Measuring toil so that people can argue about the number

**You cannot enforce the cap without a measurement.** The book requires that you first
measure how the time is spent (SRE Ch. 1, p. 23). Entry L-31 in `slo-defect-catalog.md`
covers the missing measurement.

**The named method is a quarterly survey.** Quarterly surveys of Google engineers put the
average time on toil at about 33%, better than the 50% target (SRE Ch. 5, p. 77).

**An average hides the outliers.** Some engineers report 0% toil. Others report 80%
(SRE Ch. 5, p. 77–78). Report the distribution, not only the mean.

**Rank the sources before you automate anything.** The reported order is fixed
(SRE Ch. 5, p. 77):

1. Interrupts. These are non-urgent service-related messages and emails.
2. On-call response to urgent events.
3. Releases and pushes.

Keep the hedge on item 3. Release and push processes already carry a fair amount of
automation, and still leave plenty of room for improvement (p. 77).

**Every method beyond the survey is a local decision.** Choose the method, write it down, and
keep it stable. A number that changes definition each quarter cannot be argued about.

| Field | Example | Why the field exists |
|---|---|---|
| Window | Q3, 13 weeks | The rule averages over quarters (p. 79) |
| Toil percent, per person | 12%, 31%, 44%, 78% | The mean hides the outliers (p. 77–78) |
| Rotation size | 6 people | It sets the floor (p. 77) |
| Paging events per shift | 3.2 | The ceiling is 2 (SRE App. B, p. 576) |
| Top three sources | Interrupts, releases, manual tenant creation | It sets the automation order (p. 77) |
| Valve state | Closed since Q2 | The valve needs an exit condition (SRE Ch. 1, p. 25) |

---

## 9. Toil is not always bad

**Some amount of toil is unavoidable in the role, and in almost any engineering role**
(SRE Ch. 5, p. 79). State this plainly. A plan that promises zero toil is not credible.

**Small amounts of toil do not make people unhappy.** Predictable repetitive tasks can be
calming. They produce a sense of accomplishment and quick wins. They can be low-risk and
low-stress activities (SRE Ch. 5, p. 79). Some people prefer this type of work (p. 79).

**Toil becomes toxic in large quantities** (SRE Ch. 5, p. 79). The book gives the response
for the person who carries too much of it. Be very concerned, and complain loudly (p. 79).

| Condition | Verdict | Source |
|---|---|---|
| A small dose, and the person is content with it | Not a problem | p. 79 |
| The first time or the second time you do the task | Not toil | p. 76 |
| A large quantity, sustained across quarters | Toxic. Act now. | p. 79 |
| The pager fires several times a day, and the team calls it "human judgment" | Toil. Redesign the service. | p. 81, footnote 21 |

---

## 10. The human judgment exemption

**"It needs human judgment" is the exemption to test hardest.** The book warns against the
claim directly. Be careful with it (SRE Ch. 5, p. 81, footnote 21).

**The test:** does the nature of the task intrinsically require human judgment, or could a
better design address it? (p. 81, footnote 21)

**Verdict when better design would remove the judgment.** The service is poorly designed and
carries unnecessary complexity. The team must simplify and rebuild the system. The rebuild
must remove the underlying failure conditions, or handle those conditions automatically.
Until the improved service is deployed, that human judgment work is definitely toil (p. 81,
footnote 21). `self-healing-apis/SKILL.md` owns the automatic handling. Entry L-33 in
`slo-defect-catalog.md` covers the scan.

---

## 11. The harms, named

**Use the book's names in the argument.** Two harms fall on the person. Five fall on the
organization (SRE Ch. 5, p. 79–80). A named harm is harder to dismiss than a complaint.

| Harm | Who carries it | What the book states |
|---|---|---|
| Career stagnation | The person | Career progress slows or stops with too little project time. You cannot make a career out of grunge (p. 79). |
| Low morale | The person | Every person has a limit. Too much toil leads to burnout, boredom, and discontent (p. 79). |
| Creates confusion | The organization | Excess toil undermines the message that this is an engineering organization (p. 80). |
| Slows progress | The organization | Feature velocity falls when the team is too busy with manual work and firefighting (p. 80). |
| Sets precedent | The organization | Developers gain an incentive to move more operational work to the team (p. 80). |
| Promotes attrition | The organization | The best engineers search for a more rewarding job (p. 80). |
| Causes breach of faith | The organization | New hires and transfers joined with a promise of project work, and feel cheated (p. 80). |

Entry L-32 in `slo-defect-catalog.md` carries this list as the consequence of an uncapped
operational load.

---

## 12. War stories

### The service that pages several times a day (SRE Ch. 5, p. 81, footnote 21)

Someone built a service that alerts its engineers several times a day. Each alert needed a
complex response with plenty of human judgment. The owners used that fact to argue that the
work was not toil. The book rejects the argument. It rules the service poorly designed, with
unnecessary complexity. The remedy is a rebuild that removes the underlying failure
conditions, or that handles them automatically. Until that rebuild is deployed, every alert
response is definitely toil. An exemption that depends on a design defect is not an exemption.

### The spread from 0% to 80% (SRE Ch. 5, p. 77–78)

Quarterly surveys put the average time on toil at about 33%, well inside the 50% target. The
average hid the range. Some engineers reported 0% toil, because they worked on pure
development projects with no on-call duty. Others reported 80%. The book reads an 80% report
as a management signal, not as a personal failure. The manager must spread the toil load more
evenly, and must help that engineer find a satisfying engineering project. A healthy team
average does not prove a healthy team.

### The six-server clean shutdown (Release It! Sec. 14.4, p. 248)

An order management system exposed its administration through a Java GUI. The clean shutdown
sequence needed clicking, and several minutes of waiting, on each of six servers. The change
window was one hour. Half of that hour would go to waiting on the GUI. Nygard asks how often
the team actually observed the clean shutdown sequence, and predicts many `kill -9` commands
instead. His rule: "The best interface for long-term operation is the command line" (p. 248).
Every administration duty must be scriptable (Release It! Ch. 15, p. 250). An interface that
a person cannot script converts a designed safety step into a skipped step.

### Black Monday, read as toil (Release It! Sec. 4.11, pp. 106–108)

`self-healing-apis`, in `self-healing-apis/references/02-stability-antipatterns.md`,
owns this story and the Unbounded Result Sets antipattern behind it. One line of it belongs
here. The team stabilized the system with "an extraordinary amount of hand-holding and manual
work" because one query had no `LIMIT` clause (p. 107). That manual work is the exact shape of
toil, and a bounded query would have removed the need for all of it. A design defect converts
into an operational load, and the toil number is where it becomes visible.

---

## 13. The toil block in the SLO Record

Write this block when the change adds operational work. Omit it when the change adds none.

```
### Toil accounting
- New manual task: [restart the importer after a vendor 5xx response]
- Attributes matched: [manual, repetitive, automatable, tactical, no enduring value]
- Growth: [O(n) with tenant count]
- Estimated hours per week now: [3]
- Estimated hours per week at 10x tenants: [30]
- Automation item filed: [BUG-1421, owner: named person]
```

**Estimate the load at 10 times the current scale.** Attribute 6 converts a small task into a
staffing problem (SRE Ch. 5, p. 76). One line of arithmetic exposes it before the merge.

**File the automation item with a named owner.** A toil entry with no owner is a measurement,
not a plan. The book's closing instruction is to remove a small amount of toil each week
(SRE Ch. 5, p. 80–81).

---

## 14. Defect codes, and where this file stops

`slo-defect-catalog.md` is the source of truth. This table gives the trigger only.

| Code | Name | Signal that triggers it |
|---|---|---|
| L-31 | Toil never measured | No survey, no ticket category, and no time record separates operational work from engineering work |
| L-32 | Operational work above the cap with no valve | Measured toil above 50% across several quarters, and no written rule redirects the overflow |
| L-33 | "It needs human judgment" used as a toil exemption | A service pages several times a day, and the owners argue that each page needs judgment |
| L-34 | Alert volume above the shift ceiling | More than two paging events per 8-hour to 12-hour shift, as a pattern |

**Where this file stops.** It counts hours and names buckets. It does not build the
automation, and it does not run the incident.

| Question | Owner |
|---|---|
| How do I handle the failure condition automatically? | `self-healing-apis/SKILL.md` |
| How do I remove the human from the release? | `reverse-branching/SKILL.md` |
| How do I run the page when it fires? | `incident-response/SKILL.md` |
| How do I size the rotation, and how do I unload a drowning team? | `incident-response/references/03-on-call.md` |
| How do I stop the alert that should not have paged? | `observability/references/02-symptom-based-alerting.md` |
| How many minutes does my objective allow? | `03-availability-math.md` |
| What stops the next release? | `01-risk-and-the-error-budget.md` |
| Which indicator do I measure? | `02-slis-slos-and-slas.md` |

**When this file stays silent.** A change that adds no manual step, no pager path, and no
per-customer manual work needs no toil accounting. Write one line and continue:
`Toil: not applicable — no new operational work.` That line is a valid result.

**Do not invent the local number.** The 50% cap, the 33% and 25% floors, and the ceiling of
two events per shift come from the book. The current toil percentage of your team does not.
Measure it, or write that it is unmeasured and cite L-31.
