# The Change-Induced Emergency

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 13 "Emergency Response", p. 189–199. Supporting material from Ch. 12, p. 171–186.
Ch. 14, p. 201–207. Ch. 15, p. 209 and p. 213. App. B, p. 571–572. App. D, p. 580–583.
*Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007), Sec. 18.4, p. 331–334.

Read this file when a change caused the outage. Read `05-locating-the-bad-change.md` when
you do not yet know which change caused it. Read `03-progressive-rollout-and-canary.md` for
the stages that make the reverse action cheap.

**Ownership.** `incident-response` owns the human response. It holds the roles, the command
post, the written incident state, the handoff, and the postmortem. It also holds the full
account of the three Chapter 13 case studies, in
`incident-response/references/02-emergency-response.md`. `production-troubleshooting` owns
the diagnosis method of Chapter 12. This file owns one thing only. The order in which the
reverse moves run inside that command structure.

---

## 1. The rule

**A change caused the fault. The reverse action comes first. The diagnosis comes second.**

App. B, p. 572 states this order with no condition attached. When you detect unexpected
behavior, revert first and diagnose afterward. The book gives one reason. The order
minimizes Mean Time to Recovery.

**The reverse action does not need a cause. A diagnosis needs time you do not have.**
Ch. 12, p. 173 gives the same order for any major outage. The book tells the responder to
resist the instinct to search for a root cause first. It writes "Ignore that instinct!"
Stopping the bleeding is the first priority.

**A revert is also the cheapest experiment.** Ch. 12, p. 171 names two ways to test a
hypothesis. The second way changes the system and observes the result. A revert is that
test. When the hypothesis holds, the same action repairs the service.

**A revert is rewarded behavior, not an admission of fault.** Ch. 15, p. 213 records a
release that stopped a critical service for four minutes. The outage lasted only four
minutes, because the engineer reverted the change at once. That engineer received two peer
bonuses and applause from thousands of colleagues.

---

## 2. The order of moves

Each row gives the signal that starts the move. Do not run a move before its signal.

| Signal | Move | Source |
|---|---|---|
| A metric crosses a threshold within a rollout window | Stop the rollout. Do not advance the next stage. | Ch. 17, p. 244 |
| Any of the three declaration questions answers yes | Declare an incident. Name one commander. | Ch. 14, p. 205–206 |
| Automation is making the change | Disable the automation before anything else | Ch. 13, p. 195 |
| The change removed alerts, dashboards, or probes | Revert the observability change first | Ch. 13, p. 196 |
| A named change sits in the fault window | Run the reverse action for that change | App. B, p. 572 |
| The reverse action needs more than a few minutes | Drain traffic away from the affected location | Ch. 13, p. 195 |
| Data corruption is still in progress | Freeze the system. Accept the outage. | Ch. 12, p. 173 |
| The service is stable again | Preserve the logs, then diagnose | Ch. 12, p. 173 |
| The incident is closed | Write the postmortem | Ch. 15, p. 209 |

**The three declaration questions.** Ch. 14, p. 205–206 asks three questions. Must you
involve a second team? Is the outage visible to customers? Does the problem persist after an
hour of focused work? One yes declares an incident.

**Declare early.** Ch. 14, p. 205 states that an early declaration with a simple fix costs
less than a framework started hours into a growing problem.

**One actor writes to production.** Ch. 14, p. 202–203 gives the operations role the only
write authority during an incident. Every other participant proposes to that role. See
hazard `R-40` in `rollback-hazard-catalog.md`.

---

## 3. The three emergencies of Chapter 13

`incident-response/references/02-emergency-response.md` holds the full account of these three
cases. This table keeps only the reversibility lesson of each.

| Case | Trigger | Blast radius | What reversed it | Reversibility lesson |
|---|---|---|---|---|
| Test-induced, p. 189–191 | A planned test blocked access to one database out of a hundred | Many dependent services. External and internal users. | An already-tested path restored permissions to the replicas and the failovers | A rollback procedure that no test exercised does not exist. p. 191. Hazards `R-02` and `R-04` |
| Change-induced, p. 191–194 | One configuration change pushed globally on a Friday | The whole externally facing fleet, plus internal applications | A second configuration change, five minutes later | The revert path must not run through the failing system. p. 193. Canary every change, whatever its perceived risk. p. 194. Hazards `R-05`, `R-17`, `R-19` |
| Process-induced, p. 194–197 | Two consecutive turndown requests to fleet automation | Every small server installation in the world | Nothing. The disks were wiped. | Some damage has no reverse action. Treat an empty target set as an error. p. 196. Revert the monitoring change first. p. 196. Hazards `R-34`, `R-35`, `R-45` |

