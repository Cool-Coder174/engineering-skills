# Emergency Response

**Sources:** *Site Reliability Engineering*, Ch. 13, "Emergency Response", p. 189–199 (Baye).
Supporting material from Ch. 12, p. 169–188, Ch. 14, p. 200–207, Ch. 15, p. 209–213, Ch. 6,
p. 88, and App. D, p. 579–584. *Release It!*, Sec. 2.1–2.3, p. 24–28 (Nygard).

This file answers one question. A system breaks now, and what does the responder do. Section 1 gives
the opening moves. Sections 2 to 5 give the three Google case studies and the lesson of each one.
Sections 6 to 12 give the response order, the mitigation moves, the shared rules, and the scan.

---

## 1. What to do when systems break

The chapter gives four rules for the responder (SRE Ch. 13, p. 189, p. 197).

**Do not panic.** The responder is trained for this situation. The book states that typically no
one is in physical danger.

**Add more people when you feel overwhelmed.** The book keeps the hedge. Sometimes it may even be
necessary to page the whole company.

**Know the incident response process before the page, and follow it.** A process that the company
never communicated is a process that nobody follows (p. 191).

**Set aside time after the mitigation.** The responder cleans up and writes the incident record.

`01-incident-command.md` holds the declaration gate, the four roles, and the live record.

---

## 2. The three classes of emergency

The chapter gives three case studies. Each one teaches a different control.

| Class | Trigger | Blast radius in the book | What it teaches | Pages |
|---|---|---|---|---|
| Test-induced | An SRE breaks a system on purpose, to find hidden dependencies | Many dependent services, external and internal | Test the rollback before the test | 189–191 |
| Change-induced | A configuration change passes its tests and still breaks production | The entire externally facing fleet | Canary every change, whatever its perceived risk | 191–194 |
| Process-induced | Fleet automation acts on a wrong input at machine speed | All small server installations, worldwide | Sanity-check the commands that automation sends | 194–197 |

**Read the class from the trigger, not from the symptom.** The three cases produced similar
symptoms. The correct mitigation differs for each class. Section 8 gives the moves.

---

## 3. Test-induced emergency

**The case (SRE Ch. 13, p. 189–191).** Google runs proactive disaster testing. SREs break systems,
watch how they fail, and change the systems to prevent a repeat. Here SRE wanted to find hidden
dependencies on one test database, so the plan blocked all access to one database out of a hundred.
Within minutes many dependent services reported that external and internal users could not reach key
systems. Some systems answered only intermittently or partially. SRE aborted the exercise at once.
The team then tried to revert the permissions change, and the revert failed. Instead the team
restored permissions to the replicas and the failovers, with an approach that it had already tested.
In parallel the team contacted key developers, to correct the flaw in the library that the
application uses to reach the database. Within an hour of the original decision, all access was
restored. Some teams also reconfigured their own systems to avoid the test database.

### What this case teaches

**A rollback that nobody tested will fail at the moment you need it.** The revert procedures had
never run in a test environment. They were flawed, and they made the outage longer. The book
states the new rule directly (SRE Ch. 13, p. 191).

> "We now require thorough testing of rollback procedures before such large-scale tests."

**A process that nobody read is not a process.** The responders did not follow the incident
response process. The company had established it only a few weeks before, and had not communicated
it. So services and customers did not learn about the outage (SRE Ch. 13, p. 191).

**A well-reviewed test can still be wrong about its scope.** The book keeps this hedge. The test
was thoroughly reviewed and thought to be well scoped. Reality showed an insufficient understanding
of the interaction among the dependent systems (SRE Ch. 13, p. 191).

**Prepare a second recovery path before the test.** The team recovered with an approach that it had
already tested, not with the failed revert (SRE Ch. 13, p. 190).

**Catalog codes:** N-15 untested rollback path. N-39 no drills. `reverse-branching` owns how to build
and test the revert. This file owns the gate.

---

## 4. Change-induced emergency

**The case (SRE Ch. 13, p. 191–194).** Google pushed a configuration change globally on a Friday. The
change touched the infrastructure that protects services from abuse, and that infrastructure
interacts with essentially all externally facing systems. The change triggered a crash-loop bug, and
the entire fleet began to crash-loop almost at the same moment. Internal applications also failed,
because the internal infrastructure depends on Google's own services. Monitoring alerted within
seconds. Some on-call engineers believed that the corporate network had failed, and they moved to
panic rooms with backup access to production. Within five minutes of the first push, the push
engineer pushed a second configuration change to revert the first one. Services began to recover.
Within 10 minutes the on-call engineers declared an incident and followed the internal procedures.
Some services carried unrelated bugs that the event triggered, and they did not fully recover for up
to an hour.

**What went well, in the book's own order.**

