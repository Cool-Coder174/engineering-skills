# Automation Safety: The Automation Is The Blast Radius

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 7, The Evolution of Automation at Google, p. 97–119. The case study section Automation:
Enabling Failure at Scale, p. 117–118. Supporting pages: Ch. 6, p. 91. Ch. 13, p. 193–197.
Ch. 14, p. 201–203. Ch. 17, p. 242 and p. 246. Ch. 26, p. 447.

Read this file when an automated actor performs the change or performs the revert. A deploy
pipeline, a release updater, a repair loop, a bulk job, and an AI coding agent are all
automated actors.

Every rule in this file applies to the revert path, not only to the change path. A revert
script is automation. It carries the same blast radius as the automation that it undoes.

---

## 1. The one claim

**Automation does not make a mistake more likely. It makes one mistake reach every target.**

SRE Ch. 7, p. 97–98 names consistency as the primary value of automation for a known,
well-scoped procedure. A human performs an action hundreds of times and does not perform it
the same way each time. That inconsistency produces mistakes and data quality defects.

The same property inverts. SRE Ch. 7, p. 98 states that a platform centralizes mistakes. A
defect that you fix in the code is fixed once and for every future run. A defect that you
write into the code runs against every target, at machine speed, before a person reads the
first alert.

**So the automation sets the blast radius, not the change.** A one-line logic defect in a
loop that touches one host is a small defect. The same defect in a loop that touches every
host in every location is a fleet-wide data-destruction event. SRE Ch. 7, p. 99 states the
limit that follows. Scope an automatic procedure over a well-defined domain, because
automation can make a bad situation worse faster than a human can.

**Corollary for this skill.** The reverse path is the path you run when the system is already
damaged. Automating it raises the cost of a defect in it, not only the speed of the cure.

---

## 2. The war stories

`incident-response/references/02-emergency-response.md` owns the human response in the
second and third cases below. This section keeps the automation defect and its control.

**Diskerase at the CDN. SRE Ch. 7, p. 117–118.** Google decommissions racks in third-party
colocation sites with automation. One step overwrites the full disk content of every machine
in the rack. Google calls that step Diskerase. An independent system then checks the erase.
On one occasion the decommission automation failed after Diskerase had already completed.
Engineers restarted the decommission from the beginning to debug the failure. On that second
run the set of machines that still needed Diskerase was correctly empty. The code treated the
empty set as a special value that meant everything. The automation sent almost every machine
in every colocation site to Diskerase. Within minutes the disks of the whole CDN were erased,
and those machines could no longer terminate user connections. Google's own datacenters
absorbed the traffic, and the only external effect was a small latency increase. Recovery
took the better part of two days of reinstallation, then weeks of auditing.

**The turndown emergency. SRE Ch. 13, p. 194–197.** During routine automation testing an
engineer submitted two consecutive turndown requests for the same installation. On the second
request the turndown server received an empty response for the machine rack. It did not filter
that empty response. It passed the empty filter to the machine database, which read the empty
filter as every machine. Machines in all such installations worldwide entered the Diskerase
queue. The on-call engineers drained traffic, then disabled all team automation to prevent
further damage. Traffic moved to large installations within an hour, and users saw higher
latency rather than errors. The stated root cause is that the turndown automation server
lacked sanity checks on the commands that it sent. The book records the lesson in six words:
"Yes, sometimes zero does mean all." (SRE Ch. 13, p. 196). The same automation also removed
the monitoring for those installations. The on-call engineers had to revert the monitoring
changes first, to measure the damage. The vast majority of capacity returned within three
days. Some machines took one or two months.

**The idempotent fix loop. SRE Ch. 7, p. 108–111.** A team extended the Python unit test
framework to test real production services, and named it Prodtest. Each test carried
dependencies, and a failure in one test aborted the chain. The team then paired each test
with a fix. Each fix had to be idempotent and had to assume that its dependencies were met.
Idempotence bought a concrete operational property. Teams could run the fix script every 15
minutes without fear of damage to the configuration. If a fix failed several times, the
automation stopped and notified a person. Cluster turnup time fell from six or more weeks to
one or two weeks. The book then states the honest defect of the design. The delay between the
test, the fix, and the second test produced flaky tests. A fix that was not truly idempotent,
run after a flaky test, could leave the system inconsistent (p. 111).

