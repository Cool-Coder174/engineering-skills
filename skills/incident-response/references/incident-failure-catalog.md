# Incident Failure Catalog

**Sources:** *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy), Ch. 6, Ch. 10–16,
Ch. 28–33, and Appendices B, C, D, F, pages 88–590. *Release It!* (Nygard), Sec. 2.1–2.5,
Sec. 5.2, Sec. 16.8–16.11, and Sec. 17.4–17.7, pages 21–305.

A catalog of 52 named failure modes for the human response to a production fault, `N-01` to
`N-52`. The content comes from *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy)
and *Release It!* (Nygard).

Each entry has three parts:

- **Signature** — what you search for in a runbook, an alert config, a chat log, an audit
  log, a dashboard, or an incident record. This is the text that you search for.
- **Consequence** — what goes wrong, and when.
- **Fix** — the required remedy.

**How to use it.** `code-review` runs the applicable groups against a change to an alert
rule, a runbook, a paging config, or an on-call rotation. `verify` runs them against a closed
incident record. `planner` reads the action item table of a postmortem as an input.
`detail-planning` runs group C against the rollback section of a phase spec. Report only the
failures that apply. "Not applicable. This change touches no alert, no runbook, and no
on-call rotation." is a valid result. That result is better than an invented finding.

**Modern.** Neither book describes an automated reviewer, an agent-driven pipeline, or a
paging vendor. The scan procedure above, and every named skill in it, carries this tag. The
named failure modes below carry their book pages.

**Severity.** 🔴 destroys evidence, loses data, or makes an outage unrecoverable. It blocks
the merge or the close. 🟡 lengthens the outage or produces a wrong result under load. 🔵 is
a risk to operation or to maintenance.

**This catalog names the human failure, not the machine failure.** `data-systems-design` owns
the data hazards behind a corruption incident. `self-healing-apis` owns the Circuit Breaker
and the other containment patterns (Release It! Sec. 5.2, p. 115). `reverse-branching` owns
the mechanism of a revert. This catalog owns the decision, the record, and the people.

**Two sibling catalogs cover the same ground from another angle.** Do not report the same
defect twice. `production-troubleshooting/references/diagnostic-trap-catalog.md` holds the
`D-` traps of the diagnosis, and `observability/references/observability-gap-catalog.md` holds
the `M-` gaps of the monitoring. Group C names the mitigation decision, group D names the
diagnosis conducted under command, and group E names the alert as the on-call receives it. When
a `D-` or an `M-` entry states the same defect with more detail, cite that entry and stop.

---

## A. Command and coordination

**A responder who holds the strategy and the keyboard holds neither well.** Each entry names
a way that the response, not the machine, degrades.

### 🟡 N-01 — No named incident commander
**Signature:** no message names a commander. The record has no command hierarchy block.
**Consequence:** one responder holds the technical work and the strategy at the same time.
That responder cannot think about the wider mitigation (SRE Ch. 14, p. 201).
**Fix:** transfer command to a second person at the second alert. The commander holds every
role that the commander did not delegate (SRE Ch. 14, p. 202).

### 🔴 N-02 — Freelancing
**Signature:** a deploy, a restart, a flag change, or a config push runs during a declared
incident. The actor is not the operations lead.
**Consequence:** the uncoordinated change lands on a damaged system. In the book's example
the servers restarted, took the change, and died (SRE Ch. 14, p. 201–202).
**Fix:** the operations team is the only group that changes the system during an incident
(SRE Ch. 14, p. 202–203). Every other responder proposes the change to that lead.

### 🟡 N-03 — The commander performs the operational work
**Signature:** one name appears as commander and as the actor on the production commands.
**Consequence:** nobody holds the high-level state. Communication stops. Documentation stops.
**Fix:** delegate the operational work. The first duty of the commander is the living
document, not the fix (SRE Ch. 14, p. 203).