**The one sentence to carry from the third case.** The turndown server received an empty
response for the machine rack. It passed that empty filter to the machine database. The book
writes "Yes, sometimes zero does mean all." SRE Ch. 13, p. 196. Read `07-automation-safety.md`
for the controls that stop it.

**The one sentence to carry from the second case.** Command-line tools and other access
methods performed the reversal while the normal interfaces were unreachable. SRE Ch. 13,
p. 193. Engineers must test those tools on a routine schedule.

---

## 4. Mitigate first. Diagnose second.

`production-troubleshooting` owns triage and the rest of the Chapter 12 method. This section
keeps only the three rules that decide when the reverse action runs.

**Triage precedes root-cause analysis.** Ch. 12, p. 173 tells the responder to make the
system work as well as the circumstances permit, before any search for a cause. The chapter
names three emergency options. Divert traffic from a broken cluster to working ones. Drop
traffic to prevent a cascading failure. Disable subsystems to reduce load.

**Fly the airplane.** Ch. 12, p. 173 borrows the pilot's rule. The first duty in an emergency
is to fly the airplane. Troubleshooting is secondary to a safe landing.

**Freeze the system when the damage is still growing.** Ch. 12, p. 173 gives one condition. A
defect causes data corruption that you may not recover. Then a freeze can be better than
continued operation. The freeze is the move that precedes the revert.

**Rapid triage does not permit destroying evidence.** Ch. 12, p. 173 requires both. Preserve
the logs while you mitigate. `05-locating-the-bad-change.md` needs that evidence later.

**Mitigation can stand as the end state for a period.** Ch. 12, p. 182–186 records an App
Engine case. SRE could not find the cause before the customer's public launch. The team moved
the application to a CPU-rich instance type. Latency reached an acceptable level, and the
team investigated afterward.

**A mitigation is itself a change.** Ch. 12, p. 179–180 warns that an active test alters
future results. Record every mitigation you apply, so that you can return the system to the
state before the test. Without that record you run an unknown mixed configuration.

---

## 5. Reverse the mitigation that made it worse

**A mitigation that makes the symptom worse gets reverted at once.** App. D, p. 582–583
records the sequence. At 15:32 an engineer changed load balancing to send more traffic to
the non-sacrificial clusters. At 15:33 tasks in those clusters started to fail with the same
symptoms. At 15:34 the team found an order-of-magnitude error in the whiteboard arithmetic.
At 15:36 the engineer reverted the load balancing change.

**Restore traffic in stages after a revert.** App. D, p. 583 records the ladder that the
team used. It moved through 1%, 10%, 30%, 50%, and 100% of traffic. The team checked the SLO
between each step, and confirmed that the HTTP 500 rate stayed inside the SLO with no task
failures. Never return to 100% in one step.

---

## 6. When to roll forward instead of reverting

The reverse action is the default. Four narrow conditions defeat it. Each one has a source.

| Condition | Why the revert fails | Do this instead |
|---|---|---|
| The revert crosses a destructive schema step | The old code reads data from the new version as corrupt or incomplete. Release It! Sec. 18.4, p. 333 | Roll forward. Restore from a backup only when the old shape is gone. |
| Nobody can rebuild the prior artifact | A non-hermetic build cannot reproduce the artifact. SRE Ch. 8, p. 122 | Roll forward with a corrective push. Record hazard `R-06`. |
| The fault is bad data, not bad code | A code revert leaves the data wrong. SRE App. D, p. 580 | Push the corrective data. A new index resolved the query of death. |
| No test ever exercised the reverse path | The reverse action can fail and lengthen the outage. SRE Ch. 13, p. 190–191 | Roll forward, and record the defect as a postmortem action. |

**Roll forward is a narrow exception, not a preference.** Each row above names a defect in
the reversibility of the change. Record that defect. `01-the-reversibility-contract.md` and
the hazard catalog hold the fix.