**Decider, and what automation buys. SRE Ch. 7, p. 104–106.** Manual MySQL master failover
took 30 to 90 minutes per instance and gave a best case of 99% uptime. The error budget
demanded less than 30 seconds of downtime per failover, and no human-driven procedure can
meet that number. Ads SRE built Decider in 2009. It completed planned and unplanned failovers
in less than 30 seconds 95% of the time. The cost was real. Every application needed more
failure-handling logic, because the database was no longer the most stable component in the
stack. The payoff was also real. Operational maintenance cost fell by nearly 95%, and about
60% of the hardware was freed. **Automation is worth building. This file is about the price
of the controls, not an argument against the purchase.**

**The turnup team. SRE Ch. 7, p. 111–113.** Service owners handed the execution of turnup
automation to a specialized team that worked from tickets. Latency fell at first. Then the
production systems continued to change, at over a thousand separate changes a day. The people
who felt the automation defects were no longer the domain experts. The automation became less
relevant, because new steps were missed. It became less competent, because new flags made it
fail. The book calls the end state the worst of all worlds. The remedy was to return ownership
to the service teams, behind a contract that each team published. **Automation that a separate
team maintains rots.** SRE Ch. 7, p. 103 names the mechanism as bit rot, which is the failure
of the automation to change when the underlying system changes.

---

## 3. The hierarchy of automation classes

SRE Ch. 7, p. 103 gives the path that automation follows. The book restates it at p. 114. The
example in the book is a database master failover.

| Class | Name | Example | Where the knowledge lives |
|---|---|---|---|
| 1 | No automation | A person fails the database master over between locations | In the operator's head |
| 2 | External, system-specific automation | An SRE keeps a failover script in a home directory | In one person's directory |
| 3 | External, generic automation | The SRE adds database support to a shared failover script | In a shared tool, outside the system |
| 4 | Internal, system-specific automation | The database ships its own failover script | Inside the system |
| 5 | No automation needed | The database detects the fault and fails over with no person | Inside the system, autonomously |

**Read the ladder as a statement about who owns the defect, not as a maturity score.**

- Class 2 is the class where most revert scripts live. The script is unreviewed, untested, and
  known to one person. It fails the audit-trail rule in Section 7.
- Class 3 is where a shared rollback tool lives. It is better, and it rots, because the tool
  changes on a different schedule from the systems that it touches. SRE Ch. 7, p. 103.
- Class 4 is the target for a reverse path. The system that makes the change also owns the
  action that undoes the change.
- Class 5 needs self-introspection to survive. SRE Ch. 7, p. 116 states that a system moving
  from manually triggered to automatically triggered to autonomous needs some capacity for
  self-introspection. The same page adds a requirement that this skill depends on. Expose the
  internal details that the introspection uses to the humans who manage the system.

**Rule.** Do not move a reverse path from class 2 to class 5 in one step. SRE Ch. 7, p. 117
states that autonomous operation is hard to retrofit to a large system. The same page states
that the design phase gives the largest return.

---

## 4. The three axes that measure automation

SRE Ch. 7, p. 112 gives three properties. Use them to argue about a revert tool without
adjectives.

| Axis | Question | Failure that it detects |
|---|---|---|
| Competence | How accurate is the automation? | The tool reverts the wrong artifact, or leaves a partial state |
| Latency | How fast does the automation complete every step once a person starts it? | The revert takes longer than the outage budget |
| Relevance | What proportion of the real process does the automation cover? | The tool reverts the code and leaves the schema and the flag |

**Relevance is the axis that reviews miss.** A revert tool that covers the artifact and not the
configuration is not a revert tool. SRE Ch. 6, p. 83 defines a push as any change to the
running software of a service **or its configuration**. Read
`01-the-reversibility-contract.md` for the class that a change inherits.

---

## 5. Idempotence is the entry requirement

**An automated step that is not idempotent must not run twice. Every automated step runs
twice.** Four events produce a second run. A retry. A restart after a partial failure. A
duplicate request. A debugging re-run.