### 🟡 N-04 — No recognized command post
**Signature:** the discussion runs across messages and calls. The record names no room.
**Consequence:** interested parties cannot find the commander. Engineers who could help are
not used well (SRE Ch. 14, p. 201, p. 203).
**Fix:** name one channel or one room in the record. Log the alerts and the actions into it
(SRE Ch. 14, p. 203).

### 🔴 N-05 — Incident tooling inside the blast radius
**Signature:** the record, the chat, or the paging path runs on the service under repair.
**Consequence:** the response loses its own tools. In the change-induced emergency much of
the troubleshooting stack sat behind the crash-looping jobs (SRE Ch. 13, p. 194).
**Fix:** host the record on independent infrastructure (SRE Ch. 14, p. 203). Keep out-of-band
communications and command-line access. Use them regularly (SRE Ch. 13, p. 193).

### 🟡 N-06 — Silent handoff
**Signature:** a shift change names no new commander. No acknowledgment appears.
**Consequence:** nobody leads. The next responder repeats the work of the last one.
**Fix:** brief the incoming commander. State "You are now the incident commander." Wait for
firm acknowledgment. Broadcast the handoff to every responder (SRE Ch. 14, p. 204).

### 🟡 N-41 — An incident process that nobody communicated
**Signature:** a written incident process exists. No announcement reached the responders.
**Consequence:** the responders do not follow it. In the test-induced emergency the services
and the customers learned nothing about the outage (SRE Ch. 13, p. 191).
**Fix:** communicate every change of the incident procedure to every relevant party. Refine
and test the process before you depend on it (SRE Ch. 13, p. 191).

---

## B. The live incident record

**The record is the deliverable of the commander.** A response that produces no record cannot
transfer, cannot close, and cannot teach.

### 🟡 N-07 — No live incident state document
**Signature:** the only record of the event is a chat scrollback.
**Consequence:** a new responder cannot learn the current state. The postmortem timeline
cannot be built.
**Fix:** open a living document from a template. Keep the most important information at the
top. Retain it for the postmortem (SRE Ch. 14, p. 203–204).

### 🔴 N-08 — A failed remedy is not recorded
**Signature:** an attempted revert or restart appears in the audit log and not in the record.
**Consequence:** the next responder repeats it. In the unmanaged story the rollback did not
help, and that fact stayed in one person's head (SRE Ch. 14, p. 200).
**Fix:** post every attempt and every result into the channel. Paste each one into the
document (SRE Ch. 14, p. 204–205). A rollback that does not help is data, not an ending.

### 🔴 N-09 — No divergence ledger
**Signature:** emergency flags, overrides, drains, and manual edits exist. No list names them.
**Consequence:** the system stays away from its normal state after the incident closes. The
next fault starts from an undocumented configuration.
**Fix:** the planning lead tracks how the system diverged from the norm. A responder reverts
each row when the incident resolves (SRE Ch. 14, p. 203).

### 🔵 N-10 — Stale detailed status
**Signature:** the "last updated" stamp on the status block is more than four hours old.
**Consequence:** stakeholders ask the responders for an update. The responders lose the time
that the answer costs.
**Fix:** update the detailed status at least every four hours. Update it at every handoff of
the communications role (SRE App. C, p. 577).

### 🟡 N-11 — No written exit criteria
**Signature:** the record holds no measurable condition for closing the incident.
**Consequence:** the incident closes on a feeling. The service degrades again after the
responders leave.
**Fix:** write the exit criteria as a checklist. The book's example holds the availability and
latency SLOs for 30 or more minutes (SRE App. C, p. 577, and SRE App. D, p. 583).

### 🔴 N-42 — End-user identifying data in the record
**Signature:** the record holds a name, an address, or a request that identifies one person.
**Consequence:** the team cannot share the record. A privacy rule then blocks the learning
loop that the record exists to serve.
**Fix:** never include information that identifies an end user (SRE Ch. 15, p. 211). Name a
case by an internal identifier only.

