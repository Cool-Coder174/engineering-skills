# Test and Treat

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 12, sections "Test and Treat", "Negative Results Are Magic", and "Cure", p. 179–182. The
two test strategies come from `SRE Ch. 12, p. 171`. The treatment moves come from
`SRE Ch. 22, p. 339–341`. *Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007),
Sec. 16.7–16.11, p. 259–264.

This is step 5 of the loop. You hold a short list of possible causes. You must find which
factor is at the root of the fault (`SRE Ch. 12, p. 179`). You do that with the experimental
method, not with an argument.

**A hypothesis without a test that can fail is not a hypothesis.** It is an opinion. Step 5
converts each opinion into a test. It runs the tests in a defined order. It records the
result of every test, including every test that removed a hypothesis.

---

## 1. Where this step sits

| Signal | You are in | Read |
|---|---|---|
| You hold observations and no cause | Step 4, Diagnose | `03-examine-and-diagnose.md` |
| You hold two or more hypotheses | Step 5, this file | This file |
| Every test failed and no hypothesis remains | Return to step 3 or step 4 | `03-examine-and-diagnose.md` |
| One hypothesis survives every test | Step 6, Cure | `../SKILL.md`, Section 8 |

The model is a loop, not a line. Step 5 returns to Examine or to Diagnose until one cause
holds (`SRE Ch. 12, p. 170–171`).

**Step 5 does not wait for the mitigation to end.** A fix for the proximate cause does not
wait for the root cause (`SRE Ch. 12, p. 171`). Triage owns the mitigation. Read
`02-triage-first-diagnose-second.md`.

---

## 2. The two ways to test a hypothesis

The book names two strategies (`SRE Ch. 12, p. 171`). Choose one per hypothesis. Record
which one you chose.

| Strategy | What you do | What it costs | When you choose it |
|---|---|---|---|
| **Compare** | Read the observed state. Match it against the theory. | Nothing. It changes no state. | Always try this first. |
| **Treat** | Change the system in a controlled way. Observe the result. | It changes production state. | The observed state cannot separate the hypotheses. |

**Compare before you treat.** A comparison reads the system. A treatment changes the system.
A treatment spends error budget (`SRE Ch. 1, p. 27`). It can also confound every later test.

**A treatment refines your understanding of the state and of the possible causes**
(`SRE Ch. 12, p. 171`). It is a legitimate method. It is not a guess with a restart attached.

---

## 3. Write a test that can fail

A test can be as simple as one echo request. A test can also remove traffic from a cluster
and then inject specially formed requests to find a race (`SRE Ch. 12, p. 179`). The
complexity does not matter. The failure condition does.

**State the failure condition before you run the test.** Write the sentence that you will
believe if the test fails.

| Form to reject | Required form |
|---|---|
| "I think the database is slow." | "The database answers a query from the application server in more than 200 ms." |
| "Maybe the network is bad." | "An echo request from the application server to the database host receives no answer." |
| "The deploy broke it." | "The fault start time is inside the deploy window of change `abc123`." |
| "Something is wrong with the cache." | "The cache hit rate at the fault start time is below the rate one hour before it." |

The right column names an observation with a threshold. Every threshold above is an example.
**The books state no threshold for these. Each number is a local decision.** Record the number
that your service commits to, and record where that commitment is written.

**Follow the code and imitate the code flow, step by step** (`SRE Ch. 12, p. 179`). This turns
a vague hypothesis into a specific hop that you can measure.

---

## 4. Order the tests

**Consider the obvious first. Run the tests in decreasing order of likelihood, and consider
the risk that each test poses to the system** (`SRE Ch. 12, p. 179`).

The book gives the order with an example. Test the network connection between two machines
before you investigate whether a recent configuration change removed a user's access to the
second machine (`SRE Ch. 12, p. 179`).

| Likelihood | Risk to the system | Order |
|---|---|---|
| High | Low | Run it now |
| High | High | Run the low-risk tests first. Then prepare this one. |
| Low | Low | Run it while you wait for a slow result |
| Low | High | Do not run it. Find a comparison instead. |

**Two properties decide the order.** The first is how many hypotheses the test removes. The
second is what the test can break. A cheap test that removes four hypotheses beats an
expensive test that removes one.

**Prefer the common cause.** Not all failures are equally probable
(`SRE Ch. 12, p. 171`). Prefer the simpler explanation. The counterweight is real. Footnote 62
names Hickam's dictum. Several common low-grade faults together can explain the symptoms
better than one rare fault (`SRE Ch. 12, fn. 62, p. 187`).

---

## 5. The worked example from the book

