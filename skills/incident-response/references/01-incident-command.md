# The Incident Command System

**Sources:** *Site Reliability Engineering* — Ch. 14, "Managing Incidents", p. 200–207.
App. C, "Example Incident State Document", p. 577–578. Supporting pages: Ch. 11, p. 164–165.
Ch. 12, p. 173. Ch. 13, p. 191–196.

This file answers one question. A fault needs more than one person, so who leads, and what
does the team write down?

Google built its incident management system on the Incident Command System. The book names
that system for its clarity and for its scale (SRE Ch. 14, p. 202). The chapter states the
measured result. The method reduced the mean time to recovery, and it gave staff a less
stressful way to work (SRE Ch. 14, p. 206).

**The method does not repair the machine. It repairs the human response.** The technical
facts in the two stories below are identical. Only the command structure differs. One story
runs for a day and gets worse. The other closes overnight (SRE Ch. 14, p. 200–205).

---

## 1. The anatomy of an unmanaged incident

### War story — The Firm (SRE Ch. 14, p. 200–202)

Mary is on-call. At 2 p.m. black-box monitoring reports that her service serves no traffic in
one datacenter. A second datacenter fails. Then a third of five fails. The two survivors take
more traffic than they can hold, and they overload. Mary reads thousands of log lines, sees
an error in a recently updated module, and reverts the servers to the previous release. The
rollback does not help. She wakes Josephine at 3:30 a.m. in her time zone. Sabrina and Robin
start work from their own terminals. An executive telephones Mary's boss. Vice presidents ask
for an estimate and offer advice that Mary cannot refute and cannot use. The last two
datacenters fail. Josephine, unknown to Mary, calls Malcolm. Malcolm believes a CPU-affinity
change will help, so he deploys it straight to production. The servers restart, accept the
change, and die.

**Every person in that story did the job as they saw it (SRE Ch. 14, p. 201).** No person was
careless. The response still spiraled. The book names three hazards that produced the spiral.

| Hazard | Signature in the record | What it costs |
|---|---|---|
| Sharp focus on the technical problem | One name appears on the alerts and on the production commands | Nobody thinks about the wider mitigation |
| Poor communication | No periodic update. No shared log of actions | Stakeholders interrupt. Willing engineers stay unused |
| Freelancing | A production change by an actor outside the operations role | The change lands on a damaged system |

### Sharp focus on the technical problem

**A responder who holds the technical work cannot also hold the strategy.** The book states
the mechanism plainly. Mary was busy with operational changes, so she was not in a position
to think about the bigger picture (SRE Ch. 14, p. 201). Technical load displaces incident
strategy. The remedy is a second person, not a stronger first person.

**Signal.** The second alert fires and the same person still runs the queries, the reverts,
and the updates. Transfer command now. See Section 3.

### Poor communication

**The same overload that stops the analysis also stops the reporting.** Nobody knew what
their coworkers were doing (SRE Ch. 14, p. 201). Three costs follow. Business leaders get
angry. Customers get frustrated. Engineers who could help are not used effectively.

**Signal.** A stakeholder asks a responder for an estimate. That question means the
communications role is vacant. See Section 4.

### Freelancing

**A production change by an uncoordinated actor is the most damaging act in the book's
story.** Malcolm meant well. He did not coordinate with anyone, and not with Mary, who was in
charge of the troubleshooting. The book states the result in one sentence. "His changes made
a bad situation far worse" (SRE Ch. 14, p. 202).

**Signal.** The audit log shows a deploy, a restart, a flag change, or a config push during a
declared incident. The actor is not the operations lead. That is catalog entry `N-02` in
[`incident-failure-catalog.md`](incident-failure-catalog.md).

---

## 2. The four elements of the process

The book lists four features of a well-designed process (SRE Ch. 14, p. 202–204). Build all
four. Each one closes a different hazard from Section 1.

| Element | What it fixes | Section |
|---|---|---|
| Recursive separation of responsibilities | Sharp focus, and freelancing | 3 |
| A recognized command post | Poor communication | 4 |
| Live incident state document | Lost state, and repeated work | 5 |
| Clear, live handoff | Command that lapses at the end of a shift | 6 |

**Incident management skills exist to channel the energy of enthusiastic individuals**
(SRE Ch. 14, p. 202). Read the aim carefully. The process does not reduce the effort. It
directs the effort.

---