---

## C. Response order and mitigation

**Stop the bleeding, restore service, preserve the evidence, then find the cause** (SRE
Ch. 14, p. 206). Each entry breaks that order or breaks the mitigation.

### 🔴 N-12 — Diagnosis before mitigation
**Signature:** the first entries of the record hold only queries, graphs, and hypotheses.
**Consequence:** the system dies while the responder searches for the cause. "You aren't
helping your users if the system dies while you're root-causing" (SRE Ch. 12, p. 173).
**Fix:** make the system work as well as it can first. Divert traffic, drop traffic, or
disable a subsystem (SRE Ch. 12, p. 173, and SRE Ch. 14, p. 206).

### 🔴 N-13 — The mitigation destroys the evidence
**Signature:** a restart, a wipe, or a redeploy runs with no copy of the logs and the graphs.
**Consequence:** the postmortem cannot state a cause. The failing state no longer exists.
"The body goes away" (Release It! Sec. 2.3, p. 27–28).
**Fix:** preserve the evidence while you triage (SRE Ch. 12, p. 173). Pre-script the
collection so it does not make the outage longer (Release It! Sec. 2.1, p. 24).

### 🟡 N-14 — Rollback delayed for a diagnosis
**Signature:** a change sits inside the fault window. Analysis precedes the revert.
**Consequence:** the mean time to recovery grows. A tight SLO does not permit a deep
diagnosis before a revert (SRE Ch. 30, p. 509).
**Fix:** revert first. Diagnose after (SRE App. B, p. 572). The book's four-minute outage
lasted four minutes because the engineer reverted at once (SRE Ch. 15, p. 213).

### 🔴 N-15 — Untested rollback path
**Signature:** a runbook holds a rollback step. No test record shows that step running.
**Consequence:** the rollback fails at the moment you need it. Untested procedures were
flawed, and the flaw lengthened the outage (SRE Ch. 13, p. 191).
**Fix:** test the rollback before any large-scale change or test (SRE Ch. 13, p. 191). Keep
command-line access that works when the normal interfaces do not (SRE Ch. 13, p. 193).

### 🔴 N-16 — Automation runs while it causes the damage
**Signature:** no runbook step disables the team automation. No fleet kill switch exists.
**Consequence:** the automation continues at machine speed. The turndown automation sent
every machine in every small installation to the wipe queue (SRE Ch. 13, p. 194–196).
**Fix:** add a step that disables all team automation and freezes further maintenance (SRE
Ch. 13, p. 195). Revert the monitoring changes that the automation made (SRE Ch. 13, p. 196).

### 🟡 N-17 — A partial mitigation concentrates the load
**Signature:** a fix reaches one node. The load balancer still sends traffic to that node.
**Consequence:** the repaired node takes all the load. In the book's case the load manager sent
every page request to the one repaired node, and that node was crushed (Release It! Sec. 16.10,
p. 263).
**Fix:** drain the node, or apply the change across the fleet. Restore traffic in a ladder of
1, 10, 30, 50, then 100 percent, and check the SLO between steps (SRE App. D, p. 583).

### 🟡 N-43 — Improvised evidence collection
**Signature:** no script collects the logs and the dumps. The responder improvises them.
**Consequence:** the collection costs time that the outage cannot afford. "When the fur
flies, improvisation is not your friend" (Release It! Sec. 2.1, p. 24).
**Fix:** pre-script the collection. A pre-written script does not prolong an outage, and it
still serves the postmortem (Release It! Sec. 2.1, p. 24).

### 🟡 N-44 — A restart of every server as the first remedy
**Signature:** the runbook answers a wide symptom by restarting every server, layer by layer.
**Consequence:** the restart usually works and it takes a long time (Release It! Sec. 2.1,
p. 24). The failing state disappears before anyone diagnoses it.
**Fix:** find the component that holds the fault. Restart the component, not the whole
server, when the platform permits it (Release It! Sec. 16.10, p. 263).

