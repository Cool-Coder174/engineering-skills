# Progressive Rollout and Canarying

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 17 "Testing for Reliability", p. 226–230, p. 236–237, p. 243–246. Ch. 27 "Reliable
Product Launches at Scale", p. 448–470. Ch. 6, p. 88. Ch. 9, p. 132–133. Ch. 26, p. 421.
Ch. 32, p. 542. App. B, p. 571–572. App. D, p. 583. *Release It! Design and Deploy
Production-Ready Software* — Michael T. Nygard (Pragmatic Bookshelf, 2007). Sec. 18.4
"Releases Shouldn't Hurt", p. 331–334.

This file answers one question. How much of the world sees the change before you can undo it?

One word, one meaning, in this file. **Revert** is the verb. It returns the system to the
previous state. **Reverse action** is the noun for that step. **Stage** is one rung of the
ladder. **Exposure** is the fraction of user traffic that reaches the new version.

---

## 1. Why a canary belongs in a reversibility skill

**A canary limits the size of the mistake. It does not prove that the change is correct.**
The revert stays cheap because few users saw the change. That is the whole value.

**Any change carries risk.** SRE Ch. 27, p. 461 states the rule for a replicated, globally
distributed system. App. B, p. 572 gives the remedy. Apply the change to a small fraction of
traffic and capacity at one time.

**A canary test is not a test.** SRE Ch. 17, p. 229 calls it "structured user acceptance".
A configuration test and a stress test confirm a specific condition over deterministic
software. A canary only exposes the new code to live production traffic. Nobody predicts
that traffic.

**The canary and the reverse action are one mechanism.** SRE Ch. 17, p. 229 states that the
modified servers can return to a known good state quickly. A canary with no fast reverse
action buys you nothing. It only tells you that you have a problem.

---

## 2. The rollout ladder

SRE Ch. 27, p. 461–462 gives the order for a server change. Each stage has a signal that
permits the next stage.

| Stage | What the actor changes | Signal to advance | Reverse action | Source |
|---|---|---|---|---|
| Dark launch | Nothing that a user sees. The system copies traffic to the new service and discards the responses. | The new service serves the copied traffic without error | Stop the traffic copy | SRE Ch. 32, p. 542 |
| Canary | A few machines in one datacenter | No unexpected variance during the incubation period | Revert the modified servers to the known good state | SRE Ch. 17, p. 228–229. SRE Ch. 27, p. 462 |
| One datacenter | Every machine in one datacenter | A second observation period with no variance | Drain the datacenter. Deploy the previous artifact. | SRE Ch. 27, p. 462 |
| Global | Every machine in every datacenter | None. This is the last stage. | Full artifact reversal, then a staged restore of traffic | SRE Ch. 27, p. 462. SRE App. D, p. 583 |
| Contract step | The schema loses the old shape | The release baked long enough for the team to accept it | None. Restore the data from a backup. | Release It! Sec. 18.4, p. 334 |

**The ladder covers configuration, not only a binary.** App. B, p. 572 names both. Ch. 6,
p. 83 defines a push as any change to running software or its configuration.

**The ladder covers code that does not run on your machines.** SRE Ch. 27, p. 462 describes
a gradual rollout of an Android application. The team offers the new version to a subset of
installations. The percentage grows over time until it reaches 100 percent.

**The ladder covers the number of users, not only the number of machines.** SRE Ch. 27,
p. 462 names the invite system. The service permits a limited number of signups per day.

---

## 3. The exposure ladder numbers

Use these numbers. The books give them. Every other percentage is a local decision.

| Ladder | Numbers the book gives | Source |
|---|---|---|
| Canary traffic, first stage | 0.1 percent of user traffic | SRE Ch. 17, p. 246, fn. 89 |
| Canary growth | One order of magnitude every 24 hours. Day 2 gives 1 percent. Day 3 gives 10 percent. Day 4 gives 100 percent. | SRE Ch. 17, p. 246, fn. 89 |
| Feature flag group | Between 1 and 10 percent of users | SRE Ch. 27, p. 463 |
| Traffic restore after a reverse action | 1 percent, then 10, 30, 50, and 100 percent | SRE App. D, p. 583 |
| Bake on a small server set | A couple of servers, for a day or more | Release It! Sec. 18.4, p. 334 |
| Whole rollout duration | A few hours to a few days | Release It! Sec. 18.4, p. 333 |
| Low-risk launch threshold | No new server executable, and a traffic increase under 10 percent | SRE Ch. 27, p. 468 |