## 3. Recursive separation of responsibilities

**Every responder knows their role, and no responder takes another role.** The book
calls the result counterintuitive. A clear separation gives a person more autonomy, because
that person need not second-guess a colleague (SRE Ch. 14, p. 202).

### The four roles

| Role | The role owns | The role must not do |
|---|---|---|
| Incident commander | The high-level state, the task force structure, the role assignments, the incident document | Change the system |
| Operations lead | Every change to the system, with the delegates of that lead | Answer stakeholders |
| Communications lead | Periodic updates to responders and to stakeholders. The summary. The external message | Change the system |
| Planning lead | Bugs, staff, handoffs, and the record of how the system diverged from the norm | Change the system |

**The commander holds every role that the commander did not delegate** (SRE Ch. 14, p. 202).
A small incident can therefore run with one person in four roles. The roles still exist. The
commander can also remove a roadblock that stops the operations lead from working.

**The commander may repair the process, not the service.** The first duty of the commander is
the living document, not the fix (SRE Ch. 14, p. 203). A commander who types production
commands has abandoned the high-level state.

### Who may change production

| Actor | May change production | Rule |
|---|---|---|
| The operations lead | Yes | The operations team is the only group that changes the system (SRE Ch. 14, p. 202–203) |
| A delegate of the operations lead | Yes | The delegate reports every action to that lead (SRE Ch. 14, p. 205) |
| The incident commander | No | Unless the commander has not delegated the operations role |
| Any other responder | No | Propose the change to the operations lead |
| A person outside the incident | No | Freelancing made the book's example far worse |
| An automated agent | No | The operations lead starts it. The record names it. **Modern** |

**Give full autonomy inside the assigned role. Give none outside it** (SRE Ch. 14, p. 206).
The book calls this practice Trust. The two halves are one rule. Autonomy without a boundary
is freelancing.

### Load relief

**A responder with too much load asks the planning lead for staff.** That responder then
delegates the work. The delegation can create a subincident (SRE Ch. 14, p. 202). A role
leader can also delegate system components to colleagues. Those colleagues report high-level
information back to the leaders.

**Signal to ask for staff.** You feel panicky or overwhelmed. The book names this practice
Introspect, and it directs you to request more support (SRE Ch. 14, p. 206). Ch. 11 gives the
mechanism. Stress hormones impair cognition, and the impaired responder abuses heuristics
(SRE Ch. 11, p. 164). Escalation is a principled reaction to an outage with large unknown
dimensions (SRE Ch. 11, p. 165).

---

## 4. A recognized command post

**Interested parties must know where they can interact with the commander**
(SRE Ch. 14, p. 203). Name one place in the record. Name it before the responders need it.

| Option | The book's words | Use it when |
|---|---|---|
| A central "War Room" | Appropriate in many situations | The responders share one site |
| Desks, plus email and a chat channel | Some teams may prefer this | The responders are distributed |

Google found IRC a large benefit in incident response (SRE Ch. 14, p. 203). The book names
four reasons. The medium is very reliable. It serves as a log of communications about the
event. Bots can log incident traffic for the postmortem, and other bots can log alerts into
the channel. Distributed teams can coordinate over it. A modern chat platform must supply the
same four properties, and it must not run on the failing service. **Modern**

**Signal that you have no command post.** The record names no single channel or room, and the
discussion runs across direct messages and calls. That is catalog entry `N-04`.

---

## 5. The live incident state document

**This document is the most important responsibility of the commander**
(SRE Ch. 14, p. 203). Several people should be able to edit it at the same time.

### The fields

App. C, p. 577–578, gives the artifact. The fields are these.

| Field | Content | Rule |
|---|---|---|
| Summary | One sentence. The failure mode and the suspected cause | The communications lead keeps it current |
| Status | Active or resolved, plus the incident number | — |
| Command post | The channel or the room | It must not run on the failing system |
| Command hierarchy | Commander, operations lead, planning lead, communications lead, next commander | Update at each transfer of command |
| Detailed status | What is broken, what is degraded, what the team does now | Stamp it with a time and an author |
| Exit criteria | A checklist of measurable conditions | Each line is TODO or DONE |
| Todo list and bugs | Each item with a bug number | Each line is TODO or DONE |
| Timeline | Most recent first. Times in UTC. Each entry names an actor | Seed the postmortem timeline from it |

