# What an Automated Remediation May and May Not Do

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 7 "The Evolution of Automation at Google", pp. 97–119, and Ch. 13 "Emergency Response",
pp. 189–199. *Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007), Sec. 4.6
"Attacks of Self-Denial", pp. 88–90. Supporting pages come from SRE Ch. 6, Ch. 12, Ch. 21,
Ch. 22 and Ch. 24, and from Release It! Ch. 2, Ch. 5, Ch. 16 and Ch. 17.

This file gives the safety envelope for self-healing. Read it before you let a system act on
a vendor fault without a human. Read `03-stability-patterns.md` first for the defenses
themselves. Read `04-fault-localization.md` first for the evidence that names the fault.

**`reverse-branching` owns the undo path.** Read
`reverse-branching/references/07-automation-safety.md` for checkpoints, revert commands, and
the reversibility contract. `incident-response` owns the roles and the record once a person is
paged. This file states which self-healing action a vendor fault may trigger, and the limits
on that action.

---

## 1. The two facts that set the envelope

**Automation is a force multiplier, not a panacea.** (SRE Ch. 7, p. 97.) It repeats a correct
action at machine speed. It repeats a defect at the same speed. The value comes from the
action and from the judgment that applies it.

**An automatic procedure can make a bad situation worse.** SRE states this and gives the
remedy in the same sentence. Scope every automatic procedure over a well-defined domain.
(SRE Ch. 7, p. 99.)

These two facts produce the rule that orders this file. **A remediation is a change to
production. Give it the controls you give any other change.** Google's Diskerase automation
wiped the disks of almost every machine in every colocation site within minutes.
(SRE Ch. 7, pp. 117–118.) The action was correct. The target set was wrong.

**Restore service first. Preserve the evidence as you do it.** (SRE Ch. 12, p. 173.) A
remediation stops the bleeding. It does not find the root cause. It must not destroy the data
that a human needs later.

---

## 2. Decide whether to automate this response at all

Read one row. The left column is the signal you already have.

| Signal you observe | What it means | What to do |
|---|---|---|
| The page has a rote response. The responder runs the same three commands each time. | The response needs no judgment | Automate it. "Pages with rote, algorithmic responses should be a red flag." (SRE Ch. 6, p. 95) |
| The alert fires often and nobody can remove the root cause | The human is a slow relay | "the alert response deserves to be fully automated." (SRE Ch. 6, p. 93) |
| The page is about a novel problem | The response needs intelligence | Keep the human. (SRE Ch. 6, pp. 92–93) |
| The justification is "this symptom appeared last time" | You have a temporal correlation and no mechanism | Do not automate. See `integration-fault-catalog.md`, `I-48`. (Release It! Ch. 17, pp. 281–283) |
| The action cannot be reversed | A wrong run is permanent | Do not automate the action. Automate the refusal. (SRE Ch. 24, p. 384) |
| The action runs a few times per year | The feedback cycle is long | Automate it, then run it often. Rare automation is fragile. (SRE Ch. 7, p. 103) |
| The vendor call writes money or a record | A repeat can duplicate the write | Run `data-systems-design` first. See `I-14`. |

**The alert-review question that gates this table.** SRE asks it directly. "Could the action
be safely automated?" (SRE Ch. 6, p. 92.) The word "safely" carries the rest of this file.

---

## 3. The healing ladder

The rows sit in blast-radius order. Read the table downward. **Take the highest row that
solves the problem.** A lower row is not a stronger fix. It is a wider one.

