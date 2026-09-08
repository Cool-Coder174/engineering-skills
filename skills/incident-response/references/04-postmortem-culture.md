# Blameless Postmortems

**Sources:** *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy). Ch. 15
"Postmortem Culture: Learning from Failure", p. 208–215. App. D "Example Postmortem",
p. 579–584. Supporting pages: Ch. 13, p. 197–198. Ch. 16, p. 216. Ch. 28, p. 486.
Ch. 30, p. 506–507. Ch. 33, p. 556–558. App. B, p. 574.

This file gives six things. They are the trigger criteria, the blameless rule, the template,
the action item rule, the review, and the adoption route. The adoption route serves a team that
does not yet write a record.

---

## 1. What a postmortem is

**A postmortem is a written record of an incident** (SRE Ch. 15, p. 208). The record holds the
impact, the actions that mitigated or resolved the incident, the root causes, and the actions
that prevent a repeat. **Without a formal learning process, incidents can recur without end.**
Unchecked incidents grow in complexity. They can cascade, overwhelm the system and its operators,
and reach the user (SRE Ch. 15, p. 208).

The book names three goals for the record (SRE Ch. 15, p. 208–209).

1. The incident is documented.
2. Every contributing root cause is understood.
3. Preventive actions reduce the likelihood or the impact of a repeat. The book gives this goal
   the most weight. A record with no effective preventive action fails its main purpose.

**The record costs time and effort.** The team therefore chooses deliberately when to write one
(SRE Ch. 15, p. 209). The live incident state document feeds the record. Read
`01-incident-command.md`.

---

## 2. When to write one

**Agree the trigger criteria before an incident happens, so everyone knows when a postmortem is
necessary** (SRE Ch. 15, p. 209).

| Trigger | Threshold | Signal that it fired |
|---|---|---|
| User-visible downtime or degradation | Above the agreed threshold | The SLO burn rate, or a customer report |
| Data loss | Any amount. No threshold. | A reconciliation gap, or a missing record |
| On-call engineer intervention | Any. A release rollback counts. Traffic rerouting counts. | A change that a person made by hand during the event |
| Resolution time | Above the agreed threshold | The clock in the incident state document |
| Monitoring failure | Any. A person found the incident, and the monitor did not. | No alert fired before the report |
| A stakeholder asks for one | Always | The request itself |

**Source:** SRE Ch. 15, p. 209.

**The threshold values are a local decision.** The book states that a threshold exists. It gives
no number. Set the numbers against the SLO, and write them down.

**Any stakeholder may ask for a postmortem for an event** (SRE Ch. 15, p. 209). This trigger has
no threshold. **A postmortem is not a punishment.** The book states it plainly: "Writing a
postmortem is not punishment" (SRE Ch. 15, p. 209). It is a learning opportunity for the company.
Catalog code `N-31` fires when no trigger list exists. Read `incident-failure-catalog.md`.

---

## 3. The blameless rule

**Blameless means that the record names the system fault, not the person.** A blameless record
identifies the contributing causes without indicting any individual or any team
(SRE Ch. 15, p. 209).

The record makes two assumptions about every person in the event (SRE Ch. 15, p. 209).

1. Everyone involved had good intentions.
2. Everyone did the right thing with the information that they had.

**A blame culture makes people hide problems.** When shaming prevails, people do not report
issues, because they fear punishment. Issues stay hidden, and the organization carries more risk
(SRE Ch. 15, p. 209–210).

**Blameless culture came from healthcare and avionics**, where a mistake can kill
(SRE Ch. 15, p. 209). Those industries treat each mistake as a chance to strengthen the system.
You cannot fix a person. You can fix the systems that support the person.

**Blameless does not mean toothless.** A blameless record still states where the service must
improve and how (SRE Ch. 15, p. 210).

### 3.1 Rewrite the blaming sentence

The book prints one pair of examples about the same backend system (SRE Ch. 15, p. 210). The first
version vents frustration about weekly breakage and about the pages. The second version proposes
the same rewrite as an action item, and names the long maintenance manual as the real difficulty.
Both ask for the same work. Only the second version is usable.

| Text in the draft | Verdict | Rewrite |
|---|---|---|
| "X should have checked the config" | Blame | State that the config had no validation step |
| "If I get paged again I will rewrite it myself" | Blame | Propose the rewrite as an action item with a named owner |
| "The system misled the operator at 15:32" | Blameless | Keep it. Add the action item that removes the trap. |

### 3.2 The phrasing that gets the facts

**Ask the on-call engineer to write what they thought at each point in time.** The team then
finds where the system misled the engineer, and where the cognitive demands were too high
(SRE Ch. 30, p. 507).

**Reject the Bad Apple Theory.** That theory says the system works, and that removing the bad
people keeps it working. The book states the verdict: "The Bad Apple Theory is demonstrably
false" (SRE Ch. 30, p. 507). Evidence from airline safety and other fields contradicts it.