**Do not invent a percentage.** App. B, p. 572 states what sets the numbers. The size of the
service, the size of the rollout, and the risk profile inform the percentages and the time
between stages. Write your numbers into the Reversal Record and name the reason.

**Vary the geography at each stage.** SRE App. B, p. 572 gives the reason. Different
geographies expose problems that relate to diurnal traffic cycles and to a different traffic
mix. SRE Ch. 17, p. 246, fn. 89 repeats the instruction for the canary ladder.

---

## 4. Bake time

**"Baking the binary" is the name for the incubation period of an upgraded canary server.**
SRE Ch. 17, p. 229. Nygard calls the same practice the bake. Release It! Sec. 18.4, p. 334.

**A stage ends on an observation window and a metric. A stage does not end on job success.**
This is hazard R-18 in `rollback-hazard-catalog.md`.

**The books give a range, not a formula.** Release It! Sec. 18.4, p. 333 gives a few hours
to a few days. Release It! Sec. 18.4, p. 334 gives a day or more for a couple of servers.
SRE Ch. 17, p. 246 uses 24 hours per order of magnitude. Any other bake time is a local
decision.

**A bake shorter than one day does not observe the daily peak.** SRE App. B, p. 572 ties the
stages to diurnal traffic cycles. A change that fails only at peak load survives a short
bake.

**Name the metric that gates each stage.** SRE Ch. 6, p. 88 names the four golden signals.
They are latency, traffic, errors, and saturation. App. B, p. 572 adds the constraint. Measure
availability and performance in terms that matter to an end user.

---

## 5. The canary variance model

SRE Ch. 17, p. 229 gives a model for an exponential rollout. It is `CU = RK`.

| Symbol | Meaning |
|---|---|
| C | The growing cumulative number of reported variances |
| R | The rate of those reports |
| U | The order of the fault |
| K | The period over which traffic grows by a factor of e, or 172 percent |

**Estimate the order of the fault after the reverse action, not before it.** SRE Ch. 17,
p. 229 gives the procedure. Several more reports arrive while the automation observes the
variances and responds. Once the reports settle, estimate C and R. Divide and correct for K.
The result estimates U. A worked example gives K at about 10 hours and 25 minutes, for a
24-hour interval between 1 and 10 percent. SRE Ch. 17, p. 246, fn. 90.

**The order of the fault tells you what else you must undo.** SRE Ch. 17, p. 229.

| Order | What the book says | What the reverse action must cover |
|---|---|---|
| U=1 | The request met code that is simply broken. | Revert the artifact. Convert the logs of unusual responses into new regression tests. |
| U=2 | The request randomly damages data that a future request may see. | Revert the artifact. Then repair the data. Read `06-data-and-schema-reversibility.md`. |
| U=3 | The damaged data is also a valid identifier to an earlier request. | Revert the artifact. Then repair the data and the identifiers. Escalate to a person. |

**Most faults are order one.** SRE Ch. 17, p. 229. They scale linearly with user traffic.

**A code revert alone does not fix an order-two fault.** The artifact returns. The damaged
rows stay. This is hazard R-26 and hazard R-30 in `rollback-hazard-catalog.md`.

**Keep K small.** SRE Ch. 17, p. 230 gives the method. Use many methods to establish a
traffic fraction, in sequence, with some overlap. A small K reduces the total user-visible
variance count C. It still permits an early estimate of U.

---

## 6. Who supervises a stage, and who stops it

**A rollout must be supervised.** SRE App. B, p. 572. The engineer who performs the stage may
supervise it. A monitoring system that you can demonstrate to be reliable is the better
choice. This is hazard R-19 in `rollback-hazard-catalog.md`.

**On unexpected behavior, revert first and diagnose second.** SRE App. B, p. 572 states the
reason. This order minimizes Mean Time to Recovery. Read `04-change-induced-emergency.md`.

**Automation reverts a canary that fails its validation period.** SRE Ch. 27, p. 462. The
tool that installs the software observes the new server for a while. It confirms that the
server does not crash and does not misbehave.

**A failed readiness check pauses the update and sends no user traffic to the new version.**
SRE Ch. 17, p. 244. The update fails safely and indefinitely. It waits for an engineer to
diagnose the fault and then to revert cleanly. Prefer this behavior over automatic advance.