| Action | Trigger signal | Precondition | Stop condition | Automatic? |
|---|---|---|---|---|
| Retry one request | The vendor returned a retriable error class | The operation is idempotent or carries a key. Budget remains. The breaker is closed. | The budget is spent, or the error is permanent | Yes |
| Fail one request fast | The breaker is open, or a required resource is missing | The caller has a path for an immediate refusal | — | Yes |
| Open the breaker | The failure count for this dependency passed its threshold | One breaker per integration point. A distinct exception exists. | The half-open trial call succeeds | Yes |
| Serve the degraded answer | The breaker is open, or the deadline expired | The degraded mode is written down, approved, and tested | The dependency recovers | Yes |
| Throttle the pool for this dependency | Threads block on checkout for this pool | The pool is dedicated to this vendor. The code tolerates a refused checkout. | Utilization returns below the threshold | Yes, with a floor above zero |
| Shed load by criticality | Utilization approaches the configured threshold | Criticality is set at the edge and propagated | Utilization returns below the threshold | Yes |
| Drain traffic from one location | An independent probe confirms the location cannot serve | Another location holds the capacity | The probe recovers | Yes, rate limited |
| Restart one process | A thread dump shows a deadlock, blocked threads, or a garbage-collection death spiral | The fault is localized. The restart cannot amplify it. | N consecutive restarts fail | Yes, canaried and rate limited |
| Fail over a region | The region fails your own synthetic transaction | Capacity is confirmed at the target. State loss is accepted. | — | Human approval |
| Roll back a release | The change log names a change inside the fault window | The rollback path itself was tested | — | Human approval |
| Page a human | Every automatic step is spent, or the action is not reversible | — | — | Always permitted |

**Sources for the rows.** Retry budgets, SRE Ch. 21, p. 306. Breaker states, Release It!
Sec. 5.2, p. 116. Degraded answers, SRE Ch. 21, p. 297. Pool throttle with a floor above
zero, Release It! Ch. 16, pp. 262–263. Criticality, SRE Ch. 21, pp. 302–303. Drain,
SRE Ch. 13, p. 195. Restart after localization, SRE Ch. 22, p. 340. Change log, SRE Ch. 22,
p. 335. Tested rollback, SRE Ch. 13, p. 191.

### Notes on five rungs

**The retry rung is the only rung that can amplify the fault by itself.** Every other rung
removes load. A retry adds load to a dependency that already fails. Give it randomized
exponential backoff, a per-request budget of three attempts, and a per-client budget that
stops above a 10% retry ratio. (SRE Ch. 22, p. 326. SRE Ch. 21, p. 306.) Read
`05-overload-and-load-shedding.md`.

**The breaker rung needs a distinct exception.** An open breaker must not look like an
ordinary call failure. The caller chooses the degraded path from that exception.
(Release It! Sec. 5.2, p. 116.) See `I-17`.

**The throttle rung needs a floor above zero.** Nygard's team set the scheduling pool maximum
to one when load was heavy, not to zero. A zero maximum disables the feature completely.
(Release It! Ch. 16, pp. 262–263.) A control that only reaches zero is too coarse.

**The restart rung needs localization first.** "Make sure that you identify the source of the
cascading failure before you restart your servers." (SRE Ch. 22, p. 340.) Confirm that the
restart does not merely move the load. Canary it on one instance. Then apply it slowly. See
`I-46` and `06-cascading-failure.md`.

**The last three rungs need a human.** A region failover, a release rollback, and a page all
change state that a later run cannot recover. SRE prefers a skipped launch over a double
launch for the same reason. (SRE Ch. 24, p. 384.)

---

## 4. The four properties of a safe remediation

Every automated remediation must hold all four. A remediation that holds three is not
partially safe. It is unsafe.

### 4.1 Idempotent

**Write the remediation so that a second run changes nothing.** Google required idempotent
fix scripts so a team could run them every fifteen minutes without damage to the cluster
configuration. (SRE Ch. 7, p. 110.)

**A multi-step remediation needs two synchronization points per action.** Record one before
the action. Record one after it. (SRE Ch. 24, p. 391.) Then a run that fails part of the way
through can find the steps that completed.

**Name the action before you perform it.** Construct the identifier first, with no mutating
call. Distribute it. After a failure, query the state of each named action and perform only
the missing ones. (SRE Ch. 24, p. 392.) Put the attempt time in the name. Without it a slow
failover repeats a completed action. (SRE Ch. 24, pp. 392–393.)

**The honest caveat from the book.** The test-fix-test loop has latency. That latency
produces a flaky test. A fix that is not idempotent, applied after a flaky test, can leave
the system in an inconsistent state. (SRE Ch. 7, p. 111.) The authors call their own approach
deeply flawed. Keep that hedge.

### 4.2 Rate limited

**Cap the actions per minute and cap the fraction of the fleet that one run may touch.**
Google added rate limiting to the decommission automation after Diskerase.
(SRE Ch. 7, p. 118.)