**Fix the environment, not the people** (SRE App. B, p. 574). The book names three moves.

1. Improve the system design, so it avoids whole classes of problem.
2. Make the needed information easy to reach.
3. Validate operational decisions automatically, so a dangerous state is hard to reach.

Catalog code `N-32` fires when the record names a person as the cause. It is a 🔴 code.

---

## 4. Collaborate and share knowledge

The workflow uses collaboration at every stage (SRE Ch. 15, p. 210). The tool matters less than
three features.

| Feature | Why the book requires it | Signal that you lack it |
|---|---|---|
| Real-time collaboration | It collects data and ideas fast, during the early creation of the record | Only one author can type, so the draft lags the event |
| Open commenting and annotation | It makes crowdsourced solutions easy and improves coverage | Feedback arrives as private mail, and the draft never records it |
| Email notification | It reaches collaborators inside the document, and adds other people for input | Reviewers learn about the draft by chance |

**Source:** SRE Ch. 15, p. 210–211.

**Share the record with the widest audience that benefits from it** (SRE Ch. 15, p. 211). The
team shares the first draft internally. After review it goes to the larger engineering group.
**Never include any piece of information that identifies an end user** (SRE Ch. 15, p. 211). That
rule has no exception, and it applies to internal documents.

---

## 5. The template

The book prints a complete worked record for incident #465 (SRE App. D, p. 579–584), with these
fields.

| Field | What it holds | Source |
|---|---|---|
| Date, Authors, Status | The metadata. Status is a state such as "Complete, action items in progress". | App. D, p. 579 |
| Summary | What broke, and for how long. Two sentences. | App. D, p. 579 |
| Impact | The effect on users and on revenue. The book's record states lost queries and no revenue effect. | App. D, p. 579, footnote 163 |
| Root Causes | The circumstances that produced the incident. Plural. Use a technique such as the 5 Whys. | App. D, p. 579, footnote 164 |
| Trigger | The event that started it. The book's record names a latent bug and a traffic surge. | App. D, p. 579 |
| Resolution | What restored the service, and what capacity the team holds afterwards | App. D, p. 579 |
| Detection | What alerted, and how. State it when a person found the fault. | App. D, p. 579 |
| Action Items | The table in Section 6 | App. D, p. 579–581 |
| Lessons Learned | What went well, what went wrong, and where we got lucky | App. D, p. 581 |
| Timeline | A screenplay of the incident, in UTC, with an actor on each entry | App. D, p. 581–583, footnote 167 |
| Supporting information | Links to dashboards, logs, graphs, and chat transcripts | App. D, p. 583, footnote 168 |

**Seed the timeline from the incident state document, then add the other entries**
(SRE App. D, p. 584, footnote 167). Do not rebuild it from memory. **The book asks for every
contributing cause, not one cause** (SRE Ch. 15, p. 208). A single-cause record fails the review
criterion on depth.

**Record the error budget effect.** The book's worked record states that the incident exceeded
the availability error budget by several orders of magnitude. The book puts that statement
under "What went wrong", and it files the production freeze as an action item of type `other`
(SRE App. D, p. 581). This skill also requires the number in Impact. That requirement is a
local addition. `slo-engineering` owns the budget itself.

---

## 6. Action items

Each action item carries one of four types (SRE App. D, p. 579–581).

| Type | Meaning | Example from the book's record |
|---|---|---|
| `mitigate` | Reduces the impact of the next occurrence | Update the playbook with instructions for a cascading failure |
| `prevent` | Removes the cause | Plug the file descriptor leak in the ranking subsystem |
| `process` | Changes how people work | Schedule a cascading failure test during the next drill |
| `other` | Everything else | Freeze production until the error budget recovers, or ask for an exception |

**An action item with no owner and no date is not an action item.** Nobody does unowned work, so
the next incident repeats the last one. The book's own table carries a type, a named owner, a
bug number, and a state such as `TODO` or `DONE` (SRE App. D, p. 579–581). The book prints no
due date column. The due date is a local addition, and this skill requires it.

**Name a person, never a team.** Every owner in the book's record is one named engineer.

**Reject an action item that only reverts the change that triggered the outage.** The book gives
the pair. Replace "Change the streaming timeout back to 60 seconds" with an item that finds why
the first megabyte sometimes takes 60 seconds (SRE Ch. 30, p. 506).

**Rescope an action item that is too extreme or too costly.** The book calls these "knee-jerk"
items. They over-optimize for one issue. The book's example is specific monitoring where a unit
test catches the problem much earlier in development (SRE App. D, p. 583, footnote 165).