The Diskerase incident is the exact shape of this failure. The workflow failed part of the way
through. A person restarted it from the beginning. The second run reinterpreted state that the
first run had already changed. SRE Ch. 7, p. 118 lists making the decommission workflow
idempotent as one of the three remedies that Google adopted.

SRE Ch. 7, p. 110 gives the operational meaning of the requirement. A fix that is idempotent
can run every 15 minutes without damage to the configuration. That number is Google's number
for that loop. Your interval is a local decision.

**The idempotence test for a reverse step.** Answer all four questions with yes before the step
may run without a person.

1. Does the step produce the same end state after two runs as after one run?
2. Does the step read the current state before it acts, rather than assuming a start state?
3. Does the step treat an already-correct target as success, not as an error?
4. Does the step name the target explicitly, rather than by the absence of a marker?

Question 4 covers R-37. Automation must not read a missing signal as permission to act. SRE
Ch. 7, p. 107 records that free-form scripts could not answer whether every configuration
exception was desired. Require an explicit positive signal before any destructive action.

For the retry and idempotency contract in code, read
`data-systems-design/references/07-transactions.md` and the `implement` skill. This file states
the requirement. It does not restate the mechanism.

---

## 6. A rarely used path is a broken path

SRE Ch. 7, p. 103 states the rule directly. Automation that is crucial, that runs at infrequent
intervals, and that is therefore hard to test, is particularly fragile, because the feedback
cycle is long. The book names cluster failover as the classic example, because a failover may
occur only every few months.

SRE Ch. 6, p. 91 gives the matching rule for monitoring. Collection, aggregation, and alerting
configuration that runs less than once a quarter is a candidate for removal.

**Therefore: make the automation the only path.**

| Situation | What happens to the path | Required action |
|---|---|---|
| The revert path runs only during an incident | It rots, and it fails on the day you need it | Make the routine deploy use the same code path |
| The restore path runs only in a drill | It stays correct only while the drill runs | Schedule the drill, and record the date in the Reversal Record |
| An emergency path exists beside the normal path | The emergency path is untested code | Delete it, or exercise it on the routine schedule |
| A person can perform the step by hand | The manual path is the untested one | Route the manual case through the same automation |

Two consequences follow, and they pull against each other.

**Consequence 1. Exercise the reverse path on a schedule.** This covers R-04. An untested
rollback procedure is R-02. SRE Ch. 13, p. 190–191 records a permissions rollback at Google.
The engineers attempted it, it failed, and the outage grew longer.

**Consequence 2. Keep a path that does not need the automation.** SRE Ch. 13, p. 193 states
that command-line tools and other access methods must work when the normal interfaces do not.
The same page states that teams must test them routinely. The revert path must not run through the system that is
down. That is R-05. Read `04-change-induced-emergency.md`.

---

## 7. The four controls on every automated actor

The three remedies that Google adopted after Diskerase were sanity checks, rate limiting, and an
idempotent workflow. SRE Ch. 7, p. 118. The turndown emergency added a fourth control, because
the on-call engineers disabled all team automation to stop the damage. SRE Ch. 13, p. 195.

| Control | What it does | Signal that it is missing | Source |
|---|---|---|---|
| Sanity check | Rejects a degenerate or implausible target set before the action runs | The code has no lower bound on the target count, and no test for an empty filter | SRE Ch. 7, p. 118. SRE Ch. 13, p. 196 |
| Rate limit | Caps hosts, rows, or actions per unit of time | A loop over every result of a query, with no cap and no pause | SRE Ch. 7, p. 118 |
| Canary | Applies the action to a small, measured fraction first | The action reaches 100% of targets in the first run | SRE Ch. 13, p. 194 |
| Stop control | Lets a person halt the actor at once, from outside it | No documented way to disable the automation during an incident | SRE Ch. 13, p. 195 |

### Sanity check

**Treat an empty or degenerate target set as an error, never as every target.** This is R-34.
The rule generalizes past the empty set. Any selector that expands beyond a declared bound is a
defect, not a large job. Require the caller to state the expected count, and stop when the actual
count differs.

### Rate limit