**Never wire an automatic reverse action to a flaky signal.** SRE Ch. 17, p. 236 defines a
flaky test. Repeated and seemingly identical runs give different results. SRE Ch. 17, p. 237
gives the scale of the problem. A service with more than 21,000 tests produces 42,000 results
per patch comparison. To reject only 1 patch in 100 good patches, each test must run
correctly over 99.9999 percent of the time. A revert trigger on a weaker signal reverts good
changes. This is hazard R-43.

---

## 7. What a canary cannot catch

**The canary is ad hoc, and it does not always catch a new fault.** SRE Ch. 17, p. 229 states
this plainly. Plan the other defenses.

| The canary misses | Why it misses | What catches it | Source |
|---|---|---|---|
| A rare fault | A fault that rarely affects user traffic produces few variance reports at 0.1 percent exposure | A larger stage, plus the U estimate after the reverse action | SRE Ch. 17, p. 229 |
| An overload fault | A small traffic fraction never reaches the nonlinear region near saturation | A load test. Request deadlines. Load shedding. | SRE Ch. 27, p. 458, p. 465–466 |
| A launch traffic spike | The ladder assumes today's traffic. Some Google products met a spike up to 15 times the first estimate. | Capacity planning and a kill switch | SRE Ch. 27, p. 457 |
| A version-pair fault | The canary runs beside old peers. It never meets every combination. | Version-pair monitoring across the service interface. Rollout automation that blocks a bad combination. | SRE Ch. 17, p. 244–245 |
| Slow data corruption | A low-grade deletion fault can stay invisible for months | Out-of-band validators, and a retention window longer than the detection window | SRE Ch. 26, p. 421 |
| A production configuration fault | Production is intentionally not representative of any single checked-in version during a rollout | A configuration test that compares live production against the file | SRE Ch. 17, p. 226–228 |
| A client behavior fault | Server-side exposure control does not limit what an installed client does | Exponential backoff, jitter, and server-side client configuration | SRE Ch. 27, p. 464–465 |

**A canary reduces the blast radius. It does not remove the need for a reverse path.** Read
`01-the-reversibility-contract.md`.

---

## 8. Dark launch

**A dark launch gives you production traffic at zero user exposure.** SRE Ch. 32, p. 542
describes the mechanism. The system sends part of the traffic from existing users to the new
service, in addition to the live production service. The responses from the new service are
dark. The system discards them. The team gains operational insight and resolves problems
without user impact.

**The reverse action is to stop the traffic copy.** Reverse time is seconds. Nothing that a
user saw changes.

**Discarding a response does not discard a write.** SRE Ch. 32, p. 542 says the system
discards the responses. It says nothing about a row. Name every write that the dark service performs
before you start it. Send those writes to a separate store, or block them. A dark launch that
writes to the live store has the exposure of a full rollout for that data.

---

## 9. The feature flag as a decoupled reversal switch

**A feature flag separates the exposure decision from the deploy.** Then the reverse action
falls from a deploy to a configuration change. SRE Ch. 27, p. 462 gives the purpose. It rolls
out changes slowly and permits observation of total system behavior under real workloads.

**SRE Ch. 27, p. 463 lists six requirements for a feature flag framework.** All six matter to
reversibility. The fifth one is the reversal contract.

1. Release many changes in parallel, each to a few servers, users, entities, or datacenters.
2. Grow to a larger but limited group of users, usually between 1 and 10 percent.
3. Direct traffic through different servers by user, session, object, or location.
4. Handle failure of the new code path by design, without user impact.
5. Revert each change alone and at once, on a serious fault or side effect.
6. Measure how much each change improves the user experience.

**A flag that gates several unrelated changes violates requirement 5.** The operator then
removes good changes with the bad one, so the operator hesitates. This is hazard R-20.

**The book names two classes of framework.** SRE Ch. 27, p. 463.

| Class | Mechanism | Scope key | Source |
|---|---|---|---|
| User interface change, stateless service | An HTTP payload rewriter at the frontend application servers | A subset of cookies or a similar request attribute. A cookie hash mod range, plus a whitelist and a blacklist. | SRE Ch. 27, p. 463–464 |
| Server-side or business logic change, stateful service | A proxy or a reroute of the request to a different server | The logged-in user identifier, or the product entity identifier | SRE Ch. 27, p. 464 |