| Item | Why it helped | Page |
|---|---|---|
| Monitoring detected the fault within seconds | The revert started five minutes after the push | 193 |
| Out-of-band communications | They kept everyone connected while the software stacks were unusable | 193 |
| Command-line tools and alternative access | They allowed updates and reverts when other interfaces were unreachable | 193 |
| Rate limiting of full updates to new clients | It may have throttled the crash-loop and prevented a complete outage | 193 |
| Luck | The push engineer happened to follow real-time channels | 193 |

**Keep the hedge on the rate limit.** The book says the behavior *may have* throttled the crash-loop.
It does not claim proof (SRE Ch. 13, p. 193).

**Luck is not a control.** The book calls the push engineer's habit an extra level of diligence that
is not a normal part of the release process. Without it the outage could have lasted considerably
longer (SRE Ch. 13, p. 193). Make the loop from signal to revert part of the process.

### What this case teaches

**Canary every change, whatever its perceived risk.** An earlier push of the feature used a
thorough canary and did not trigger the bug. That canary never exercised a rare configuration
keyword together with the new feature. The triggering change was not considered risky, so it
followed a less stringent canary process (SRE Ch. 13, p. 194).

**An alert storm attacks the responders.** Every location was offline for a few minutes, so alerts
fired repeatedly and constantly. They overwhelmed the on-calls. They spammed the regular channel
and the emergency channel. They made communication harder (SRE Ch. 13, p. 193–194). Target one
alert per incident (SRE Ch. 11, p. 167).

**Do not put the response tools behind the fault.** Much of the troubleshooting and communication
stack sat behind the crash-looping jobs. A longer outage would have hindered debugging severely
(SRE Ch. 13, p. 194). Section 9 gives the controls.

**Catalog codes:** N-05 incident tooling inside the blast radius. N-14 rollback delayed for a
diagnosis. N-24 unactionable page. N-25 alert fan-out with no grouping.

---

## 5. Process-induced emergency

**The case (SRE Ch. 13, p. 194–197).** During routine automation testing, an engineer submitted two
consecutive turndown requests for the same server installation. On the second request a subtle bug
in the automation sent all machines in all such installations, worldwide, to the Diskerase queue.
Their drives were destined to be wiped. The on-call engineers received a page as the first small
installation went offline, and they determined that the machines had moved to that queue. Following
normal procedure, they drained traffic away from the location, because the wiped machines could not
answer requests. As pagers fired worldwide, the on-call engineers disabled all team automation to
prevent further damage. They then stopped or froze additional automation and production maintenance.
Within an hour all traffic ran to other locations. Users saw elevated latency, and their requests
succeeded. The book calls the outage officially over at that point. Recovery was the hard part.
Network links reported heavy congestion. One installation was rebuilt within three hours. The team
divided into three parts, and each part owned one step of a manual reinstall. The vast majority of
capacity returned within three days. Stragglers took the next month or two.

**The root cause.** The turndown automation server lacked the appropriate sanity checks on the
commands that it sent. On the second run it received an empty response for the machine rack. It did
not filter that response. It passed the empty filter to the machine database, which read that empty
filter as every machine (SRE Ch. 13, p. 196).

> "Yes, sometimes zero does mean all."

### What this case teaches

**Automation multiplies speed and blast radius together.** The book states the trade in one
sentence (SRE Ch. 13, p. 194).

> "This is one example where moving fast was not such a good thing."

**An empty selector must never mean everything.** Filter an empty response inside the automation.
Never pass an empty filter to the system that acts (SRE Ch. 13, p. 196).

**A kill switch for your own automation is an incident control.** The on-call engineers disabled all
team automation, then froze further maintenance (SRE Ch. 13, p. 195). A runbook without that step
cannot stop the damage.

**Revert the monitoring changes that the automation made.** The turndown automation also removed the
monitoring for the small installations. The on-call engineers reverted those monitoring changes
promptly, so they could measure the damage (SRE Ch. 13, p. 196).

**Some damage is not revertible.** Diskerase destroyed the drives, so no revert existed. Recovery
used a manual reinstall (SRE Ch. 13, p. 195–197).

**The recovery path has its own limits, and they surface only in recovery.** The reinstall used TFTP
at the lowest network quality of service, from distant locations. The BIOS either halted or entered
a constant reboot cycle, and each cycle taxed the installers further. A regression also stopped the
infrastructure from running more than two setup tasks per worker machine (SRE Ch. 13, p. 196–197).
The engineers raised the priority of installation traffic, and restarted the stuck machines.

**Catalog codes:** N-16 automation runs while it causes damage. N-02 freelancing. N-13 lost evidence.

---

## 6. What the three cases share

The chapter closes with the traits that the three responses share (SRE Ch. 13, p. 198–199). Use this
list as the review question after any emergency.

