# Make It Work First, Understand It Second

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 12 "Effective Troubleshooting", the Triage section, p. 172–174 and p. 185. Ch. 13
"Emergency Response", p. 189–197. The mitigation menu comes from Ch. 22 "Addressing
Cascading Failures", p. 339–341. Four moves come from *Release It!* — Michael T. Nygard
(Pragmatic Bookshelf, 2007), Sec. 5.2, p. 115, Sec. 7.6, p. 158–160, and Sec. 16.7–16.11.

Triage means that you stop the bleeding before you find the wound. This file gives the rule,
the order of work, the moves you can make, and the evidence that each move destroys. Triage
is step 2 of the model in `01-the-troubleshooting-model.md`. Read
`03-examine-and-diagnose.md` after the system serves again.

---

## 1. The rule

**Stop the bleeding first.** Your first response in a major outage can be a search for the
root cause. Ignore that instinct. Make the system work as well as it can under the circumstances.
A correct diagnosis does not help a user whose system already died (`SRE Ch. 12, p. 173`).

**Fly the airplane.** A pilot's first duty in an emergency is to fly the airplane.
Troubleshooting is secondary to a safe landing. The same order holds for a computer system
(`SRE Ch. 12, p. 173`).

**Freeze a fault that corrupts data.** When a defect can write data that you cannot recover,
freeze the system. A frozen system is better than a system that corrupts more records
(`SRE Ch. 12, p. 173`).

**Rapid triage does not remove the duty to preserve evidence.** The book states both duties
in the same paragraph. Capture the logs and the state that root-cause analysis will need
(`SRE Ch. 12, p. 173`).

**The fix for the proximate cause does not wait for the root cause.** You may repair the
symptom now and find the origin later (`SRE Ch. 12, p. 171`).

**This order is counterintuitive.** The book states that engineers from product development
find it unsettling (`SRE Ch. 12, p. 173–174`).

---

## 2. The signal that starts triage

Triage starts when the problem report states a user impact. It does not wait for a cause.

Size the impact before you choose a move. Read **the four golden signals** on the affected
service. They are latency, traffic, errors, and saturation (`SRE Ch. 6, p. 88`).

Answer four questions. Each answer is one line in the Fault Record.

1. Who is affected, and how many?
2. Which user journey fails?
3. Does a workaround exist?
4. Does the fault write wrong data, or destroy data?

A "yes" to question 4 promotes the fault to the highest class, whatever the user count says.

---

## 3. Table 1 — Impact to response

The response must be proportionate to the impact. An all-hands emergency is correct for a
global outage. The same response for one user with a workaround is overkill
(`SRE Ch. 12, p. 173`).

`incident-response` owns the declaration gate, the response classes, and the four roles. The
table below is the short form for a lone responder. Read
`incident-response/references/01-incident-command.md` when the fault needs more than
one person.

| Impact | Who responds | Page a person? | Protocol |
|---|---|---|---|
| One user. A workaround exists. | One engineer, in business hours | No | Ticket only |
| One feature is degraded for many users | The on-call engineer | Yes | Incident ticket |
| A user-facing service is down in one region | The on-call engineer and the service owner | Yes | Incident ticket, plus a status update |
| A global outage | The formal incident protocol. Call more people. | Yes | Formal incident management |
| Data loss or data corruption | The formal incident protocol. Freeze first. | Yes | Formal incident management |

**The escalation rule.** Adopt the formal incident protocol in two cases. The fault crosses
teams. Or you cannot estimate an upper bound for its duration (`SRE Ch. 11, p. 165`).

**The staffing rule.** Call more people when you feel overwhelmed. It can be correct to page
the whole company (`SRE Ch. 13, p. 189`).

**The state rule.** The person whose action triggered the event holds the most state. Use
that person (`SRE Ch. 13, p. 197`).

**The stress rule.** Stress promotes habit over deliberate thought. A habitual response is
unconsidered, so it can be disastrous (`SRE Ch. 11, p. 164–165`).

---

## 4. The evidence rule

Every mitigation is a change to the system. Some changes erase the state that names the
cause. Capture that state first.