**A kill switch is a flag that you build before the launch, not during the incident.** SRE
Ch. 27, p. 449 names the switches that SRE built into the NORAD Tracks Santa experience. The
team called them the "make-children-cry switches".

**A client can carry dormant functionality.** SRE Ch. 27, p. 465. The team ships the code
inactive inside the client application. The client downloads a server-side configuration file
that enables or disables the feature. Then the team aborts a launch with a configuration
change. This also avoids a combinatorial explosion of client versions.

**Delete the code after the flag retires.** SRE Ch. 9, p. 132–133. Source control reverses
the change. A disabled flag that guards dead code is hazard R-21.

**Modern.** A hosted flag platform postdates both books. So do the SDK, the targeting rule,
and the audit log that such a platform provides. The requirement list above still governs the
choice. A platform that cannot revert one change alone and at once does not satisfy SRE
Ch. 27, p. 463.

**A flag is configuration.** SRE Ch. 27, p. 459 requires that you check all code and
configuration files into version control. Give the flag value the same review and the same
ladder. Read `02-release-engineering.md`.

---

## 10. Deploy topology and reversal property

`system-design` chooses the topology. This skill states only the reversal property of each
one.

| Topology | Reverse action | Reverse time | What it does not reverse | Source |
|---|---|---|---|---|
| Two service pools in the load balancer | Drain the new pool. Send every session to the old pool. | Minutes | The schema. The bridging triggers still run. | Release It! Sec. 18.4, p. 334 |
| Rolling deploy | Deploy the previous artifact to the changed machines | Minutes to tens of minutes | Nothing during the reversal. Both versions run again while the reversal proceeds. | Release It! Sec. 18.4, p. 331 |
| Canary pool | Revert the modified servers to the known good state | Minutes | Data that the canary already damaged | SRE Ch. 17, p. 229 |
| Feature flag | Disable the flag | Seconds | Data that the new code path already wrote | SRE Ch. 27, p. 463 |
| Dark launch | Stop the traffic copy | Seconds | Any write that the dark service performed | SRE Ch. 32, p. 542 |
| Blue and green pools (**Modern** name) | Move traffic back to the old pool | Seconds to minutes | The schema, and every external side effect | The mechanism is Nygard's two service pools, Release It! Sec. 18.4, p. 334 |

**A rolling deploy runs two versions at once, by construction.** Release It! Sec. 18.4,
p. 331. Redundancy means that some servers run version N and some run version N+1. Enterprise
applications reference databases, web services, search engines, and asset URLs. Each
reference is an opportunity for a version conflict.

**No topology in this table reverses the database.** Read Section 11.

---

## 11. The ladder protects the code. It does not protect the schema.

Release It! Sec. 18.4, p. 331–334 names three phases. They are Expansion, Rollout, and
Cleanup. This skill calls the third phase the contract step.

`data-systems-design/references/04-encoding-and-evolution.md`, Section 4, owns the
compatibility mechanics of this pattern. This section states only which phase keeps a code
revert safe.

| Phase | What the actor does | Is a revert of the code safe? |
|---|---|---|
| Expansion | Adds a nullable column, a table, a new endpoint name, a versioned asset URL. Installs bridging triggers. | Yes |
| Rollout | Deploys the new artifact. Bakes it. Runs both versions behind two pools. | Yes |
| Cleanup, the contract step | Removes the bridging triggers and the extra pools. Drops unused columns and tables. Converts columns to NOT NULL. Adds referential integrity. | No |

**Cleanup is the point of no return of the whole rollout.** Release It! Sec. 18.4, p. 334
gives the precondition. Run Cleanup only after every application server runs the new version,
and only after the release bakes long enough for the team to accept it.

**Add every future NOT NULL column as nullable during Expansion.** Release It! Sec. 18.4,
p. 333. The old version does not know how to fill the column. Add no referential integrity
rule during Expansion, because the old version violates it at once.

**Bridging triggers are what make the code revert safe.** Release It! Sec. 18.4, p. 333. One
trigger fills the new structure from an old-version insert. The other fills the old structure
from new-version data. Without them the old version reads the new data as corrupt or
incomplete.

Read `06-data-and-schema-reversibility.md` for the restore side.

---

## 12. War stories