**A rate limit is the control that converts a logic defect into a small incident.** The
Diskerase automation had no cap, so a single degenerate input reached every colocation site
within minutes. (SRE Ch. 7, pp. 117–118.)

**Rate limiting also protects the target.** In the global crash-loop incident the affected
system rate limited how quickly it gave full updates to new clients. That limit may have
throttled the crash-loop and kept jobs alive long enough to serve a few requests.
(SRE Ch. 13, p. 193.) See `I-42`.

### 4.3 Observable

**A remediation records the actor, the action, the target, the time, and the result.**
Google's Admin Server logs the requestor, the parameters, and the results of every RPC. The
stated purpose is debugging and the security audit. (SRE Ch. 7, p. 113.)

**Publish a "manual override applied" state.** Nygard lists it as an exposed parameter for
every integration point and for every circuit breaker. (Release It! Ch. 17, p. 271.) A
responder must be able to see that a human or an automation pinned the component.

**Expose the internal detail that the automation acts on.** A system that moves toward
autonomy needs self-introspection. The details that the introspection relies on must also
reach the humans who manage the system. (SRE Ch. 7, p. 116.) See `I-18` and `I-35`.

### 4.4 Reversible

**Test the reverse path before you trust the forward path.** An SRE test blocked access to
one database of a hundred. The team then tried to reverse the permissions change and failed,
because the rollback procedure had never run in a test environment. The outage grew longer.
(SRE Ch. 13, pp. 190–191.)

**Keep a control path that works when the normal interface does not.** Google keeps
command-line tools and alternative access methods that perform updates and rollbacks when
other interfaces are unreachable. Engineers must use them routinely. (SRE Ch. 13, p. 193.)

**Prefer a runtime control to a deploy.** Nygard reconfigured and recycled one connection
pool in under five minutes. A configuration-file change and a full restart would have taken
more than six hours under that load. (Release It! Ch. 16, p. 263.)

---

## 5. The kill switch

**Build one control that stops every automated action, and test it.** During the Diskerase
incident the on-call engineers disabled all team automation to prevent further damage.
(SRE Ch. 13, p. 195.) That response needs the control to exist before the incident.

**Give operations a way to trip and to reset each breaker directly.** Nygard asks for this so
operations can disable a bad integration without a deploy. (Release It! Sec. 5.2, p. 117.)

**Add a per-feature disable property.** Nygard's team had to improvise a throttle from a pool
maximum because the developers had never added an enabled property.
(Release It! Ch. 16, p. 262.)

**A feature-flag platform is the common form of this control today. It is Modern.** The books
predate it. The requirement they state is the switch, not the product.

**Rules for the switch.**

1. One switch stops every automated remediation at once.
2. A second switch stops one integration point.
3. The switch takes effect without a deploy.
4. The switch state appears as "manual override applied", and the system records who used it.
5. Exercise the switch on a schedule. An untested switch is a plan, not a control. See `I-45`.

---

## 6. A remediation may not make a change it cannot detect the effect of

This is the rule that the other rules serve. State it in the Integration Record.

**Do not infer safety from the absence of a signal.** Automation assumed that an unused first
disk meant no storage was configured, so the machine was safe to wipe. It wiped a
multi-petabyte Bigtable cluster. "Automation needs to be careful about relying on implicit
'safety' signals." (SRE Ch. 7, p. 107.)

**Model the partial state.** A push to a cluster is not atomic. A binary can be staged and
not pushed, pushed and not restarted, or restarted and not verifiable.
(SRE Ch. 7, p. 102.) An automation that cannot model these states must halt and call for a
human. SRE notes that bad automation systems do not even do that.

**Stop after repeated failure and notify a human.** Google's fix loop assumes the fix failed
after several attempts. It stops. It tells the user. (SRE Ch. 7, p. 110.) Set N in the
Integration Record. The book gives no number. The number is a local decision.

**Never let a remediation change the observability plane.** The turndown automation removed
monitoring for the small installations. The on-call engineers had to reverse those monitoring
changes before they could measure the damage. (SRE Ch. 13, p. 196.) Nygard states the same
boundary from the other side. A failure in the operations database must have no noticeable
effect on the primary function of the system. (Release It! Ch. 17, p. 302.) See `I-47`.