Two hypotheses remain. The network between the application logic server and the database
server failed. Or the database refuses connections (`SRE Ch. 12, p. 179`).

| Test | Vantage point | Pass | Failure | Which hypothesis it removes |
|---|---|---|---|---|
| Connect to the database with the credentials that the application server uses | The application server | The database accepts the connection | The database refuses the connection | A pass removes "the database refuses connections" |
| Send an echo request to the database host | The application server | The host answers | The host does not answer | An answer removes "the network failed" |

**Both tests can be confounded.** Network topology and firewall rules change the result
(`SRE Ch. 12, p. 179`). Read Section 6.2.

**Neither test is complete on its own.** Run both. Each test removes one hypothesis. Together
they leave one.

---

## 6. The pitfalls of a test in production

The book names four of these (`SRE Ch. 12, p. 179–180`). The fifth follows from the same two
pages. Each pitfall has a signal that tells you it is present.

| Pitfall | Signal that it is present | Trap code |
|---|---|---|
| The test does not separate the alternatives | A pass and a failure both fit two open hypotheses | D-28 |
| A confounding factor changes the result | The test runs from a place that the real caller does not use | D-26 |
| The test changes what it measures | You enabled logging, added a processor, or cleared a cache | D-27 |
| You cannot run the test a second time | The first run changed state that the second run needs | D-27, D-29 |
| The result is suggestive and not definitive | The fault is a race or a deadlock | D-30 |

The trap codes are entries in `diagnostic-trap-catalog.md`.

### 6.1 A test that does not separate the alternatives

**An ideal test has mutually exclusive alternatives, so that it keeps one group of hypotheses
and removes another group** (`SRE Ch. 12, p. 179`). The book states that this is often
difficult to reach in practice. Keep the hedge. Do not claim a clean separation that you do
not have.

**Signal.** You write the pass sentence and the failure sentence, and both sentences fit
hypothesis H1 and hypothesis H2. Then the test removes nothing.

**Fix.** Change the test until one result removes a group. If you cannot, split the
hypothesis into two hypotheses that a test can separate.

### 6.2 A confounded test

An experiment can give a misleading result because of a confounding factor
(`SRE Ch. 12, p. 179`). The book's example is exact. A firewall rule permits access only from
one address. An echo request from your workstation to the database then fails, although the
same request from the application server would have succeeded (`SRE Ch. 12, p. 179`).

**Signal.** The command ran from a laptop, a bastion host, or a build agent. The real caller
is a server in a different network.

**Fix.** Run the test from the same vantage point as the real caller. Record the vantage
point beside the result. A black-box probe follows the same rule. Probe the user-facing
address, and probe again behind the load balancer, so you can locate the failing hop
(`SRE Ch. 10, p. 156`).

### 6.3 A test that changes what it measures

**An active test can have side effects that change a later test result**
(`SRE Ch. 12, p. 179`). The book gives two examples. More processors for a process make
operations faster and make a data race more likely. Verbose logging can make a latency fault
worse (`SRE Ch. 12, p. 179–180`).

**The question that you can no longer answer.** Does the fault grow on its own, or does it
grow because of the instrumentation (`SRE Ch. 12, p. 180`)?

**Signal.** Your incident record lists no change, and the error rate moved after you touched
the system.

**Fix.** Treat an active test as a change to the system. Record it before you run it. Reverse
it when the test ends. Read the latency distribution, not the mean, because instrumentation
moves the tail first (`SRE Ch. 6, p. 89–90`).

### 6.4 A test that you cannot run a second time

**Signal.** The first run consumed the state that the fault needed. The queue drained. The
cache warmed. You repaired the wrong record. A second run now measures a different system.

**Consequence.** You hold one observation and no way to repeat it. You cannot separate the
effect of the treatment from the recovery of the system.

**Fix.** Before you treat, capture the state that the treatment will destroy. Capture the
thread dump, the queue depth, the pool checkout count, and the memory use
(`Release It! Sec. 16.7, p. 259`). Then run the treatment on one node, not on the fleet.

**The state rule.** Real systems are path dependent. They must be in a specific state before
the fault appears (`SRE Ch. 12, p. 182`). A test that destroys that state destroys your
ability to repeat the fault.

### 6.5 A result that is suggestive and not definitive

**Some tests are not definitive. They are only suggestive**
(`SRE Ch. 12, p. 180`). It can be very difficult to make a race condition or a deadlock
happen in a timely and reproducible way. You may have to accept less certain evidence that
these are the causes (`SRE Ch. 12, p. 180`).