**Order the runbook so that the capture step sits above the destructive step.** A runbook
that orders a restart before a thread dump loses the blocked stacks forever
(`SRE Ch. 12, p. 173`. `Release It! Sec. 16.7, p. 259`).

**Capture is cheap. State is not.** A thread dump takes seconds. The process that held the
stacks is gone after the restart.

**Record the capture in the Fault Record.** Write the time, the move, the effect, and the
evidence that you captured before it. Section 3 of the record holds this table.

**Take clear notes during the incident, not after it.** A shared document gives the
timestamps that the postmortem needs. It also informs other people without an interruption
to the responder (`SRE Ch. 12, p. 180`).

---

## 5. Table 2 — What each move destroys, and what to capture first

| Signal | Move | Evidence the move destroys | Capture this first |
|---|---|---|---|
| One cluster is unhealthy. Others are healthy. | Send the traffic to the healthy clusters (`SRE Ch. 12, p. 173`) | The live state of the unhealthy cluster | Thread dump, queue depth, memory use |
| Every replica crash-loops | Admit about 1% of the traffic. Raise it slowly. (`SRE Ch. 22, p. 340`) | Nothing | Crash log, restart count, start time |
| The fault started at a deploy | Reverse the change (`SRE Ch. 22, p. 335`) | The running version and its configuration | Version id, configuration snapshot, deploy time |
| One dependency is saturated | Throttle the pool for that dependency (`Release It! Sec. 16.9, p. 262`) | The blocked-thread evidence | Thread dump on the caller and on the callee |
| Threads are busy and processor use is low | Take a thread dump. Then restart the component. (`Release It! Sec. 16.7, p. 259`) | The blocked stacks | Thread dumps from several nodes |
| Batch work runs on the serving path | Stop the batch work (`SRE Ch. 22, p. 341`) | Nothing | Job names and start times |
| One request shape crashes the process | Block that request shape (`SRE Ch. 22, p. 341`) | The crashing request | A copy of the request |
| New sessions exceed the capacity | Throttle new sessions at the edge (`Release It! Sec. 7.6, p. 158`) | The demand profile | Session count over time, source addresses |
| The fault writes wrong data | Freeze the system (`SRE Ch. 12, p. 173`) | Throughput evidence | The wrong records and their identifiers |
| Automation causes the damage | Disable all team automation (`SRE Ch. 13, p. 195`) | The automation's next action | The command it sent and its target set |

**The restart caution.** Name the source of the fault before you restart a server. Confirm
that the restart does not only move the load. A restart amplifies a cold-cache fault
(`SRE Ch. 22, p. 340`).

**The single-node rule.** Apply the move to one node. Confirm the effect. Then apply it to
the fleet (`Release It! Sec. 16.10, p. 262–263`).

**The single-node caution.** One repaired node behind a load balancer receives all the
traffic. It can fail for that reason alone. Read the verdict from the fleet, not from the
one node (`Release It! Sec. 16.10, p. 263`).

---

## 6. Table 3 — The mitigation menu

Ch. 22 names seven immediate steps for a cascading failure. Ch. 12 names three emergency
options at the triage step. This table joins them. Each row gives the signal.