**Verify recovery with an external signal.** Nygard declared recovery when the external
synthetic monitor went green about ninety seconds after the change, not when an internal
metric moved. (Release It! Ch. 16, p. 263.) A vendor status page is not that signal. See
`I-26`.

---

## 7. Degenerate input is the most common defect in a remediation

**Treat an empty target set as an error.** Never treat it as "all". The Diskerase automation
received an empty response for the machine rack. It passed the empty filter to the machine
database, and the database read it as everything. "Yes, sometimes zero does mean all."
(SRE Ch. 13, p. 196.) See `I-43`.

**Put sanity checks on the commands the automation sends.** SRE names this as the root cause
of the process-induced emergency. The turndown automation server lacked them.
(SRE Ch. 13, p. 196.)

**Cap the target count as well as the rate.** A run that names more targets than the change
justifies must refuse and ask for a human. Place these checks before the executor.

1. Reject an empty target list.
2. Reject a target list above the configured cap.
3. Reject a target that the localization step did not name.
4. Reject a run while the kill switch is active.
5. Record the rejection with the same detail as an action.

---

## 8. Attacks of self-denial

Nygard's name for this class is **Attacks of Self-Denial**. His definition covers any
situation in which the system, or the extended system that includes humans, conspires against
itself. (Release It! Sec. 4.6, p. 88.) An automatic retry storm belongs to this class. We
build it, we aim it at ourselves, and it needs no attacker.

**The arithmetic.** A 1% overload becomes a self-sustaining flood when the retry has no
backoff. Google's worked case grows from 100 to 200 to 300 extra queries per second until the
backend fails. (SRE Ch. 22, pp. 325–326.) Three retries at each of three layers give 64
attempts against the vendor for one user action. (SRE Ch. 22, p. 327.) A three-attempt cap
alone still lets request volume grow to just under 3x. The 10% per-client budget reduces the
growth to about 1.1x. (SRE Ch. 21, p. 306.)

**A remediation that fires too often is the same class.** Nygard says a cache flush triggered
too often produces attacks of self-denial. His rule is to limit how often a flush can be
triggered. (Release It! Sec. 10.2, p. 209.) Read that as the general rule for every
remediation trigger.

**A shared resource turns one defect into a farm-wide stall.** A single programming error
made thousands of request-handling threads on hundreds of servers wait for one write lock.
(Release It! Sec. 4.6, p. 89.)

**Nygard's remedies, in his stated order.** (Release It! Sec. 4.6, p. 89.)

1. Build a shared-nothing architecture.
2. Where that is impractical, apply decoupling middleware to reduce the effect of excessive
   demand.
3. Or make the shared resource horizontally scalable, with redundancy and a backside
   synchronization protocol.
4. Design a fallback mode for the case where the shared resource does not answer.

**Apply Fail Fast when the dedicated capacity stops answering.** Otherwise front-end resources
stay tied to a response that will not arrive. The human half of this class needs training,
education, and communication, so that you learn what is coming. (Release It! Sec. 4.6, p. 90.)

---

## 9. The war stories

**Diskerase, and the empty set that meant everything (SRE Ch. 7, pp. 117–118, and Ch. 13,
pp. 194–197).** Google decommissions racks in third-party colocation sites with automation.
One step overwrites the full disk content of every machine in the rack. The automation for one
rack failed after that step succeeded, so the process restarted from the beginning. On that run
the set of machines that still needed the erase was correctly empty. The empty set was a
special value that meant everything. Within minutes the automation wiped the disks of almost
every machine in every colocation site, and those machines could no longer terminate user
connections. Good capacity planning held the external effect to a slight latency increase.
Recovery took the better part of two days of reinstallation and then weeks of auditing. The
three remedies are the ones this file repeats. Add sanity checks. Add rate limiting. Make the
workflow idempotent.

**The global crash-loop, and the five-minute rollback (SRE Ch. 13, pp. 191–194).** A
configuration change to the abuse-protection infrastructure reached every location on a Friday.
That infrastructure touches almost every externally facing system, so the whole fleet began to
crash-loop at once. Internal applications also failed, because internal infrastructure depends
on the same services. Monitoring alerted within seconds. The push engineer pushed a second
configuration change to reverse the first within five minutes. SRE records the luck honestly.
That engineer happened to watch the real-time communication channels, which was not a normal
part of the release process. Three lessons are stated. Run a thorough canary regardless of the
perceived risk of the change. Keep out-of-band communications and command-line tools that work
when the normal interfaces do not.