**A rate limit converts a fleet-wide event into a bounded one and buys a person time to react.**
This is R-35. The limit must live inside the automation, not only in the interface that a person
uses. SRE Ch. 26, p. 434 lists rate-limiting features among the requirements for out-of-band data
validation, alongside monitoring, alerts, dashboards, and playbooks. The book does not publish a
universal rate number. **Set the number so that a person can read an alert and act in time.** The person must act
before the job passes the point of no return. That number is a local decision.

### Canary

**Canary every change, whatever its perceived risk.** SRE Ch. 13, p. 194. This applies to a
revert. A revert is a push, and a push can be wrong. SRE Ch. 17, p. 246, fn. 89 gives Google's
ladder. Start at 0.1% of user traffic. Then scale by one order of magnitude every 24 hours. Vary the
geography of the servers that you upgrade. Read
`03-progressive-rollout-and-canary.md` for the stages and the gating metric.

### Stop control

**Every automated actor needs a control that a person can use to stop it from outside the actor.**
Teams commonly call this a kill switch. The control has four requirements.

1. A person can reach the control when the target system is down. SRE Ch. 13, p. 193.
2. The control stops the actor without a code change and without a deploy.
3. The control is documented in the runbook, with the name of the actor.
4. Stopping the actor leaves a state that a person can inspect, not a partial write.

SRE Ch. 7, p. 110 gives the automatic version of the same control. If a fix fails several times,
the automation assumes that the fix failed, stops, and notifies a person. That covers R-39.

---

## 8. Report the defect. Do not repair it by deleting data

SRE Ch. 17, p. 242 gives the rule for a tool that detects a discrepancy. The tool should report
the problem before it becomes an outage, and it should avoid repairing the problem on its own by
deleting user data. The same page names the mechanism as defense in depth. Each anticipated
misbehavior should be reported by some test, or by the input validator of another tool.

**The value of a detector is the report.** This is R-38.

| The automation finds | It may do this without a person | It may not do this |
|---|---|---|
| A configuration drift | Write the correct configuration and re-check | Delete the configuration and rebuild from a default |
| A missing index | Create the index | Truncate the table |
| An orphaned row | Report the row and the count | Delete the row |
| A stale replica | Stop serving from the replica | Erase the replica and re-clone during an incident |
| A failed rollout stage | Return to the last known good state | Apply a corrective data migration |

The right column is not a style preference. SRE Ch. 26, p. 438–444 records the Google Music
incident. A redesigned deletion pipeline wrongly removed about 600,000 audio references and
affected 21,000 users. Recovery needed more than 5,000 backup tapes and took just short of
seven days. Read `06-data-and-schema-reversibility.md` for the restore ladder.

**The detector that this section requires.** SRE Ch. 26, p. 447, fn. 133 gives the concrete alert.
Alert when the global deletion rate, aggregated across all users, crosses an extreme threshold,
such as 10 times the observed 95th percentile. Do not alert on the per-user rate, because that
rate varies naturally. This covers R-33.

---

## 9. Automation removes the human ability to operate

SRE Ch. 7, p. 116–117 states the second-order cost. Automation progressively removes the operator
from direct contact with the system. Then the automation fails, and the operators cannot run the
system. Their reactions have lost fluidity through lack of practice, and their mental models no
longer match the system. The book cites analogies in aviation and in industry, including Air
France Flight 447.

The page adds the worse case. The manual actions are not always performable, because the
functionality that permitted them no longer exists.

**Three consequences for a reverse path.**

1. Keep a manual reverse path that a person can run, and test it. SRE Ch. 13, p. 193.
2. Practice the reverse path on a schedule. SRE Ch. 7, p. 119, fn. 35 names regular practice
   drills and Disaster Role Playing.
3. Expose what the automation observed and what it did, to the people who manage the system.
   SRE Ch. 7, p. 116.

---

## 10. The AI coding agent as an automated actor (**Modern**)

An AI coding agent that reverts on its own is a class 2 or class 3 actor from Section 3. It has
the latency of a class 5 actor. It acts faster than a person can read the plan. Every control
in Sections 5 through 8 applies to it without change.

The books predate LLM coding agents. The mapping below is **Modern**. Each rule beside it carries
its page.

