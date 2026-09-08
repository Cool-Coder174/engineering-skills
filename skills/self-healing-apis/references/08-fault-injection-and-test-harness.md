# Proving It: Test Harness and Fault Injection

**Sources:** *Release It! Design and Deploy Production-Ready Software*, Michael T. Nygard
(Pragmatic Bookshelf, 2007), Sec. 5.7, pp. 136-140. *Site Reliability Engineering: How Google
Runs Production Systems*, Beyer, Jones, Petoff, Murphy (O'Reilly, 2016), Ch. 17, pp. 222–247,
with the failure tests from Ch. 22, pp. 336–341. Page numbers are the PDF page numbers of the
extracted sources.

You built a timeout, a retry, a circuit breaker, and a degraded mode. This reference answers
one question. How do you prove that any of them work?

---

## 1. The rule this reference exists to enforce

**An untested failure path is a broken failure path.** SRE states it directly. "the code path
you never use is the code path that (often) doesn't work" (SRE Ch. 22, p. 324). A degraded
mode runs rarely, so the team has little experience with it. That absence is itself a risk.

**Every system eventually operates outside its specification.** This is Nygard's own theme.
So test what your system does when the remote system misbehaves (Release It! Sec. 5.7, p. 136).

**A test that passes proves less than a test that fails.** A passing test does not prove
reliability. A failing test generally proves the absence of it (SRE Ch. 17, p. 223).

**Signal that you must read this file.** The change adds a fallback branch, a breaker, a
retry, a degraded mode, or an automated remediation. Catalog entry `I-40` in
`integration-fault-catalog.md` fires when no test reaches that branch.

---

## 2. What each test method can prove

Use this table before you argue about coverage. Each method answers a different question.

| Method | What it proves | What it cannot prove | Source |
|---|---|---|---|
| Unit test with a mock | The unit calls the interface correctly | Nothing about out-of-spec network behavior | Release It! Sec. 5.7, p. 138 |
| Integration test environment | The caller works when the dependency works | Nothing outside the dependency's specification | Release It! Sec. 5.7, p. 136 |
| Test harness | The caller survives faults no real vendor will produce on demand | Nothing about functional correctness | Release It! Sec. 5.7, pp. 137, 140 |
| Configuration test | How the live binary is actually configured | Nothing about the code | SRE Ch. 17, pp. 227–228 |
| Stress test | The load at which the component stops degrading and fails | Nothing about the recovery | SRE Ch. 17, p. 228 |
| Canary test | The behavior of new code under live traffic | Nothing with certainty. It misses faults. | SRE Ch. 17, pp. 228–229 |
| Production probe | That production and the release environment are equivalent | Nothing about unknown user data | SRE Ch. 17, pp. 243–244 |
| Load test to failure | The breaking point and the failure shape | Nothing about a fault the test did not produce | SRE Ch. 22, p. 337 |

**Read the table as a set, not as a ladder.** The harness supplements the other methods. It
does not replace unit tests, acceptance tests, or FIT tests (Release It! Sec. 5.7, p. 140).

---

## 3. Why a mock cannot produce the faults that matter

**A mock conforms to the interface. The faults that hurt you do not.** A mock supplies an
alternative implementation that the unit test controls. Some mocks throw exceptions on demand.
That covers only the failures the interface already models (Release It! Sec. 5.7, p. 138).

**A test harness runs as a separate server.** It is therefore "not obliged to conform to any
interface" (Release It! Sec. 5.7, p. 138). It can produce network errors, protocol errors,
and application-level errors. A mock cannot.

**Keep Nygard's hedge.** If every low-level error were guaranteed to be recognized, caught,
and thrown as the right exception type, you would not need a harness (Release It! Sec. 5.7,
p. 138). No vendor SDK gives that guarantee. Entry `I-32` covers the SDK that hides the socket.

---

## 4. Why the integration test environment cannot do it either

**Version locking.** For the strongest assurance you test against the dependency versions that
will be current at release. That constrains the whole company to one new piece of software at
a time (Release It! Sec. 5.7, p. 136).

**The environment becomes unitary.** Interdependencies produce one global environment that
shadows all of production. It needs change control as rigorous as production, or more rigorous
(Release It! Sec. 5.7, p. 136).

**It only tests in-spec behavior.** The environment can force a documented error code, and the
remote system stays inside its specification while it returns that code (p. 136). Integration
environments examine failures mainly in the seventh layer, and not even all of those
(Release It! Sec. 5.7, p. 139).

**War story. The set-theory proof and the unitary environment.** Nygard writes that he could
prove the limit with set theory. Testing against release-current dependency versions holds the
whole enterprise to one new piece of software at a time. The point survives the joke. The web of
interdependent systems collapses every team's integration environment into one shared
environment that mirrors production, and that environment then needs production-grade change
control. The harness escapes the trap, because each team runs its own copy in its own location
(Release It! Sec. 5.7, pp. 136–137).

---

## 5. The fault list a harness must be able to produce

Nygard lists the socket failures a web services call is exposed to (Release It! Sec. 5.7,
p. 137). Build the harness to produce all of them. Each row names the caller defense the
fault tests, and the catalog code that fires when that defense is absent.

| # | Fault the harness produces | Category | Caller defense it tests | Catalog |
|---|---|---|---|---|
| 1 | The harness refuses the connection | Network transport | Error handling. The breaker counts the failure. | `I-01` |
| 2 | The harness holds the connection in a listen queue and never answers | Network transport | The connect timeout | `I-01` |
| 3 | The harness replies with SYN/ACK and then sends no data | Network protocol | The read timeout and the deadline | `I-02` |
| 4 | The harness sends nothing except RESET packets | Network protocol | Error classification. Retry only on the right class. | `I-14` |
| 5 | The harness reports a full receive window and never drains the data | Network protocol | A bound on the blocking write | `I-02` |
| 6 | The harness establishes the connection and never sends one byte | Network protocol | The read timeout | `I-02` |
| 7 | The harness drops packets, which causes retransmit delays | Network transport | A deadline near the observed latency distribution | `I-33` |
| 8 | The harness never acknowledges a packet, which causes endless retransmits | Network transport | An application timeout, so the code does not inherit the OS timeout | `I-02` |
| 9 | The harness sends response headers and never sends the response body | Application protocol | A deadline on the whole call, not only on the connect | `I-02` |
| 10 | The harness sends one byte of the response every thirty seconds | Application protocol | A deadline on the whole call. An idle read timeout alone passes this fault. | `I-02` |
| 11 | The harness sends HTML where the caller expects XML | Application protocol | The status check, the content-type check, and the schema check | `I-30` |
| 12 | The harness sends megabytes where the caller expects kilobytes | Application logic | The maximum body size and the maximum result count | `I-29` |
| 13 | The harness refuses every authentication credential | Application logic | Permanent-error classification. The caller must not retry. | `I-14` |

**The four categories.** These faults fall into network transport problems, network protocol
problems, application protocol problems, and application logic problems. You can find a
failure mode in every layer of the seven-layer OSI model (Release It! Sec. 5.7, p. 139).

**Row 10 finds the most defects.** A caller with only a read timeout survives a hang and fails
this row. The bytes keep arriving, so the read timer keeps resetting. Read
`03-stability-patterns.md` for the deadline mechanics, and `01-integration-points.md` for the
socket behavior behind rows 1 to 8.

**Row 12 needs production-sized data.** Nygard pairs the harness with the Unbounded Result
Set antipattern (Release It! Sec. 5.7, p. 138). Read `02-stability-antipatterns.md`.

---

## 6. How to build the harness

**Signal to build one.** The integration point is critical, or the vendor SDK hides the
socket, or the degraded mode has no test that reaches it.

1. Write a server that substitutes for the remote end of the integration point.
2. Call the low-level network APIs directly. A real application would not do this. The
   harness may (Release It! Sec. 5.7, p. 139).
3. Produce faults in all four categories of Section 5, not only application errors.
4. Use the port number as the fault selector, so the harness needs no mode switch.
5. Send bytes too quickly and very slowly. Create very deep listen queues. Bind a socket and
   never service a connection attempt (Release It! Sec. 5.7, p. 139).
6. Log every request the harness receives.
7. Design the harness like an application server, with pluggable behavior. Subclass one
   framework for each application protocol, and for each perversion of it (Release It!
   Sec. 5.7, pp. 139–140).
8. Let developers call a shared harness from their workstations. Let them also run their own
   instances (Release It! Sec. 5.7, p. 139).

**Make the harness devious.** Nygard's instruction is that the harness should act like a
little hacker, and that it should leave scars on the system under test (Release It! Sec. 5.7,
pp. 137, 139). Its purpose is to make the caller cynical.

**War story. The killer harness.** Nygard ran one harness that listened on several ports. Port
10200 accepted connections and never replied. Port 10201 answered with content copied from
`/dev/random`. Port 10202 opened a connection and dropped it immediately. The port carried the
mode, so no test had to reconfigure the harness between runs. One harness broke many
applications across HTTP, RMI, and RPC. The same harness helped functional testing in the
development environment. Several developers could call it at the same time from their own
workstations (Release It! Sec. 5.7, p. 139). Choose your own port numbers. Nygard gives these
three as an example, not as a standard. The rule you keep is one port per fault mode.

**The harness logs requests.** A good harness is good at killing applications. Log the
requests, so a caller that dies with no trace can still be diagnosed (Release It! Sec. 5.7,
p. 139). Keep the record on the far side of the boundary.

**Never add simulated-failure switches to the application.** Nygard rejects this design and
gives the reason. Nobody wants to risk enabling a simulated failure after the system reaches
production (Release It! Sec. 5.7, p. 139). Put the fault in the harness, outside the
deployed code.

**Modern equivalents.** Recording proxies, protocol-level fault proxies, service-mesh fault
injection, and chaos engineering platforms are **Modern**. They implement the same pattern.
Neither book names them. Judge each one by the table in Section 5. A tool that cannot produce
rows 3, 5, 6, and 10 is not a harness.

---

## 7. The SRE testing hierarchy

SRE separates traditional tests, which evaluate correctness offline, from production tests,
which evaluate a deployed system (SRE Ch. 17, p. 224).

| Test | Class | What it evaluates | Source |
|---|---|---|---|
| Unit test | Traditional | One separable unit, a class or a function. It also serves as a specification. | SRE Ch. 17, p. 225 |
| Integration test | Traditional | An assembled component, with mocked dependencies by dependency injection | SRE Ch. 17, p. 225 |
| System test | Traditional | An undeployed system, end to end | SRE Ch. 17, p. 225 |
| Smoke test | Traditional | Simple critical behavior. It short-circuits costlier testing. | SRE Ch. 17, p. 225 |
| Performance test | Traditional | Resource use and latency across the lifecycle | SRE Ch. 17, pp. 225–226 |
| Regression test | Traditional | The gallery of bugs that already caused a failure once | SRE Ch. 17, p. 226 |
| Configuration test | Production | How a binary is really configured, against the checked-in file | SRE Ch. 17, pp. 227–228 |
| Stress test | Production | The limit where the component stops degrading and fails | SRE Ch. 17, p. 228 |
| Canary test | Production | New code under live traffic, on a subset of servers | SRE Ch. 17, pp. 228–229 |

**A zero-MTTR bug never reaches a user.** A system-level test applied to a subsystem can
detect the same problem that monitoring would detect. That test blocks the push. The bug then
has zero mean time to repair, and the users see a higher mean time between failures
(SRE Ch. 17, p. 223).

**Components do not degrade forever.** Past a certain point a component fails
catastrophically instead of degrading (SRE Ch. 17, p. 228). That is why stress tests exist.
Find the point on purpose. Read `06-cascading-failure.md`.

**Where to start when coverage is low.** Rank the components by importance. Test the
mission-critical and business-critical ones first. Then test the APIs that other teams
integrate against (SRE Ch. 17, p. 230).

**Flakiness has a numeric budget.** For a service with more than 21,000 simple tests, a patch
comparison uses 42,000 results. Users tolerate 1 rejection in 100 perfect patches. The
42,000th root of 0.99 requires each test to run correctly over 99.9999% of the time
(SRE Ch. 17, p. 237). Use the arithmetic, not a feeling, when you argue about a flaky test.

**A harness suite is a batch test.** It orchestrates several binaries, so it starts in seconds
and cannot give interactive feedback (SRE Ch. 17, pp. 236–237). Keep the fast tier free of it.

---

## 8. Production probes

**Split the request bank into three sets.** SRE prescribes known bad requests, known good
requests you can replay against production, and known good requests you cannot. Use each set
as an integration test, as a release test, and as a monitoring probe (SRE Ch. 17, p. 242).

**Those probes should never fail** (SRE Ch. 17, p. 243). A probe failure means the frontend
API or the backend API is not equivalent between production and the release environment.
Unless you already know why the two environments differ, the site is likely broken (p. 243).
Read `04-fault-localization.md` next, because a failed probe starts a localization, not a fix.

**The probe itself is a previously untested configuration.** The release test wraps the server
with a fake backend. The probe wraps the same binary with a real load balancer and a real
persistent backend. Those are not the same configuration (SRE Ch. 17, pp. 242-243).

**Test four version combinations, not one.** During an update, generate old probe against old
application, old probe against new application, new probe against old application, and new
probe against new application. Your monitoring must know every release version on both sides
of the interface (SRE Ch. 17, pp. 244–245).

**Two controls follow from that.** Inspect the probes inside the readiness check, so a bad
update fails safely and no user traffic reaches the new version (SRE Ch. 17, p. 244). Let the
rollout automation block the rollout until no problematic combination remains (SRE Ch. 17,
p. 245).

**Apply the same rule to a vendor.** Your client version and the vendor API version form the
same kind of pair. Record both in the Integration Record. Ask the vendor for a fake backend,
or build one from the harness. SRE names the fake backend as a build dependency that the peer
team maintains (SRE Ch. 17, pp. 244–245).

**Reporting is worth more than self-repair.** A tool that finds a discrepancy should report it
before an outage, and should avoid remediation of its own (SRE Ch. 17, p. 242). Read
`07-automated-remediation-safety.md`.

**War story. The half-parsed user list.** An edit to the user list file makes the parser stop
halfway. The machine keeps running. Recently created users are simply absent, and many users
notice nothing. A second tool, the one that maintains home directories, sees the mismatch
between the directories that exist and the directories the partial list implies. It reports
the discrepancy urgently. Its value is the report. SRE states that it should not try to fix
the problem itself, because the repair would delete a large amount of user data (SRE Ch. 17,
p. 242). This is defense in depth with the safe division of labor. One tool detects. A human
decides.

---

## 9. Canary tests

**A canary is not really a test.** SRE calls it structured user acceptance. A subset of
servers takes the new version and stays in an incubation period, which the book calls baking
the binary. If no unexpected variance appears, the rest of the servers follow. If something
goes wrong, the modified servers revert to a known good state (SRE Ch. 17, pp. 228–229).

**Keep the hedge.** The canary is ad hoc. It exposes code to unpredictable live traffic. It
is not perfect and does not always catch newly introduced faults (SRE Ch. 17, p. 229).

**The variance model.** SRE gives `CU = RK`. C is the cumulative count of reported variances.
R is the report rate. U is the order of the fault. K is the period over which traffic grows by
a factor of e, which is 172% (SRE Ch. 17, p. 229). After a revert, C and R estimate U.

| Order U | What the request did | How you catch it |
|---|---|---|
| 1 | The request met code that is simply broken | Convert logs of unusual responses into new regression tests (SRE Ch. 17, p. 229) |
| 2 | The request randomly damaged data a later request can read | The exponential rollout and the U estimate. Regression tests from logs do not work. |
| 3 | The damaged data is also a valid identifier for an earlier request | The same. Operational load grows quickly. |

Most bugs are order one and scale linearly with user traffic (SRE Ch. 17, p. 229).

**The rollout rule of thumb.** SRE's footnote gives one shape. Start at 0.1% of user traffic.
Then scale by orders of magnitude every 24 hours, and vary the geographic location of the
upgraded servers. Day 2 is 1%, day 3 is 10%, day 4 is 100% (SRE Ch. 17, p. 246, footnote 89).
A worked example puts K at about 10 hours and 25 minutes for continuous exponential growth
between 1% and 10% over 24 hours (SRE Ch. 17, p. 246, footnote 90).

**Any other percentage is a local decision.** The books give these numbers for these examples.
Record the number you choose and the reason in the Integration Record. A vendor cutover takes
the same shape. Send a small share of one operation to the new client, endpoint, or region.
Read `09-vendor-slas-and-degradation.md` before you promise a number for the new path.

---

## 10. Testing for cascading failure before it happens

SRE states the reason plainly. The specific way a service fails is very hard to predict from
first principles (SRE Ch. 22, p. 336). So you produce the failure on purpose.

| Test | The question it answers | Pass condition | Source |
|---|---|---|---|
| Load one component to its breaking point | Which resource ends first | The component sheds load or serves degraded results. It does not crash. | SRE Ch. 22, p. 337 |
| Gradual ramp, then impulse load | Does the cache change the answer | Both patterns give a known breaking point | SRE Ch. 22, p. 337 |
| Return to nominal load after overload | Can the degraded mode end with no human | The component leaves the degraded mode on its own | SRE Ch. 22, p. 337 |
| Load a stateful or caching service | Does concurrency corrupt state at the limit | State stays correct at high load | SRE Ch. 22, p. 337 |
| Make a noncritical backend unavailable | Does the frontend survive an error | No mass rejection and no resource exhaustion | SRE Ch. 22, pp. 338–339 |
| Blackhole a noncritical backend | Does the frontend survive silence | No mass rejection, no exhaustion, no very high latency | SRE Ch. 22, pp. 338–339 |
| Stage a failure with the largest client | Does the client amplify our outage | The client queues work and uses randomized exponential backoff | SRE Ch. 22, p. 338 |
| Reduce the task count in a small slice of production | Where the real limit is | You measure the limit and keep spare capacity for a manual failover | SRE Ch. 22, pp. 337–338 |

**Load test each component separately, and record each breaking point.** Components have
different breaking points, and you do not know which one arrives first. The recorded number
feeds capacity planning, regression testing, and the trade between utilization and safety
margin (SRE Ch. 22, p. 337).

**Blackholing is the vendor-outage test.** Drop the requests to a backend so it never answers
at all. This is the fault that finds a missing deadline, because an error returns and silence
does not (SRE Ch. 22, pp. 338–339). It is harness row 6 applied to a live dependency.

**Test popular clients, and remember that you are one.** Ask whether the client queues work
during an outage. Ask whether it uses randomized exponential backoff, and whether an external
trigger can make it create load (SRE Ch. 22, p. 338). Turn each question on your own client of
the vendor. Read `05-overload-and-load-shedding.md` for the retry budget.

**War story. The MapReduce and the slack CPU.** A team started a MapReduce job that consumed
a large amount of CPU across many machines. The aggregate slack CPU in the cell fell sharply
at once. Unrelated jobs then met CPU starvation. The lesson SRE records is a rule for load
tests. Stay within your committed resource limits while you test (SRE Ch. 22, p. 336). A load
test that leans on spare capacity measures a limit that will not exist during the incident.

**Exercise the degraded path on a schedule.** SRE gives one remedy for the unused code path. Run
a small subset of servers near overload on a regular basis. Alert when too many servers enter
the degraded mode (SRE Ch. 22, p. 324). Record the schedule in the Integration Record under "How
we exercise it".

---

## 11. Statistical tests and the replay rule

**Some tests are not repeatable.** SRE names fuzzing with Lemon, Netflix's Chaos Monkey, and
Jepsen for distributed state. Rerunning such a test after a fix does not definitively prove
that the fault is gone (SRE Ch. 17, pp. 235–236).

**So log what the test chose.** Record every randomly selected action. Sometimes the random
seed alone is enough (SRE Ch. 17, p. 236).

1. Refactor the log into a release test at once.
2. Run that test a few times before you file the bug report. The rate of non-failure on
   replay tells you how hard it will be to assert later that the fault is fixed.
3. Compare the variations in how the fault expresses itself. They name the suspicious code.
4. Escalate the severity of the report when a later run reveals a worse failure.

**Signal that you need one.** The integration holds state across calls. Read
`data-systems-design` for the correctness hazards behind that state.

---

## 12. The harness plan for one integration point

Fill this in per integration point. It feeds the "How we exercise it" line and the Failure
Model table of the Integration Record in `../SKILL.md`, Section 7.

| Failure class | Harness behavior | The caller must | Evidence to record |
|---|---|---|---|
| Connection refused | Refuse the connection | Fail fast and count the failure in the breaker | Breaker transition count, error class |
| Connection waits in the listen queue | Accept into a deep queue and never answer | End the wait at the connect timeout | Measured wait, timeout value |
| Connection accepted, no answer | Complete the handshake and send no byte | End the wait at the read timeout | Measured wait, thread state |
| Answer arrives slowly | Send one byte every thirty seconds | End the call at the deadline | Deadline value, latency record with timeouts counted |
| Answer is protocol garbage | Send HTML where XML is expected | Reject before the parser runs | Which check rejected it, and the logged body |
| Answer is a well formed error | Refuse every credential | Classify the error as permanent and not retry | Retry count, breaker state |

**Each row must produce evidence, not only a passing assertion.** `verify` reads this table,
and a row with no recorded number is not complete.

---

## 13. Proportionality

**A harness is not free. Match the effort to the exposure.**

| Situation | Required proof |
|---|---|
| The change makes no call to a party you do not operate | Nothing from this file |
| The change adds a call to an integration point that already has a harness | Add the new operation to the existing harness ports |
| The change adds a new integration point or a new vendor SDK | Rows 1, 3, 6, 10, 11, and 12 of Section 5, at least |
| The change adds a degraded mode or a fallback branch | A test that reaches the branch, plus a schedule that exercises it |
| The change adds an automated remediation | The full Section 12 table, plus a test of the kill switch |
| The integration moves money, or the call cannot be reversed | All thirteen rows of Section 5, plus a replayed write test |

**"Not applicable" is a valid result.** A change that adds a field to an internal handler
needs no harness. Say so, and stop.

**Do not build a separate harness for every integration point.** Nygard states that you do
not necessarily need one per point. One killer server on several ports produces the common
network faults for all of them (Release It! Sec. 5.7, p. 140). Add a dedicated harness only
for a protocol the shared one cannot speak.