**Voodoo Operations, and the failover that ran for six months
(Release It! Ch. 17, pp. 281–283).** Nygard watched an administrator take a page and
immediately start a database failover. The message that triggered it was a debug message Nygard
had written himself. It said that an encrypted channel to an outside vendor had carried enough
data that the key would soon be weak. The application then reset the channel by itself. The
message had nothing to do with the database. Six months earlier it had happened to be the last
line logged before a Sybase crash. There was a temporal connection and no causal connection.
The team ran a weekly production database failover during peak hours for six months. An
automated trigger built on the same evidence performs the same superstition at machine speed.

**Black Friday, and the throttle that saved the weekend (Release It! Ch. 16, pp. 256–263).** An
online retailer lost about a million dollars an hour on Black Friday morning. Thread dumps
showed three thousand front-end threads blocked on a connection pool with no timeout. They
waited on an order management system, whose own threads waited on an external home-delivery
scheduling system. That vendor handled about twenty-five concurrent requests and received about
ninety. The team rejected several proposals because nobody knew how the application behaved in
those states. The store happened to have a separate connection pool for scheduling requests,
and that pool was the only available throttle. They set the maximum to zero on one server and
learned that the maximum applies only at pool start-up. They recycled the component, verified
that node, and then applied the change to every server. The site recovered about ninety seconds
later. Through the weekend the sponsor moved the maximum up when load was light and down to
one, not zero, when load was heavy.

**The airline, and the restart that healed one consumer and not the other (Release It! Ch. 2,
pp. 24–25).** Two consumer systems failed. The team restarted the shared dependency tier. The
interactive voice response system recovered. The kiosks did not, because their own thread pools
were still poisoned, so the team restarted that tier too. Nygard also names the blunt option
and its cost. You can restart every server, layer by layer, and that is almost always
effective, but it takes a long time. The escalation trigger was the one-hour service level
agreement clock. An automated restart must consider the consumers of a dependency, not only the
dependency.

**The idempotent fix loop, and its stated flaw (SRE Ch. 7, pp. 108–111).** Google paired each
production test with an idempotent fix, so a team could run the fix every fifteen minutes
without fear of damage. A fix that failed several times made the automation stop and notify a
user. A small group then reached one percent of live search and advertisement traffic in a week
or two. Each cluster had taken six or more weeks before that. The authors then call the
approach deeply flawed, because the delay between the test and the fix produced flaky tests.
Keep both halves of that finding.

---

## 10. The gate

Answer every question before a remediation reaches production. Record the answers in the
Remediation section of the Integration Record.

1. What signal triggers this action, and what mechanism connects the signal to the fault?
2. Which rung of Section 3 is this, and does a higher rung solve the problem?
3. What is the precondition, and who checks it?
4. What is the stop condition, and what happens after it?
5. Is the action idempotent? If not, what key or state lookup makes a repeat safe?
6. What is the rate limit, and what is the cap on targets per run?
7. What does the action record, and where does a responder read it?
8. How does a human reverse the action, and when did you last test that path?
9. Where is the kill switch, who may use it, and when did you last test it?
10. What effect does the action produce that you can measure from outside the system?
11. What does the automation do with an empty target set?
12. Does this action touch monitoring, logging, or alerting? If yes, remove that step.

**"Not applicable" is a valid answer to this whole file.** A change that adds no automated
action needs no entry here. Say so and continue.

---

## 11. What this file does not decide

| Question | Read this |
|---|---|
| Which defense to apply to an integration point | `03-stability-patterns.md` |
| Where the fault is | `04-fault-localization.md` |
| Retry budgets, criticality, quotas, and deadlines | `05-overload-and-load-shedding.md` |
| Why the failure crossed a layer boundary | `06-cascading-failure.md` |
| How to prove the failure path works | `08-fault-injection-and-test-harness.md` |
| Which degraded mode to serve | `09-vendor-slas-and-degradation.md` |
| The named defects `I-42` to `I-48` | `integration-fault-catalog.md` |
| Whether repeating a write is safe | `data-systems-design`, hazard catalog |