| Signal | Move | Constraint | Source |
|---|---|---|---|
| Idle capacity exists and the service is not in a death spiral | Add tasks | It is not enough for a death spiral | `SRE Ch. 22, p. 339` |
| The scheduler kills tasks while they start | Disable the health checks for a time | Separate a process check from a service check | `SRE Ch. 22, p. 339` |
| A garbage-collection death spiral, a deadlock, or requests with no deadline | Restart the servers | Name the source first. Canary the restart. Run it slowly. | `SRE Ch. 22, p. 340` |
| One cluster is broken and other clusters serve | Divert the traffic to the working clusters | The target must hold the extra load | `SRE Ch. 12, p. 173` |
| A location cannot answer at all | Drain the traffic away from that location | Draining avoids an outright request failure | `SRE Ch. 13, p. 195` |
| A true cascading failure that no other move stops | Drop the traffic | The big hammer. Use the four-step ramp below. | `SRE Ch. 22, p. 340` |
| The service can serve a smaller answer | Enter a degraded mode | You must engineer this mode before the incident | `SRE Ch. 22, p. 341` |
| Batch work shares the serving path | Stop the index updates, the copies, and the statistics jobs | None | `SRE Ch. 22, p. 341` |
| One request shape crashes the process | Block that request shape | Keep a copy of the request first | `SRE Ch. 22, p. 341` |
| One subsystem consumes the capacity | Disable that subsystem | The caller must tolerate its absence | `SRE Ch. 12, p. 173` |
| One dependency is slow or refuses connections | Open the **Circuit Breaker** for it | The Circuit Breaker relieves the pressure on that dependency | `Release It! Sec. 5.2, p. 115`. `Release It! Sec. 4.8, p. 98` |
| New sessions arrive faster than the capacity | Throttle new sessions at the edge | Watch the session count. Reduce the throttle before saturation. | `Release It! Sec. 7.6, p. 158` |
| Automation causes the damage | Disable all team automation | Then freeze all further automation and maintenance | `SRE Ch. 13, p. 195` |
| The fault writes data that you cannot recover | Freeze the system | This is the one case where a stopped system is the goal | `SRE Ch. 12, p. 173` |

**The four-step ramp for dropped traffic** (`SRE Ch. 22, p. 340`).

1. Address the condition that started the fault. Add capacity where you can.
2. Reduce the load until the crashes stop. Be aggressive. Admit about 1% of the traffic.
3. Wait until most servers report healthy.
4. Raise the load slowly, so that caches warm and connections establish.

Drop the least important traffic first. Repair or hide the origin before you raise the load
again, or the cascade returns with the traffic (`SRE Ch. 22, p. 340`).

**A note on the throttle.** A session throttle needs a person who watches the session count.
Recovery from full saturation took nearly an hour in the Nygard case, so the lever must move
before the threshold (`Release It! Sec. 7.6, p. 158`).

---

## 7. Triage in seven steps

1. State the impact in one sentence. Use the four questions in Section 2.
2. Choose the response size from Table 1. Escalate when the fault crosses teams.
3. Choose the move from Table 3. Read the signal, not the habit.
4. Read the evidence column in Table 2 for that move.
5. Capture that evidence. Write what you captured, and where you stored it.
6. Apply the move to one node. Confirm the effect. Then apply it to the fleet.
7. Write the time, the move, and the effect in Section 3 of the Fault Record.

Stop here. Section 4 of the record is the change window. Section 5 is Examine. Do not start
Examine while the error rate stays at its peak.

---

## 8. When the mitigation destroys the evidence

Some moves and some captures conflict. The book gives the priority, not a procedure. Use
this table to choose.

| Situation | Decision | Reason |
|---|---|---|
| The capture takes seconds and the impact is bounded | Capture, then mitigate | Rapid triage does not remove the duty to preserve evidence (`SRE Ch. 12, p. 173`) |
| The capture takes minutes and the fault corrupts data | Freeze first. Capture the frozen state. | The corrupt record count grows for the length of the delay (`SRE Ch. 12, p. 173`) |
| The capture needs a tool that the fault removed | Mitigate. Record the gap in Fault Record Section 10. | The tools sit behind the fault. See trap D-34. |
| The mitigation is a restart and you hold no dump | Take the thread dump. Then restart. | The blocked stacks exist only in the running process (`Release It! Sec. 16.7, p. 259`) |
| The mitigation removes the monitoring for the target | Reverse the monitoring change first | One team had to reverse the monitoring changes before it could measure the damage (`SRE Ch. 13, p. 196`) |

**When you cannot capture, say so.** Write the sentence "we could not capture X because Y"
in Section 10 of the Fault Record. That sentence becomes an observability requirement for
`detail-planning`. It is a negative result, and `05-negative-results-and-bias.md` explains
why you publish it.

**A local decision, not a book rule.** Some teams remove one faulty instance from the load
balancer and keep it running, so that a responder can examine it after the fleet recovers. Neither book states
this practice. Decide it locally, and write the decision in the runbook.

---

## 9. Redirect, reverse, drain

Three moves change where the traffic goes or which code runs. They are the fastest moves,
and they carry the most risk to the evidence.