**Do not upgrade the claim.** Write "probable cause" and name the remaining doubt. Read the
proof strength table in `../SKILL.md`, Section 8.

### 6.6 A negative result that you discard

A negative result is an outcome in which the expected effect is absent
(`SRE Ch. 12, p. 180`). A test that removed a hypothesis is a result. It is not a wasted
test.

**Negative results are conclusive.** They tell you something certain about production, about
the design space, or about the performance limits of an existing system
(`SRE Ch. 12, p. 180`). Record every one. `05-negative-results-and-bias.md` owns this
subject and the bias traps that travel with it.

---

## 7. Treat: change the system in a controlled way

A treatment is a test. It carries a hypothesis, a pass sentence, and a failure sentence.

| Treatment | Signal that selects it | What a recovery proves | Risk |
|---|---|---|---|
| Add tasks to the job (`SRE Ch. 22, p. 339`) | Idle capacity exists in the cluster | The service lacked capacity | Low. It is insufficient in a death spiral. |
| Disable the health checks for a period (`SRE Ch. 22, p. 339`) | Tasks are killed while they start | The health check itself removed the capacity | Medium. A sick task now serves. |
| Restart the servers (`SRE Ch. 22, p. 340`) | A death spiral in garbage collection, blocked threads, or a deadlock | The fault lives in process state | High with a cold cache. It can amplify the fault. |
| Admit about 1% of the traffic (`SRE Ch. 22, p. 340`) | Every replica crash-loops | Overload drove the fault | High. It is the big hammer. |
| Stop the batch work (`SRE Ch. 22, p. 341`) | Batch work runs on the serving path | The batch load competed with serving | Low |
| Block one request shape (`SRE Ch. 22, p. 341`) | One request shape crashes the process | That request shape is the trigger | Low. It denies one caller. |
| Set the pool maximum to zero and recycle the pool (`Release It! Sec. 16.10, p. 262–263`) | Threads block on one dependency | Calls to that dependency drove the fault | Medium. The feature degrades. |
| Open a Circuit Breaker on the dependency (`Release It! Sec. 5.2, p. 115`) | One integration point is saturated | The dependency drove the fault | Medium. The feature degrades. |

**Three rules control this table.**

1. **Name the source of the fault before you restart a component.** Confirm that the restart
   does not only move the load (`SRE Ch. 22, p. 340`).
2. **Test the treatment on one node first.** Then apply it to the fleet
   (`Release It! Sec. 16.10, p. 262–263`).
3. **Read the verdict from the fleet, not from the one node.** One repaired node behind a load
   balancer receives all the traffic. It can fail for that reason alone
   (`Release It! Sec. 16.10, p. 263`).

**Confirm the degraded path before you degrade it.** The Release It! team asked the developers
what the code does when the pool returns nothing. The answer was a message to the user. The
team acted only after that answer (`Release It! Sec. 16.9, p. 262`).

**Read the four golden signals before and after each treatment.** They are latency, traffic,
errors, and saturation (`SRE Ch. 6, p. 88`). A treatment with no before-and-after reading
proves nothing.

---

## 8. War stories

**The pool that was the only throttle (`Release It!` Sec. 16.6–16.11, p. 258–264).** An
online retailer failed on Black Friday morning and lost about a million dollars an hour.
Rolling restarts failed. Thread dumps showed the front end's 3,000 request-handling threads
blocked on a connection pool with no timeout. The back end's 450 threads waited on an external
scheduling system that could serve about 25 concurrent requests and received about 90. The
team set the pool maximum and the checkout block time to zero on one server. Nothing changed,
because the maximum has an effect only when the pool starts. The team then called the stop and
the start methods on that pool component, and that one server began to serve. The team applied
the same treatment to every server after that proof. The monitor showed green about 90 seconds
later. The whole reconfiguration took less than five minutes. A configuration file change plus
a full restart would have taken more than six hours under that load.

**The correlation that was a coincidence (`SRE Ch. 12, p. 182–186`).** An App Engine customer
reported a latency rise of nearly an order of magnitude. Processor time and serving process
count nearly quadrupled. No code change and no traffic rise explained it. The App Engine
developers found a correlation between the latency rise and a rise in `merge_join` datastore
calls. That call often indicates suboptimal indexing. The team started to build composite
indices. Request tracing then showed that requests for static content were also much slower,
and static content never touches the datastore. That single observation exposed the
correlation as spurious and the indexing theory as flawed. Tracing also showed a gap of about
250 ms between the request start and the first remote call. The tracing system had no view
inside that gap, because it traces remote calls only. The team mitigated the fault. It moved
the application to a processor-rich instance type. The application developers then
instrumented the application themselves and found the real cause. An access-control defect
created and stored an object on every access to one path. A security scanner produced
thousands of these objects in half an hour. Every later request then checked all of them.