**Update the detailed status at least every four hours, and at each handoff of the
communications role** (SRE App. C, p. 577). The book states no other cadence. A shorter
cadence is a local decision.

**Write measurable exit criteria.** The book's example holds two conditions. The team adds
the new sonnet to the search corpus. The service then holds two SLOs for 30 or more minutes.
The availability SLO is 99.99 percent. The latency SLO is 100 ms at the 99th percentile
(SRE App. C, p. 577). Copy the shape, not the numbers. Your SLO is a local decision, and
`slo-engineering` owns it.

### Rules that govern the document

1. Use a template. A template makes the document easier to produce (SRE Ch. 14, p. 204).
2. Keep the most important information at the top. That order makes it usable
   (SRE Ch. 14, p. 204).
3. Accept a messy document. It must be functional, and it need not be tidy
   (SRE Ch. 14, p. 204).
4. Post every attempt and every result. A failed remedy is data, not an ending
   (SRE Ch. 14, p. 204–205).
5. Retain the document for the postmortem and for meta analysis (SRE Ch. 14, p. 204).

### Host the tooling outside the fault

**Do not run your incident tooling on the service that you repair.** Google Docs SRE keep
their incident document on Google Sites for this reason. The book states the risk in one
sentence. Depending on the software that you try to fix is unlikely to end well
(SRE Ch. 14, p. 203).

### War story — the crash-loop that hid the tools (SRE Ch. 13, p. 191–194)

The team pushed a configuration change to the abuse-protection infrastructure globally on a
Friday. It triggered a crash-loop bug, and the fleet began to crash-loop almost at once.
Internal applications failed with the external ones, because Google's internal infrastructure
depends on Google's own services. Some on-call engineers believed that the corporate network
had failed, and they moved to panic rooms with backup production access. The push engineer
pushed a second config change to revert the first within five minutes. The book names what
saved the response. Out-of-band communications kept everyone connected. Command-line tools
and alternative access methods allowed updates and reverts when the normal interfaces were
unreachable (SRE Ch. 13, p. 193). The book also names the exposure. Much of the software
stack used for troubleshooting and communication sat behind the crash-looping jobs
(SRE Ch. 13, p. 194). Keep those backup paths, and use them regularly.

---

## 6. Clear, live handoff

**Transfer command explicitly at the end of the working day** (SRE Ch. 14, p. 204). A shift
that ends with no transfer leaves nobody in charge.

The procedure has four steps.

1. Brief the incoming commander in full. Use a telephone call or a video call for a remote
   colleague.
2. State the transfer in words. The book's script is "You're now the incident commander,
   okay?"
3. Stay on the call until you receive firm acknowledgment.
4. Broadcast the transfer to everyone who works the incident, so the leader is clear at all
   times.

**Signal of a silent handoff.** A shift changes, and no message names the new commander. That
is catalog entry `N-06`.

### War story — the same incident, managed (SRE Ch. 14, p. 204–205)

The second alert fires. Mary asks Sabrina to take command. Sabrina takes a rundown, and she
writes the details into an email to a prearranged mailing list. She cannot yet scope the
impact, so she asks Mary. Mary answers that users are not yet affected, and Sabrina records
the answer in the live incident document. On the third alert Sabrina updates the email
thread, which keeps the vice presidents current without minutiae. She asks an external
communications representative to draft user messaging. She checks with Mary before she adds
the developer on-call. She reminds the new joiners of two duties. They prioritize the tasks
that Mary delegates, and they inform Mary of any other action that they take. Mary tries the
old binary release, and it does not help. Robin reports that failure to the channel, and
Sabrina pastes it into the document. At 5 p.m. Sabrina finds replacement staff. At 5:45 p.m.
a brief phone conference makes the situation common knowledge. At 6 p.m. the team transfers
responsibility to the sister office. Mary returns the next morning to a mitigated problem, a
closed incident, and a postmortem already in progress.

---

## 7. When to declare an incident

**Set the conditions before an event happens** (SRE Ch. 14, p. 205). An argument about the
threshold during an outage costs the outage.

| Question | Answer | Action |
|---|---|---|
| Do you need a second team to fix the problem? | Yes | Declare an incident |
| Is the outage visible to customers? | Yes | Declare an incident |
| Is the issue unsolved after one hour of concentrated analysis? | Yes | Declare an incident |
| All three answers are no | — | Work the page. Ask the three questions again later. |