**The point of no return decides this table.** Release It! Sec. 18.4, p. 334 puts every
destructive step in the Cleanup phase. That phase drops columns, deletes old assets, converts
columns to NOT NULL, and adds referential integrity. Before Cleanup, a code revert is safe.
After Cleanup, it is not. Read `06-data-and-schema-reversibility.md`.

---

## 7. The capabilities the emergency needs beforehand

You cannot build these during the outage. Each row names the capability, the source, and the
hazard that its absence creates.

| Capability | Source | Hazard when absent |
|---|---|---|
| A rollback procedure that a test exercises | Ch. 13, p. 191 | `R-02` |
| Command-line tools that work when the interfaces do not | Ch. 13, p. 193 | `R-05` |
| Out-of-band communication that the outage cannot reach | Ch. 13, p. 193 | `R-05` |
| A canary for every change, whatever its perceived risk | Ch. 13, p. 194 | `R-17` |
| A rate limit on updates to new clients | Ch. 13, p. 193 | `R-35` |
| Sanity checks on every command that automation sends | Ch. 13, p. 196 | `R-34` |
| Observability that survives the change and the teardown | Ch. 13, p. 196 | `R-45` |
| A production log of every deploy and configuration change | Ch. 12, p. 177 | `R-41` |
| One named write holder during the incident | Ch. 14, p. 202–203 | `R-40` |

**Engineers must practice the out-of-band path.** Ch. 13, p. 193 records the caveat with the
praise. The tools worked, but the engineers needed more familiarity with them and more
routine testing.

**Alerting can defeat the response.** Ch. 13, p. 193–194 records alerts that fired repeatedly
across every location. They overwhelmed the on-call engineers and spammed the emergency
channel. Bound the alert volume before the incident. Ch. 16, p. 218–219 covers aggregation.

---

## 8. Luck is not a control

**The book names luck as a factor in the fast recovery.** Ch. 13, p. 193 states that the push
engineer happened to follow real-time communication channels. The book calls that an extra
level of diligence that the release process does not require. The engineer saw complaints
about corporate access right after the push and reverted almost at once.

**Convert that habit into a mechanism.** Without the habit, the outage could have lasted much
longer and become far harder to troubleshoot. A monitoring system, not the person who pushed,
must supervise each rollout stage. App. B, p. 572 states the requirement. See hazard `R-19`
and `03-progressive-rollout-and-canary.md`.

---

## 9. After the reverse action

**A rollback is a postmortem trigger.** Ch. 15, p. 209 lists on-call intervention, which
includes a release rollback and a traffic reroute, among the common triggers. Agree the
trigger list before an incident, not during one.

**Record a revert that did not fix the symptom.** Ch. 14, p. 204–205 records a responder who
reverted to the previous release with no result. Write the attempted action and its outcome
in the live incident document. The next responder must not repeat it. See hazard `R-44`.

**Return the system to the repository.** Any file that a responder edited on a host during
the incident must return to version control. Release It! Sec. 14.2, p. 245 requires an audit
that reports the difference. See hazard `R-24`.

**Keep the negative result.** Ch. 12, p. 180–182 states that a negative result is conclusive
and worth publishing. A reverted change is a negative result. It tells the next team what
production does under that change.

**Ask the improbable question.** Ch. 13, p. 198 asks four questions. What if the building
power fails? What if the equipment racks stand in two feet of water? What if the primary
datacenter goes dark? Could the person beside you do what you would do? Answer those
questions in the Reversal Record.

---

## 10. When this file does not apply

**No change in the window means the cause is not a change.** Ch. 12, p. 184 records the App
Engine team excluding change as the cause. The regression started on a Saturday with no
application push and no production push in flight. The most recent pushes had completed days
before.

**State the absence with the same confidence as the presence.** A change-attribution step
must be able to report that no change sits in the window. When no change sits there, leave
this file and troubleshoot the system. Ch. 12, p. 170–186 holds that method.

**Correlation is not causation.** `production-troubleshooting` owns this rule and the App
Engine case behind it. The reversibility consequence is one line. Design a test with mutually
exclusive outcomes before you reverse anything large. Ch. 12, p. 179. See hazard `R-42`.

**A small change needs none of this.** A one-file change with no persisted state, no deploy,
and no external effect needs no incident procedure. "Not applicable" is a valid result.