| Agent action | Permitted without a person | Required control | Rule and source |
|---|---|---|---|
| Commit a checkpoint | Yes | None | A checkpoint adds state. It removes none. |
| Revert its own unpublished branch | Yes | The branch has one holder | Two actors must not write at once. SRE Ch. 14, p. 201–203 |
| Open a revert request for review | Yes | The request names the reverse path | SRE Ch. 27, p. 461 |
| Push to a shared branch | No | A named person approves | SRE Ch. 14, p. 202–203 |
| Deploy or move a release label | No | A person approves, and a canary gates the stage | SRE Ch. 13, p. 194 |
| Run a bulk edit over many files or rows | No | Sanity check, rate limit, and stop control | SRE Ch. 7, p. 118 |
| Run a destructive migration or a data restore | No | A named human approver | SRE Ch. 17, p. 242 |
| Act during a declared incident | No | One actor holds the write | SRE Ch. 14, p. 201–203 |

**Rule A. One actor writes to production during an incident.** SRE Ch. 14, p. 201–202 records
the freelancing failure. An engineer deployed an uncoordinated change during an outage with good
intentions, and the change made a bad situation far worse. An agent that deploys while a human
operator also deploys reproduces that failure exactly. This covers R-40.

**Rule B. Every automated action leaves an audit record.** SRE Ch. 7, p. 113 describes the Admin
Server that replaced free-form SSH. It logs the requestor, the parameters, and the results of
every remote call, for debugging and for security audits. No one could install or modify a server
without an audit trail. Apply the same requirement to an agent. Record the actor, the command, the
target set, the count, and the result.

**Rule C. Give the agent the minimum privilege that the task needs.** SRE Ch. 7, p. 113 records
that Google reduced SRE privileges to the absolute minimum and gated changes to the automation
substrate on code review. `security-engineering` owns the mechanism. This file states the
requirement.

**Rule D. Do not wire an automatic revert to a flaky signal.** SRE Ch. 17, p. 236–237 requires
you to measure the flake rate of a test before you trust it. The same pages require you to hold
the signal to the accuracy that the volume demands. An automatic revert on a flaky test reverts
good changes and thrashes the fleet. This covers R-43. Read `05-locating-the-bad-change.md`.

**Rule E. The agent must not remove the monitoring that shows its own damage.** SRE Ch. 13,
p. 196 records that the turndown automation removed the monitoring for the affected
installations. The on-call engineers reverted those monitoring changes first. This covers
R-45.

---

## 11. The review pass for this file

Run these checks against any change that adds or edits an automated actor. Report only the checks
that fail.

1. Does the actor have a declared bound on its target set? If no, report R-34.
2. Does the actor have a rate limit inside its own code? If no, report R-35.
3. Is every step idempotent under the four questions in Section 5? If no, report R-36.
4. Does any step infer permission from an absent signal? If yes, report R-37.
5. Does any automatic remediation delete or rewrite user data? If yes, report R-38.
6. Does the actor stop and notify a person after repeated failure? If no, report R-39.
7. Can two actors write to production at the same time? If yes, report R-40.
8. Does a person have a stop control that works when the target system is down? If no, report R-05.
9. Does the reverse path run on a schedule, or only during an incident? If only during an
   incident, report R-04.
10. Does the actor remove monitoring, alerts, or dashboards? If yes, report R-45.

**"Not applicable" is a valid result.** A change that adds no automated actor, and that edits no
existing one, needs nothing from this file.

---

## 12. Cross-references

| Read this next | When |
|---|---|
| `rollback-hazard-catalog.md`, group G | You run the hazard scan for an automated actor |
| `01-the-reversibility-contract.md` | You classify the change and name its reverse path |
| `03-progressive-rollout-and-canary.md` | You set the canary stages and the gating metric |
| `04-change-induced-emergency.md` | The automation is causing damage right now |
| `05-locating-the-bad-change.md` | You wire a revert trigger to a signal |
| `06-data-and-schema-reversibility.md` | The automated action touches data or schema |
| `08-agent-and-operator-shortcuts.md` | You need the exact checkpoint or revert command |
| `security-engineering` | You set the privileges and the audit records for the actor |