1. The responders did not panic.
2. They added other people when they judged it necessary.
3. They studied and learned from the earlier outages.
4. They built their systems to respond better to those outage types.
5. They documented each new failure mode, for other teams.
6. They tested their systems on purpose.

**Each outage or test produces an incremental improvement to both the process and the system**
(SRE Ch. 13, p. 199). One large fix is not the expected output of an emergency.

---

## 7. Restore service before you find the cause

**Stop the bleeding first.** The book names the instinct to root-cause, and rejects it. "Ignore that
instinct!" (SRE Ch. 12, p. 173). A responder does not help users while the system dies.

**Restoring service takes precedence over investigation** (Release It! Sec. 2.1, p. 24). Collect
data for the postmortem only when the collection does not make the outage longer.

| Order | Action | Signal that the step is complete |
|---|---|---|
| 1 | Stop the bleeding | The error rate stops rising |
| 2 | Restore service | The service meets its SLO |
| 3 | Preserve the evidence | Logs, dumps, and graphs sit outside the failing system |
| 4 | Find the cause | The postmortem states the contributing causes |

**Step 3 is not optional, and rapid triage does not cancel it** (SRE Ch. 12, p. 173). After a
restart the failing state no longer exists. The postmortem is then harder than a murder, "because
the body goes away" (Release It! Sec. 2.3, p. 28).

**Pre-script the collection.** Nygard's team held scripts that took thread dumps and database
snapshots. The collection was not improvised, it did not make the outage longer, and it fed the
analysis (Release It! Sec. 2.1, p. 24).

**Measure the exit with the four golden signals.** They are latency, traffic, errors, and saturation
(SRE Ch. 6, p. 88). Write the exit criteria as a measurable condition. The book's example holds the
availability and latency SLOs for 30 or more minutes (SRE App. D, p. 583).

---

## 8. Choose the mitigation move

Match the observed signal to the move. Do not select a move from the class of the emergency.

| Signal | Move | Source |
|---|---|---|
| One location fails and other locations are healthy | Divert traffic away from that location | SRE Ch. 12, p. 173 |
| Machines in one location cannot answer at all | Drain traffic, so requests do not fail outright | SRE Ch. 13, p. 195 |
| Total load is above total capacity | Drop a fraction of traffic upstream, retries included | SRE App. B, p. 575 |
| One subsystem drives the load | Disable that subsystem | SRE Ch. 12, p. 173 |
| A change sits inside the fault window | Revert the change first. Diagnose after. | SRE Ch. 13, p. 192 |
| A bug can corrupt data that you cannot recover | Freeze the system | SRE Ch. 12, p. 173 |
| Your own automation causes the damage | Disable all team automation, then freeze maintenance | SRE Ch. 13, p. 195 |
| Your automation removed the monitoring | Revert the monitoring changes first, so you can measure | SRE Ch. 13, p. 196 |
| A crash-loop spreads across the fleet | Revert the config, and let the update rate limit hold the fleet | SRE Ch. 13, p. 192–193 |

**Check that the fallback can absorb the load.** The large installations were designed to carry a
full load. The diverted traffic still congested some network links, and network engineers had to
mitigate each choke point (SRE Ch. 13, p. 196).

**Restore traffic in a ladder after a mitigation.** Use 1, 10, 30, 50, then 100 percent. Check the
SLO between the steps (SRE App. D, p. 583).

The mechanics of the revert belong to `reverse-branching`. Timeouts, retries, and the Circuit Breaker
belong to `self-healing-apis`. `production-troubleshooting` Table 4 states which evidence each move
destroys. This file selects the move and records the attempt.

---

## 9. Keep the response tools outside the fault

The change-induced case is the proof (SRE Ch. 13, p. 194).

| Control | What it must survive | Source |
|---|---|---|
| Out-of-band communications | The loss of the main software stack | SRE Ch. 13, p. 193 |
| Command-line tools and alternative access | Unreachable normal interfaces | SRE Ch. 13, p. 193 |
| Panic rooms with backup production access | The loss of the corporate network | SRE Ch. 13, p. 192 |
| The live incident record | The failure of the service under repair | SRE Ch. 14, p. 203 |

**Use these systems regularly, not only in an emergency** (SRE Ch. 13, p. 193). The book adds a
caveat. The engineers needed more familiarity with the tools, and more routine testing of them.

**Modern.** A chat platform, a paging vendor, and a status page each carry the same rule. Host them
outside the failure domain of the service, and test them on a schedule. Neither book names them.

---

## 10. All problems have solutions

**A solution exists, even when it is not obvious to the person whose pager is screaming**
(SRE Ch. 13, p. 197). That is a rule about behavior, not a promise about the system.

**Ask for help before the situation forces you.** Involve more teammates. Seek help. Do it quickly.
The highest priority is a quick resolution of the issue at hand (SRE Ch. 13, p. 197).