### 🟡 N-45 — An unchecked calculation drives the mitigation
**Signature:** a capacity change in the timeline cites no source for its number.
**Consequence:** an order-of-magnitude error returns traffic to a system that cannot hold it.
In the book's example the tasks failed again one minute later (SRE App. D, p. 582–583).
**Fix:** ask a second responder to check every number that changes capacity or traffic.
Revert the change at once when the symptom returns (SRE App. D, p. 583).

---

## D. Diagnosis under pressure

**Stress replaces deliberate thought with habit, and a habitual response is unconsidered**
(SRE Ch. 11, p. 164–165). Each entry names a way to reach a wrong conclusion.

### 🟡 N-18 — Post hoc reasoning taken as proof
**Signature:** the record names the last change as the cause. No test rules out another.
**Consequence:** the team fixes the trigger and leaves the cause. In the airline case the
failover was the trigger, and a single uncaught `SQLException` was the cause (Release It!
Sec. 2.3–2.5, p. 27–34).
**Fix:** treat the last change as a good place to start and a bad place to stop (Release It!
Sec. 2.3, p. 27). Correlation is not causation (SRE Ch. 12, p. 171).

### 🔴 N-19 — Voodoo remediation in the runbook
**Signature:** a runbook step with no stated mechanism, such as a weekly database failover.
**Consequence:** the team repeats a costly action forever on a temporal coincidence. "There
was no causal connection, but there was a temporal connection" (Release It! Sec. 17.4,
p. 281–283).
**Fix:** every runbook step states the mechanism that it acts on. Remove a step that no
evidence supports.

### 🟡 N-20 — Confirmation bias on a repeat alert
**Signature:** the record blames the previous cause within minutes. No test appears.
**Consequence:** the responder follows a line of reasoning that was wrong from the start (SRE
Ch. 11, p. 165).
**Fix:** name the assumption in the record. Run one test that can disconfirm it (SRE Ch. 12,
p. 179).

### 🟡 N-21 — No request identifier in the logs
**Signature:** log records carry no request, session, or transaction identifier.
**Consequence:** the responder cannot join an upstream entry to a downstream entry. The
responder reads thousands of lines with no string to search for (Release It! Sec. 17.4,
p. 283).
**Fix:** propagate a unique request identifier through the whole span of calls (SRE Ch. 12,
p. 186). Log the interesting state transitions (Release It! Sec. 17.4, p. 283).

### 🔵 N-22 — No notes of the tests and the results
**Signature:** the record lists actions and no results. Unlisted test changes remain.
**Consequence:** the responder repeats steps. The system runs in an unknown mixed
configuration (SRE Ch. 12, p. 180).
**Fix:** record every idea, every test, and every result. Change the system in a documented
way, so you can restore the state before the test (SRE Ch. 12, p. 180).

### 🟡 N-23 — A test with a side effect that changes the next test
**Signature:** the responder enables verbose logging or adds CPU, then reads the same metric.
**Consequence:** you cannot tell whether the fault grew on its own or because of your
instrumentation (SRE Ch. 12, p. 179–180).
**Fix:** name the side effect before the test. Prefer a test with mutually exclusive
alternatives (SRE Ch. 12, p. 179).

---

## E. Detection and alert hygiene

**Every paging alert must be actionable, and every paging alert must align with a symptom
that threatens the SLO** (SRE Ch. 11, p. 166). Monitoring has three outputs only. They are
pages, tickets, and logs (SRE App. B, p. 573).

### 🟡 N-24 — Unactionable page
**Signature:** an alert rule with `severity=page` that the on-call closes with no action.
**Consequence:** alert fatigue. Serious alerts then receive less attention than they need
(SRE Ch. 11, p. 166).
**Fix:** ask two questions for every page. Should it have paged in that way, and should it
have paged at all. Remove the unactionable pages (SRE Ch. 31, p. 515).