**NORAD Tracks Santa (SRE Ch. 27, p. 448–449).** Keyhole serves satellite imagery for Google
Maps and Google Earth. It normally serves up to several thousand images per second. On
Christmas Eve 2011 it received 25 times its normal peak, and more than one million requests
per second. The launch had a hard deadline that could not slip, heavy publicity, an audience
of millions, and a very steep traffic ramp. SRE prepared the infrastructure and built kill
switches into the experience. The team named them the "make-children-cry switches". This
proves the rule for a launch whose ladder cannot be slow. When you cannot control the
exposure, build the switch that sheds the feature.

**The logging death spiral (SRE Ch. 27, p. 466).** A service logged debugging information in
response to a backend error. Logging that information cost more than a normal backend
response. As the service became overloaded, it timed out backend responses inside its own RPC
stack. It then spent even more CPU to log those responses. More requests timed out. The
service ground to a halt. This proves why a canary at low exposure cannot report an overload
fault. The error path costs more than the success path, and only a load test finds it.

**The empty DNS entry file (SRE App. B, p. 571–572).** In 2005 Google's global DNS load and
latency balancing system received an empty DNS entry file, because of file permissions. The
system accepted the empty file. It served NXDOMAIN for six minutes for every Google property.
The system now performs sanity checks on a new configuration. It confirms the presence of
virtual IP addresses for google.com. It continues to serve the previous entries until a new
file passes the input checks. This proves two rules. A configuration change needs the same
ladder as a binary. The safe state under an implausible input is the previous state.

**The Shakespeare traffic restore (SRE App. D, p. 583).** After the team mitigated the outage
at 15:36, one engineer restored load balancing across all clusters for 1 percent of traffic at
15:41. At 15:43 the HTTP 500 rates were nominal. She moved to 10 percent at 15:45, and the
rates stayed within SLO. She moved to 30 percent at 15:50 and 50 percent at 15:55. All traffic
was balanced at 16:00. The incident closed at 16:30, at the exit criterion of 30 minutes of
nominal performance. This proves that the ladder governs the return path as well as the
outbound path. Read `04-change-induced-emergency.md`.

**The low-risk launch lane (SRE Ch. 27, p. 467–468).** One Launch Coordination Engineer ran
350 launches through the checklist in 3.5 years. A team of five engineers ran more than 1,500
in the same period. To keep pace, the engineers defined a low-risk class. A launch qualified
when it added no new server executable and increased traffic by under 10 percent. That class
received an almost trivial checklist. By 2008, 30 percent of reviews were low-risk. This
proves that you scale a gate by risk-tiering the review. It does not license a canary
exemption. SRE Ch. 13, p. 194 requires a canary for every change, whatever its perceived risk.
That is hazard R-17.

---

## 13. What to write in the Reversal Record

Fill this table in the Reversal Record for every change above the smallest class. `../SKILL.md`
holds the full record format.

| Field | What it must name |
|---|---|
| Stage | The rung of the ladder |
| Exposure | The percentage, and the reason for that percentage |
| Geography | The region for this stage, and why it differs from the last stage |
| Bake time | The observation window, and whether it covers a daily peak |
| Gate metric | The SLO name and the threshold, in end-user terms |
| Supervisor | The monitoring system that watches the stage |
| Reverse action | The exact command or the exact label move |
| Reverse time | The measured time from the last rehearsal |

---

## 14. Hazards this file governs

Run these entries from `rollback-hazard-catalog.md` against any change that deploys.

| Hazard | Short name |
|---|---|
| 🔴 R-16 | A one-step global rollout |
| 🔴 R-17 | A risk-based canary exemption |
| 🟡 R-18 | A rollout with no bake time |
| 🟡 R-19 | A rollout judged by the actor that pushed it |
| 🟡 R-20 | A feature flag that cannot revert alone |
| 🔴 R-21 | Dead code kept behind a disabled flag |
| 🟡 R-43 | A revert trigger on a flaky signal |

---

## 15. Where to read next

| Question | File |
|---|---|
| What is the reverse path for this change? | `01-the-reversibility-contract.md` |
| Can I rebuild and name the previous artifact? | `02-release-engineering.md` |
| A stage failed. What do I do first? | `04-change-induced-emergency.md` |
| Which change caused the variance? | `05-locating-the-bad-change.md` |
| The canary damaged data. What now? | `06-data-and-schema-reversibility.md` |
| Who may stop the rollout without a person? | `07-automation-safety.md` |
| What must the pre-merge gate ask? | `09-launch-coordination.md` |
| Which topology do I choose? | `system-design/references/03-scaling-ladder.md` |