**Redirect the traffic.** Send the traffic from a broken cluster to clusters that still work
(`SRE Ch. 12, p. 173`). The evidence at risk is the live state of the broken cluster.
Capture a thread dump, the queue depth, and the memory use first. Confirm that the target
cluster holds the extra load. A target that cannot hold it widens the outage
(`SRE Ch. 22, p. 314–316`).

**Reverse the change.** Another configuration change reverses a configuration change that
broke the fleet. One team pushed the reverse change within five minutes of the first push,
and the services started to recover (`SRE Ch. 13, p. 192`). Capture the running version
identifier and a configuration snapshot first. A rollback path that nobody tested fails at
the moment you need it (`SRE Ch. 13, p. 190–191`). Keep command-line tools and alternative
access that work when the normal interfaces do not (`SRE Ch. 13, p. 193`).

**This skill does not own the reverse.** Produce the change window in Fault Record Section 4
and the decision to reverse. `reverse-branching` owns the revert mechanism, the canary, and
the safety checks.

**Drain a region.** A location whose machines cannot answer should not receive requests.
Draining sends the traffic to locations that can respond, and it avoids an outright failure
for those requests (`SRE Ch. 13, p. 195`). Users see higher latency, and the requests
succeed. The fallback location must hold a full load by design. Watch the network links,
because one drain exposed congestion that engineers mitigated separately
(`SRE Ch. 13, p. 195–196`).

---

## 10. A mitigation can replace a cure, for a time

The book records a case where mitigation replaced diagnosis. SRE could not find the cause of
an App Engine latency fault before the customer's public launch. The team moved the
application to the most processor-rich instance type. Latency reached an acceptable level,
not a preferred one. The team then investigated at leisure (`SRE Ch. 12, p. 185`).

The book states the general position. It is often impractical to remove all known defects, so
a team accepts a second-best measure and reduces the risk (`SRE Ch. 12, p. 188, fn. 75`).

**Give every temporary mitigation an expiry date.** A temporary fix with no expiry date
becomes permanent architecture (`Release It! Sec. 7.6, p. 160`). Fault Record Section 9
holds the expiry date and the owner.

---

## 11. War stories

**The test-induced emergency, and the rollback that failed** (`SRE Ch. 13, p. 189–191`).
SRE planned to expose hidden dependencies. The test blocked all access to one database out
of a hundred. Within minutes many dependent services reported that external and internal
users could not reach key systems. SRE aborted the test and attempted a rollback of the
permissions change. The rollback failed, because nobody had tested the procedure in a test
environment, and the outage grew longer. The team then restored permissions to the replicas
and the failovers with an approach that it had already tested. Full access returned within
an hour. Two lessons followed. Test the rollback procedure before a large-scale test.
Communicate a new incident procedure before you rely on it. The responders did not follow
the procedure that the company had introduced a few weeks earlier.

**The change-induced emergency, and the five-minute reverse** (`SRE Ch. 13, p. 191–194`).
A team pushed a configuration change to the abuse-protection infrastructure globally, on a
Friday. That infrastructure interacts with almost every externally facing system, so the
whole fleet started to crash-loop at nearly the same time. Internal applications also failed, because
the internal infrastructure depends on the company's own services. Monitoring alerted within
seconds. Some responders believed the corporate network had failed and moved to panic rooms
that hold backup access to production. Within five minutes the push engineer sent another
configuration change that reversed the first one, and the services started to recover.
Responders declared an incident within 10 minutes. Some services did not recover fully for
up to an hour. The book names luck as a factor. The push engineer happened to follow the
real-time chat channels and saw the complaints right after the push. Two defects surfaced.
Alerts fired constantly and overwhelmed the responders. And much of the troubleshooting and
communication stack sat behind the crash-looping jobs.

**The process-induced emergency, and the drain** (`SRE Ch. 13, p. 194–197`). During routine
automation testing, a team submitted two consecutive turndown requests for the same
installation. On the second request a defect sent the machines in all such installations
worldwide to the queue that wipes hard drives. The on-call engineers paged as the first
installation went offline. They followed normal procedure and drained the traffic from the
location. The wiped machines could not respond, and a drain avoids an outright request
failure. As pagers fired worldwide, they disabled all team automation. They then froze
further automation and production maintenance. Within an hour all traffic ran through large
installations, which by design hold a full load. Users saw higher latency and their requests
succeeded. The root cause was the absence of sanity checks on the commands that the turndown
server sent. It received an empty response for the machine rack and did not filter it. It
passed the empty filter to the machine database, which read it as "all". The automation also
removed the monitoring, so the engineers had to reverse those monitoring changes before they
could measure the damage.