### 🟡 N-25 — Alert fan-out with no grouping
**Signature:** one event produces many notifications across many queues, with no grouping.
**Consequence:** teams debug the same event in parallel. The overview display becomes
unusable (SRE Ch. 16, p. 218–219).
**Fix:** group multiple alerts into one incident. One email per alert does not scale (SRE
Ch. 16, p. 219). Target one alert per incident (SRE Ch. 11, p. 167).

### 🔴 N-26 — Alerts routed to email
**Signature:** an alert destination that is a mailing list, with no paging path and no bug.
**Consequence:** the team ignores the alerts. The strategy works for a while, and the
inevitable outage becomes more severe (SRE App. B, p. 574).
**Fix:** anything that is worth interrupting a person must page, or must become a tracked bug
(SRE App. B, p. 573–574).

### 🟡 N-27 — False positives train the team to ignore the alarm
**Signature:** an alert with a high firing count and a low action count.
**Consequence:** the team ignores the real event. In the retail case the false positives had
trained the group to ignore high CPU, and the site went down (Release It! Sec. 16.8, p. 261,
and Sec. 17.7, p. 304).
**Fix:** set the expectations from the historical data so they match reality. Start loose,
then tighten (Release It! Sec. 17.7, p. 303–304).

### 🟡 N-28 — Undiagnosed recurring alert
**Signature:** an alert that no team has ever diagnosed. Responders label it transient.
**Consequence:** it distracts the team from real problems. It is an emergency waiting to
happen (SRE Ch. 30, p. 505).
**Fix:** investigate the alert fully, or fix the alerting rule. There is no third option (SRE
Ch. 30, p. 505).

### 🔵 N-29 — Silent alert suppression
**Signature:** a threshold change or a silence in the config, with no bug and no owner.
**Consequence:** a real signal disappears with no record. Nobody can determine why.
**Fix:** record every threshold change as a tracked item with a named owner. The book raises
a delay threshold from 60 to 180 seconds and files bug 4821600 (SRE App. F, p. 589). Silence
an interrupt only until the date of its fix (SRE Ch. 29, p. 501).

### 🟡 N-30 — A probe that hides the failing response
**Signature:** a black-box prober that stores pass or fail, and no status, header, or body.
**Consequence:** the responder must reproduce the failure by hand to learn that it was a 502
with no payload (SRE Ch. 12, p. 175–176).
**Fix:** store the failing status, the headers, and the body. Validate the payload, not only
the status code (SRE Ch. 10, p. 156).

### 🔵 N-46 — A problem report sent to a person
**Signature:** users report faults by email or by chat to one named engineer. No bug exists.
**Consequence:** the report needs a transcription step. The rest of the team cannot see it.
The load lands on the engineers that the reporters know (SRE Ch. 12, p. 172).
**Fix:** open a bug for every issue, including an issue that arrives by email or by chat.
Route the report to the person on duty (SRE Ch. 12, p. 172).

### 🟡 N-47 — The dependency owner is not alerted
**Signature:** the record blames an upstream service. No alert reached its owner.
**Consequence:** the owning team never starts work. The resolution waits for a signal that
nobody sent (SRE Ch. 16, p. 221).
**Fix:** check whether the dependency owner received an alert. Alert that team by hand when
the owner's own alerting stays silent (SRE Ch. 16, p. 221).

**Alert design note.** Page a human when one of the four golden signals is problematic. For
saturation, page when the signal is nearly problematic (SRE Ch. 6, p. 88–89).

---

## F. The postmortem and the learning loop

**Blameless does not mean toothless.** The record still states how the service must improve
(SRE Ch. 15, p. 210).