**The web server that handled 800 of 8,000 connections (`SRE Ch. 12, p. 180`).** A development
team rejected a particular web server. It handled only about 800 connections out of the 8,000
that the team needed, and it then failed because of lock contention. That is a negative
result, and the team recorded it. A later team that evaluated web servers did not start from
nothing. It read the recorded result and decided quickly whether it needed fewer than 800
connections, or whether somebody had repaired the lock contention
(`SRE Ch. 12, p. 180–181`).

---

## 9. Record what you tested

**Record clear notes of the ideas you had, the tests you ran, and the results you saw**
(`SRE Ch. 12, p. 180`). This matters most in a complex case that lasts a long time. It stops
you from repeating a step.

Use this table. It is Section 6 of the Fault Record in `../SKILL.md`.

| # | Hypothesis | Test | Compare or treat | Vantage point | Risk | Result |
|---|---|---|---|---|---|---|
| H1 | The database refuses connections | Connect with the application server credentials | Compare | `app-01` | low | removed |
| H2 | The network to the database failed | Send an echo request to the database host | Compare | `app-01` | low | kept |

**The reversal rule.** If you changed the system for a test, make the change in a systematic
and documented way. That lets you return the system to its state before the test, instead of
an unknown configuration (`SRE Ch. 12, p. 180`).

**Write the record during the incident, not after it.** A shared document gives the timestamps
that the postmortem needs. It also keeps other people informed without an interruption to the
responder (`SRE Ch. 12, p. 188, footnote 70`).

**A test that you repeat at every incident is toil** (`SRE Ch. 5, p. 75–76`). Build a
diagnostic tool for that service instead (`SRE Ch. 12, p. 178`). The tools and the methods
outlive the experiment (`SRE Ch. 12, p. 181`).

---

## 10. When to stop testing

| State | Next step |
|---|---|
| One hypothesis survives, and the fault reproduces | Go to Cure. Claim a proven cause. |
| One hypothesis survives, and no reproduction exists | Go to Cure. Claim a probable cause. Name the doubt. |
| Several factors hold, and no single factor is sufficient | Go to Cure. List the causal factors. |
| Every hypothesis is removed | Return to Examine. You need new observations. |
| The tests are too risky to run now | Mitigate. Then test in a nonproduction environment. |

**A nonproduction environment can reduce these limits.** It costs another copy of the system
(`SRE Ch. 12, p. 182`).

**One incident can have several root causes.** Repair each one (`SRE Ch. 6, p. 83`).

---

## 11. Checklist for step 5

1. List every open hypothesis. Give each one an identifier.
2. Write the pass sentence and the failure sentence for each hypothesis.
3. Reject any test whose two results fit the same two hypotheses.
4. Order the tests by likelihood, then by the risk that each test poses.
5. Choose compare or treat for each test. Prefer compare.
6. Name the vantage point of each test. Match it to the real caller.
7. Capture the state that a treatment will destroy, before you run the treatment.
8. Run a treatment on one node. Confirm the effect. Then apply it to the fleet.
9. Record every result, including every removed hypothesis.
10. Reverse every change that you made for a test.

---

## 12. Cross-references

| Subject | File |
|---|---|
| The six steps and the loop | `01-the-troubleshooting-model.md` |
| Mitigation before diagnosis, and evidence capture | `02-triage-first-diagnose-second.md` |
| How to produce the hypotheses that this file tests | `03-examine-and-diagnose.md` |
| Negative results, confirmation bias, and correlation | `05-negative-results-and-bias.md` |
| Telemetry that makes a test cheap to run | `06-making-troubleshooting-easier.md` |
| Trap codes D-26 to D-30 | `diagnostic-trap-catalog.md` |
| The Fault Record and the proof strength table | `../SKILL.md`, Section 8 |

**Hand-offs to other skills.**

- The test names a change to reverse. `reverse-branching/SKILL.md` owns the revert.
- The cause is a concurrency or replication anomaly. Continue in
  `data-systems-design/references/hazard-catalog.md`.
- The cause is a blocked call or a lost durable write on one machine. Continue in
  `systems-programming/references/failure-catalog.md`.
- The trigger is untrusted input or a privilege change. Hand the fault to
  `security-engineering/SKILL.md`. Do not diagnose an attack as load.
- **Modern:** an LLM coding agent can propose a cause. That output is a hypothesis. Every rule
  in this file applies to it without change.