**The App Engine mitigation that preceded the diagnosis** (`SRE Ch. 12, p. 182–186`). An
internal customer reported latency up by nearly an order of magnitude, with processor time
and serving-process count up by nearly four times. No code change and no traffic rise
explained it. The App Engine developers found a correlation with a rise in `merge_join`
datastore calls. That call usually indicates poor indexing, so the team started to build
composite indices. Request tracing then showed that static content, which never touches the
datastore, was also slow. That result exposed the correlation as a coincidence and the index
theory as fatally flawed. The team could not solve the fault before the customer's launch,
so it mitigated instead and moved the application to the most processor-rich instance type.
The cause appeared later. An access-control defect created a whitelist object on every
access to one path. A security scanner produced thousands of them in half an hour. Every
request then scanned all of them in memory, and no remote call explained the time.

---

## 12. What triage does not do

Triage does not name the cause. It gives you the time in which you find the cause.

**A mitigation is not a diagnosis.** "We restarted it and the fault stopped" is not a result.
Write what you captured, and the cause that the restart is consistent with. See
`../SKILL.md`, the language table.

**An amplifier is not the origin.** A retry rate and a restart count grow during a fault.
They rarely start it. State which term you removed (`SRE Ch. 22, p. 319, p. 327`).

---

## 13. Proportionality

This file applies when a fault has a user impact now. It does not apply to a design review or
to a change that no user can see.

| Fault class | What this file requires |
|---|---|
| A question about a log line. No user impact. | Nothing |
| One user. A workaround exists. | Section 2 only. No mitigation move. |
| One degraded feature | Sections 2, 4, 5, and 7 |
| A user-facing outage | The whole file, plus the trap scan in Section 14 |
| Data loss or corruption | Freeze first. Then the whole file. Preserve the evidence. |

"Not applicable" is a valid result.

---

## 14. Traps that this step produces

Scan these codes in `diagnostic-trap-catalog.md` after any incident with a mitigation.
Report only the traps that apply.

| Code | Short name | The question it answers |
|---|---|---|
| D-01 | Diagnosis before mitigation | Did the first mitigation come after five investigation actions? |
| D-02 | A mitigation that erases the evidence | Did a restart or a flush run before any capture step? |
| D-03 | A restart before the source is named | Did a restart repeat more than twice with no localization between? |
| D-04 | A fleet-wide remedy with no single-node test | Did the remediation command carry an "all hosts" flag on its first run? |
| D-05 | A corrupting fault left running | Did a fault that writes wrong values continue during the investigation? |
| D-25 | A remedy that removes the monitoring | Did the turndown or drain procedure disable the monitoring? |
| D-34 | The diagnosis tools sit behind the fault | Did the dashboard or the chat system depend on the failing service? |
| D-35 | A rollback path that was never tested | Did anyone rehearse the rollback before the change that needed it? |
| D-36 | Remediation that needs a console | Does a script exist for the remediation, or only a graphical console? |
| D-37 | Restart is available only for the whole server | Can an operator restart one component instead of the host? |

---

## 15. Where to read next

| You need | Read |
|---|---|
| The six steps and the loop | `01-the-troubleshooting-model.md` |
| The telemetry that answers each question | `03-examine-and-diagnose.md` |
| A test that separates two hypotheses | `04-test-and-treat.md` |
| The bias that a repeat alert creates | `05-negative-results-and-bias.md` |
| The design that makes the next fault easier | `06-making-troubleshooting-easier.md` |
| The full trap list | `diagnostic-trap-catalog.md` |
| The revert mechanism and its safety checks | `reverse-branching/references/04-change-induced-emergency.md` |
| The roles and the protocol during an incident | `incident-response/references/02-emergency-response.md` |
| The observability specification | `detail-planning/SKILL.md` |