**Hold yourself and other people accountable for the actions in the record**
(SRE Ch. 13, p. 198). That accountability stops a near-identical outage with near-identical
triggers. An item that becomes real work leaves this skill. Hand it to `planner/SKILL.md`.
Catalog codes `N-33` and `N-34` cover these two defects. Both are 🔴 codes.

---

## 7. Lessons learned and near misses

The Lessons Learned section has three parts (SRE App. D, p. 581).

- **What went well.** The book's record names fast monitoring detection and fast corpus
  distribution.
- **What went wrong.** The book's record states that the team was out of practice at responding
  to cascading failure, and that it exceeded the error budget.
- **Where we got lucky.** This section is for near misses (SRE App. D, p. 584, footnote 166).

**Treat every entry under "Where we got lucky" as a preemptive postmortem.** The manufacturing
and chemical industries scrutinize a near miss, because the harm did not arrive this time
(SRE Ch. 33, p. 557). A latent error plus an enabling condition produces the failure. The book
states the conclusion: "Near misses are effectively disasters waiting to happen"
(SRE Ch. 33, p. 558). The United Kingdom runs CHIRP, a central point where aviation and maritime
personnel report a near miss in confidence. Reports and analyses then appear in periodic
newsletters (SRE Ch. 33, p. 558).

**Signal:** an entry under "Where we got lucky" that has no matching action item. Luck is not a
control. Write the item, or state why the team accepts the risk.

**Write down what went wrong even when it embarrasses the team.** A record that lists only
successes teaches nothing.

---

## 8. The review

**No postmortem is left unreviewed.** The book states the reason: "An unreviewed postmortem
might as well never have existed" (SRE Ch. 15, p. 211). **Signal to schedule the review:** a
completed draft with open comments. Hold regular review sessions. In them the team closes the
open discussions, captures the ideas, and sets the final state. A group of senior engineers then
assesses the draft against these criteria (SRE Ch. 15, p. 211).

| Review criterion | Verdict | What a failed verdict means |
|---|---|---|
| Was key incident data collected for posterity? | Pass / Fail | The evidence is gone. Fix the capture path first. |
| Are the impact assessments complete? | Pass / Fail | Users, duration, and money are not all stated. |
| Was the root cause sufficiently deep? | Pass / Fail | The record stops at the trigger and names no contributing cause. |
| Is the action plan appropriate, and are the bug priorities correct? | Pass / Fail | Items are rollback-only, unowned, or knee-jerk. |
| Did we share the outcome with relevant stakeholders? | Pass / Fail | The lesson stays inside one team. |

**After the review, add the record to a team or organization repository of past incidents**
(SRE Ch. 15, p. 212). Transparent sharing makes the record easy to find. The book names Etsy's
released tool **Morgue** as a way to start a repository (SRE Ch. 15, p. 215, footnote 81). Search
the archive for actions that prevent the outage both tactically and strategically
(SRE Ch. 13, p. 197). Catalog code `N-35` fires on an unreviewed draft.

---

## 9. Introducing a postmortem culture

**Signal that a team needs this section:** the archive is empty, members call the process
punitive, or a member asks "Why me?" (SRE Ch. 30, p. 507). The effort needs continuous
cultivation (SRE Ch. 15, p. 212). One announcement does not create the practice.

### 9.1 The three adoption strategies

The book gives these against the objection that a postmortem costs too much
(SRE Ch. 15, p. 213).

| Step | Action | Signal to take the next step |
|---|---|---|
| 1 | Ease postmortems into the workflow. Run a trial of several complete records. | The trial produces useful action items |
| 2 | Reward and celebrate effective records, in public and in performance management | People write records without a request |
| 3 | Get senior leadership acknowledgment and participation | The practice survives a busy quarter |

**Use the trial period to identify the trigger criteria** (SRE Ch. 15, p. 213). The trial
produces the table in Section 2. Do not write that table first and then argue about it.
**Management can encourage the culture. Engineers own it.** The book states that a blameless
record is ideally the product of engineer self-motivation (SRE Ch. 15, p. 212).

### 9.2 The dissemination activities

| Activity | Cadence | What it does |
|---|---|---|
| Postmortem of the month | Monthly newsletter | Shares one interesting and well-written record across the organization |
| Postmortem discussion group | Continuous | Discusses internal and external records, and best practices |
| Postmortem reading club | Regular, per team | Opens a dialogue on one record, often months or years old |
| Wheel of Misfortune | Weekly, 30 to 60 minutes | Reenacts a past record with engineers in the roles it names |

**Source:** SRE Ch. 15, p. 212–213. Cadence for the Wheel of Misfortune from SRE Ch. 28, p. 486.
**The original incident commander attends the Wheel of Misfortune.** That presence makes the
exercise real (SRE Ch. 15, p. 213). Read `03-on-call.md` for the drill formats.

### 9.3 The route for one engineer inside an unhealthy team