**Use the person with the most state.** That person is often the one whose action triggered the event
(SRE Ch. 13, p. 197). The postmortem stays blameless, so this rule costs that person nothing.

| Signal | Who joins | Source |
|---|---|---|
| The responder cannot follow the state | A second person takes command | SRE Ch. 14, p. 201 |
| One responder carries too much work | The planning lead supplies staff | SRE Ch. 14, p. 202 |
| The fault crosses a team boundary | The on-call of that team | SRE Ch. 13, p. 191 |
| The fault sits in a library or an application | The developers who own it | SRE Ch. 13, p. 190 |
| Nothing works, and the impact grows | The book permits a page to the whole company | SRE Ch. 13, p. 189 |

---

## 11. Learn from the past. Do not repeat it.

The chapter gives three practices (SRE Ch. 13, p. 197–198).

### Keep a history of outages

1. Document what has broken.
2. Be thorough and be honest.
3. Ask hard questions.
4. Search for the actions that prevent a repeat, tactically and strategically.
5. Publish and organize the postmortems, so everyone in the company can learn.
6. Hold yourself and others accountable for the follow-up actions in those postmortems.

**Accountability for the action items is the step that prevents a near-identical outage**
(SRE Ch. 13, p. 198). An item with no owner and no date is not an action item (SRE App. D, p. 580).

### Ask the big, even improbable, questions

The book lists open scenarios. The building power fails. The network racks stand in two feet of
water. The primary datacenter goes dark. Somebody compromises your web server (SRE Ch. 13, p. 198).
For each scenario the book asks the same eight questions. What do you do? Who do you call? Who will
write the check? Do you have a plan? Do you know how to react? Do you know how your systems will
react? Could you minimize the impact if it happened now? Could the person sitting next to you do
the same?

**The last question is the one that finds a person who is a single point of failure.** The other
seven force a first move, an escalation path, a cost owner, a plan, a trained reaction, a
prediction, and a readiness measure.

### Encourage proactive testing

**There is no greater test than reality** (SRE Ch. 13, p. 198). Until a system has actually failed,
nobody knows how that system, its dependent systems, or its users will react. Do not rely on an
assumption that you have not tested. The book frames the choice as a schedule question. A failure
with your best engineers present beats a failure at 2 a.m. on a Saturday.

Section 3 is the cost side of this practice. A proactive test can produce a real outage. Test the
rollback first, and communicate the incident process first.

---

## 12. The emergency scan

Run these codes from `incident-failure-catalog.md` against a runbook, an alert rule, or a record.

| Symptom in this file | Code | The question to ask |
|---|---|---|
| The revert failed when the team needed it | N-15 | Does a test record show the rollback running in a test environment? |
| The response tools sat behind the fault | N-05 | Does the record name a command post outside the failing system? |
| The team analyzed before it reverted | N-14 | Did a change sit inside the fault window? |
| The alerts overwhelmed the responders | N-25 | Does one event produce more than one alert? |
| The automation continued to damage the fleet | N-16 | Does the runbook hold a step that disables all team automation? |
| The mitigation destroyed the failing state | N-13 | Did a responder copy the logs, the dumps, and the graphs out first? |
| A responder outside the incident changed production | N-02 | Does the audit log name an actor who is not the operations lead? |
| The team never ran a drill | N-39 | Does the calendar hold a dated disaster exercise? |
| The action items have no owner | N-34 | Does every item carry a type, an owner, a bug, and a date? |

Report only the codes that apply. "Not applicable" is better than an invented finding.

---

## 13. Where to read next, and when this file stays silent

This file covers the live fault only. Every other question belongs to a sibling.

| Situation | Read instead |
|---|---|
| A fault needs more than one person. Who leads, and who may change production? | `01-incident-command.md` |
| The question is pager load, shift design, or how many incidents a shift absorbs | `03-on-call.md` |
| The event has closed, and the team needs the record and the review | `04-postmortem-culture.md` |
| The team needs a trend across many small events, not one story | `05-tracking-outages.md` |
| The team produces nothing, and toil is above the 50 percent cap | `06-interrupts-and-overload.md` |
| Any scan of a runbook, an alert rule, or a closed record | `incident-failure-catalog.md` |
| The fault localizes to one integrated vendor API | `self-healing-apis` |
| The incident is a compromise | `security-engineering`, with the roles from `01-incident-command.md` |
| The fault lost or corrupted data | `data-systems-design` owns the hazard |
| The cause is unknown, and you need the method that finds it | `production-troubleshooting` |
| The alert rule itself is the question | `observability` |

**This file stays silent when no system is broken.** A design review, a capacity estimate, or a
schema question belongs to `system-design` or `planner`. "Not applicable" is the correct result.