### 🟡 N-31 — No postmortem triggers agreed in advance
**Signature:** no written trigger list. The team argues after each event.
**Consequence:** the events with the most to teach receive no record. Incidents recur (SRE
Ch. 15, p. 208).
**Fix:** define the criteria before an incident, so everyone knows when a postmortem is
necessary (SRE Ch. 15, p. 209).

### 🔴 N-32 — Blame in the postmortem
**Signature:** the record names a person as the cause, in the form "X should have checked."
**Consequence:** people stop bringing issues to light. Incidents get swept away, and the
organization carries more risk (SRE Ch. 15, p. 209–210).
**Fix:** name the system fault, not the person. Ask the on-call to write what they thought at
each point in time, so the team finds where the system misled them (SRE Ch. 30, p. 507). The
Bad Apple Theory is demonstrably false (SRE Ch. 30, p. 507).

### 🔴 N-33 — Rollback-only action items
**Signature:** every action item reverts the change that triggered the outage.
**Consequence:** the change goes away and the defect stays. The same class of outage returns
(SRE Ch. 30, p. 506).
**Fix:** replace the revert item with an item that explains the behavior. The book replaces
"Change the streaming timeout back to 60 seconds" with an item that asks why the fetch
sometimes takes 60 seconds (SRE Ch. 30, p. 506).

### 🔴 N-34 — An action item with no owner and no date
**Signature:** an action item table with empty owner cells, or with a team name.
**Consequence:** nobody does the work. The next incident repeats the last one.
**Fix:** every item carries a type, an owner, a bug, and a state (SRE App. D, p. 579–581).
Hold people accountable for the follow-up (SRE Ch. 13, p. 198).

### 🟡 N-35 — Unreviewed postmortem
**Signature:** a draft with open comments, and no review session on the calendar.
**Consequence:** "An unreviewed postmortem might as well never have existed" (SRE Ch. 15,
p. 211).
**Fix:** senior engineers review the draft against five criteria. Was the key data collected.
Are the impact assessments complete. Was the root cause deep enough. Is the action plan
appropriate. Did we share the outcome (SRE Ch. 15, p. 211). Then add it to the repository
(SRE Ch. 15, p. 212).

### 🔵 N-36 — Postmortem coverage gap
**Signature:** the archive holds only large incidents. Frequent small events have no record.
**Consequence:** issues that are individually small and widely spread stay invisible. Fixes
with large horizontal benefit never get funded (SRE Ch. 16, p. 216, and p. 221 footnote 84).
**Fix:** add an outage tracker that receives every alert. Group, tag, and analyze the data
(SRE Ch. 16, p. 217–220).

### 🔵 N-48 — A near miss with no record
**Signature:** the postmortem holds no "Where we got lucky" section.
**Consequence:** the same latent error waits for its next enabling condition. Near misses are
"disasters waiting to happen" (SRE Ch. 33, p. 558).
**Fix:** write the near misses into the record. Treat each one as a preemptive postmortem
(SRE App. D, p. 581, and SRE Ch. 33, p. 557–558).

### 🔵 N-49 — A raw count with no baseline
**Signature:** a report states an incident count with no prior period next to it.
**Consequence:** nobody can interpret the number. "That's the third time this week" can be
good or bad (SRE Ch. 16, p. 220).
**Fix:** report every count against a baseline. Count incidents per day and alerts per day as
two separate numbers (SRE Ch. 16, p. 218, p. 220).

---

## G. On-call load and readiness

**A team above the 50 percent operational cap makes no engineering progress** (SRE Ch. 11,
p. 160). Load and practice decide the quality of the next response.

### 🟡 N-37 — More than two incidents per shift
**Signature:** the tracker shows more than two incidents in a 12-hour shift.
**Consequence:** response quality falls. Something else will break, and the shift cannot
absorb it (SRE Ch. 11, p. 163). It also indicates a fault in the design, in the monitoring
sensitivity, or in the unclosed postmortem bugs (SRE App. B, p. 576).
**Fix:** apply the overload ladder. Install corrective measures for the next quarter (SRE
Ch. 11, p. 163).