**Do not review the archive and leave comments.** That exercise does not help the team, and it
makes the team defensive. **Take ownership of the next record instead.** An outage arrives. Write
a good blameless record with the on-call engineer. The record demonstrates that permanent bug
fixes reduce the cost of outages on the team's own time (SRE Ch. 30, p. 506–507).

### 9.4 Ask whether the practice works

Survey the teams regularly (SRE Ch. 15, p. 213–214). The book asks four questions.

1. Is the culture supporting your work?
2. Does writing a postmortem entail too much toil?
3. What best practices does your team recommend for other teams?
4. What tools would you like developed?

**Do not stigmatize frequent postmortems** (SRE Ch. 15, p. 210). Volume is a service property.

---

## 10. War stories from the books

**The four-minute outage that earned applause (SRE Ch. 15, p. 213).** At a 2014 all-hands meeting
titled "The Art of the Postmortem", one SRE described a release that he had pushed. Testing had
been thorough. An unexpected interaction still stopped a critical service for four minutes. It
lasted only four minutes because the engineer reverted the change at once, which averted a much
longer and larger outage. He received two peer bonuses immediately, and applause from thousands
of colleagues. The organization rewarded the fast revert, and it did not punish the push.

**Lifeguarding and the paperwork rule (SRE Ch. 33, p. 558).** Mike Doherty told the authors that
a lifeguard whose feet enter the water generates paperwork. Any incident at a pool or on a beach
requires a detailed write-up. For a serious incident the team examines the event end to end, and
states what went right and what went wrong. Operational changes follow the findings, and training
often follows the changes. After a traumatic incident a counselor comes on site, because the
lifeguards may have performed well and still feel that they failed. The field treats incident
analysis as blameless, because many factors contribute to any incident.

**The worked record for incident #465 (SRE App. D, p. 579–584).** News of a discovered sonnet
drove an 88-fold traffic increase in two minutes. The corpus did not hold the sonnet. Searches
for its unique term then reached a latent file descriptor leak on the no-results path, and the
search backends failed in cascade. The record runs 66 minutes, states about 1.21 billion lost queries
and no revenue effect, and lists nine action items across all four types. Its timeline records a
whiteboard calculation that was wrong by an order of magnitude, and the revert that followed. The
record names the error, and it names no culprit.

---

## 11. The coverage gap that a postmortem cannot close

**Postmortems alone leave a gap.** Teams write them only for incidents with a large impact
(SRE Ch. 16, p. 216). Issues with small individual impact that are frequent and widespread fall
outside that scope. **Signal:** the archive holds only large incidents, and the on-call load
still rises. The remedy is an outage tracker that receives every alert, and that supports
grouping, tagging, and analysis (SRE Ch. 16, p. 217–220). Read `05-tracking-outages.md`. Catalog
code `N-36` covers this gap.

---

## 12. Failure codes in this area

Run these against a draft record before it reaches the repository. Full entries live in
`incident-failure-catalog.md`.

| Code | Severity | Signature |
|---|---|---|
| `N-31` | 🟡 | No postmortem triggers agreed in advance |
| `N-32` | 🔴 | Blame in the postmortem |
| `N-33` | 🔴 | Rollback-only action items |
| `N-34` | 🔴 | An action item with no owner and no date |
| `N-35` | 🟡 | Unreviewed postmortem |
| `N-36` | 🔵 | Postmortem coverage gap |

---

## 13. Proportionality and limits

**Not every event needs this file.** A page that the on-call engineer resolved inside the
runbook, with no user impact and no manual change, produces a tracker entry and no record. Read
Section 2 before you ask for a document. **This file stays silent on three topics.** The books
contain no material on legal notification, on regulator reporting, or on customer credits.

**Severity labels such as SEV1 and P1 appear in neither book.** Tag them **Modern** when a team
uses them, and map them to the response classes in `01-incident-command.md`. **Automated record
creation is partly Modern.** The book describes a working group that assembles templates, creates
records from incident tooling data, and extracts data for trend analysis (SRE Ch. 15, p. 214). It
names machine learning only as future work. Any tool claim beyond that carries the tag
**Modern**.

| Question | File |
|---|---|
| Who commands, and what does the live document hold? | `01-incident-command.md` |
| What do the three emergency classes teach? | `02-emergency-response.md` |
| How do we size a rotation, and how do we drill? | `03-on-call.md` |
| How do we see trends across many small events? | `05-tracking-outages.md` |
| How does an overloaded team recover? | `06-interrupts-and-overload.md` |
| How do we revert safely, and how do we test the rollback? | `reverse-branching/SKILL.md` |
| An action item became real work. Who plans it? | `planner/SKILL.md` |
| Which method finds the root cause that this record states? | `production-troubleshooting/SKILL.md` |
| Where does the error budget in Impact come from? | `slo-engineering/SKILL.md` |