The book gives these three as broad guidelines from one team, and any one of them is
sufficient (SRE Ch. 14, p. 205–206). The re-check interval is a local decision.

**Declare early, then close early.** The book prefers a declared incident with a simple fix
over an incident framework that starts hours into a growing problem (SRE Ch. 14, p. 205). The
cost of an early declaration is one email and one document. The cost of a late declaration is
the whole spiral in Section 1.

**A declaration is not a severity.** The book names no severity levels. `SEV1` and `P1` appear
in neither book. Map any local severity scheme onto this gate, and tag the scheme
**Modern**.

---

## 8. Best practices for incident management

The chapter closes with seven named practices (SRE Ch. 14, p. 206–207).

| Practice | The rule | The signal to apply it |
|---|---|---|
| Prioritize | Stop the bleeding, restore service, then preserve the evidence for root-causing | The response starts |
| Prepare | Develop and document the procedures in advance, with the participants | No incident is running |
| Trust | Give full autonomy inside the assigned role | A responder asks for permission inside their own role |
| Introspect | Watch your emotional state. Request support when you feel overwhelmed | You feel panicky |
| Consider alternatives | Re-evaluate whether to continue the current approach | A period passes with no change in the symptom |
| Practice | Use the process routinely, so it becomes second nature | No incident is running |
| Change it around | Rotate the roles between incidents | You held the same role last time |

**Prioritize is an order, not a list.** Step one comes before step four. Ch. 12 states the
reason for the order. You do not help your users while the system dies and you search for the
root cause (SRE Ch. 12, p. 173). The mitigation moves themselves live in
[`02-emergency-response.md`](02-emergency-response.md).

**Change it around trains every member.** Every team member should gain familiarity with
every role (SRE Ch. 14, p. 207). A team with one competent commander has a single point of failure.

---

## 9. Practice, because the skill decays

**The book states the decay directly.** "Incident management proficiency atrophies quickly
when it's not in constant use" (SRE Ch. 14, p. 206). A framework that a team uses twice a
year is not a capability.

Three vehicles keep the skill current (SRE Ch. 14, p. 206).

1. Apply the framework to routine change management that spans time zones or teams.
2. Include incident management in disaster-recovery testing.
3. Role-play the response to an on-call issue that a colleague already solved.

The drill formats and their cadence live in [`03-on-call.md`](03-on-call.md).

---

## 10. The scan that this file supports

Run these catalog entries against a runbook, a chat log, an audit log, or an incident record.
Each entry is defined in [`incident-failure-catalog.md`](incident-failure-catalog.md).

| Code | Question to ask the record |
|---|---|
| `N-01` | Does any message name an incident commander? |
| `N-02` | Did an actor outside the operations role change production? |
| `N-03` | Does the commander's name also appear on the production commands? |
| `N-04` | Does the record name one command post? |
| `N-05` | Does the incident tooling run on the failing service? |
| `N-06` | Did each shift change name a new commander and record the acknowledgment? |
| `N-07` | Does a live incident document exist, or only a chat scrollback? |
| `N-08` | Does the record hold every attempted remedy and its result? |
| `N-09` | Does a list name each divergence from the norm, for reversal? |
| `N-10` | Is the detailed status less than four hours old? |
| `N-11` | Are the exit criteria measurable? |

"Not applicable" is a valid result. Report only the entries that the evidence supports.

---

## 11. Where to read next

| File | Question it answers |
|---|---|
| [`02-emergency-response.md`](02-emergency-response.md) | The system breaks now. Which mitigation move do I pick? |
| [`03-on-call.md`](03-on-call.md) | How do I design the rotation that produces the commander? |
| [`04-postmortem-culture.md`](04-postmortem-culture.md) | The incident closed. What do I write, and who reviews it? |
| [`05-tracking-outages.md`](05-tracking-outages.md) | I need the trend across many incidents, not one story. |
| [`06-interrupts-and-overload.md`](06-interrupts-and-overload.md) | The team produces nothing but response. |
| [`incident-failure-catalog.md`](incident-failure-catalog.md) | The 52 named failure modes, with an index by symptom. |

Three boundaries. `slo-engineering` owns the SLO that Section 5 uses as an exit criterion.
`production-troubleshooting` owns the method that finds the cause under this command structure.
A compromise keeps this command structure, and `security-engineering` owns the technical work.