### 🟡 N-38 — On-call work and project work at the same time
**Signature:** an engineer holds the pager and a delivery date in the same week.
**Consequence:** constant interruption. A 20-minute interruption costs about two hours of
productive work (SRE Ch. 29, p. 498).
**Fix:** write off the on-call week for project work. Polarize the time into interrupt weeks
and project weeks (SRE Ch. 29, p. 498–499). "A person should never be expected to be on-call
and also make progress on projects" (SRE Ch. 29, p. 499).

### 🟡 N-39 — No drills, and therefore skill decay
**Signature:** no drill on the calendar. The last disaster exercise has no date.
**Consequence:** proficiency atrophies quickly when the framework is not in constant use (SRE
Ch. 14, p. 206). The team makes a long outage out of a short one (SRE App. B, p. 576).
**Fix:** run the framework during routine change management (SRE Ch. 14, p. 206). Run
disaster role playing weekly, for 30 to 60 minutes (SRE Ch. 28, p. 486). Run a company drill
each year (SRE Ch. 33, p. 553).

### 🟡 N-40 — Operational load above the 50 percent cap
**Signature:** quarterly records show operational work above half. Daily tickets exceed five.
**Consequence:** no engineering progress. The team responds to growth by adding
administrators, which is ops mode (SRE Ch. 30, p. 504).
**Fix:** apply the overload ladder. Write the SLO first (SRE Ch. 30, p. 508). More tickets
must not require more engineers (SRE Ch. 30, p. 504).

### 🟡 N-50 — Tickets assigned at random across the team
**Signature:** the ticket system assigns each new ticket to any team member.
**Consequence:** every assignment forces a context switch. The team never reaches the flow
state that project work needs (SRE Ch. 29, p. 497–500).
**Fix:** make tickets a full-time role for one person, or for two people when the load is
high. Do not spread the load across the whole team (SRE Ch. 29, p. 499–500).

### 🔵 N-51 — Operational underload
**Signature:** an engineer holds the pager less than once per quarter.
**Consequence:** the engineer becomes overconfident or underconfident. The knowledge gaps
appear only when an incident occurs (SRE Ch. 11, p. 168).
**Fix:** size the team so every engineer is on-call once or twice per quarter. Run Wheel of
Misfortune exercises and the yearly company drill (SRE Ch. 11, p. 168).

### 🔵 N-52 — No ticket handoff and no scrub
**Signature:** the rotation transfers no state. No meeting scrubs the interrupt classes.
**Consequence:** each person survives the shift and returns to other duties. The team meets
the same issues forever, with no forward movement (SRE Ch. 29, p. 501).
**Fix:** define a handoff for tickets as well as for on-call. Run a regular scrub. Silence an
interrupt only until the date that its fix is due (SRE Ch. 29, p. 501).

---

## Index by symptom

| Symptom | Check these entries |
|---|---|
| The response is disorganized | N-01, N-03, N-04, N-06, N-41 |
| Someone made it worse | N-02, N-16, N-17, N-45 |
| We lost the evidence | N-05, N-08, N-13, N-43, N-44 |
| The outage lasted too long | N-12, N-14, N-15, N-18, N-20, N-23 |
| We cannot reconstruct what happened | N-07, N-09, N-10, N-21, N-22 |
| The incident closed on a feeling | N-11 |
| We cannot share the record | N-42 |
| The alerts are noise | N-24, N-25, N-26, N-27, N-28, N-29 |
| We did not detect it | N-30, N-31, N-46, N-47 |
| It happened again | N-19, N-33, N-34, N-35, N-36, N-48 |
| The numbers tell us nothing | N-36, N-49 |
| The team is worn out | N-37, N-38, N-40, N-50, N-52 |
| The team is out of practice | N-39, N-51 |
| Nobody wants to write the record | N-32 |
