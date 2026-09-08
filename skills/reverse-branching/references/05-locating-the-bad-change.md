# Locating the Bad Change

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 12 "Effective Troubleshooting", pp. 169–188. *Release It!* — Michael T. Nygard
(Pragmatic Bookshelf, 2007), Ch. 16 "Phenomenal Cosmic Powers, Itty-Bitty Living Space",
pp. 252–264. Supporting pages: SRE Ch. 6, p. 83. SRE Ch. 8, pp. 122–126. SRE Ch. 9, p. 135.
SRE Ch. 13, pp. 191–196. SRE Ch. 17, pp. 229–237. SRE Ch. 26, p. 421. SRE App. B, p. 572.
SRE App. D, pp. 579–584. Release It! Sec. 17.4, pp. 281–283.

Read this file when the service is faulty and nobody knows which change caused it. It gives
the search method. It does not give the reverse action. For the reverse action read
`01-the-reversibility-contract.md`. For the order of moves during the outage read
`04-change-induced-emergency.md`.

**This file answers one question. Which change caused this symptom?** It also states when the
correct answer is "no change caused this symptom".

**Ownership.** `production-troubleshooting` owns the six-step troubleshooting method of
SRE Ch. 12, the hypothesis rules, the negative-result rule, and the Chapter 12 and Chapter 16
war stories. This file owns the change-attribution half only. It names the change list, the
binary search over that list, and the rule that lets you report "no change". Read the sibling
for everything else.

---

## 1. The first question is "what changed"

**A working system stays working until a force acts on it.** SRE Ch. 12, p. 177 states that
systems have inertia. The force is usually a configuration change or a shift in the load.

**Ask "what touched it last" before you ask "why".** SRE Ch. 12, p. 177 names recent changes
as a productive place to start the search.

**Triage comes before the search.** SRE Ch. 12, p. 173 gives the ordering. Make the system
work as well as it can under the circumstances. Then find the cause. SRE App. B, p. 572
states the same rule for a release. Revert first. Diagnose second.

**A change is not only a commit.** SRE Ch. 6, p. 83 defines a push as any change to a
service's running software **or its configuration**. Release It! Ch. 16, pp. 260–261 widens
the set further. Two of four scheduling servers went down for planned maintenance, and
marketing ran a newspaper advertisement. Neither event appears in a commit list.

### The change list

| Change source | Where the record lives | Cost of a missing record |
|---|---|---|
| Application commit | The deploy record for the release | You cannot name a candidate set. R-10 |
| Configuration push | The configuration repository and the push log | The search reads code only, and finds nothing. SRE Ch. 8, p. 127 |
| Feature flag state | The flag service audit trail | The binary is identical on both sides of the fault |
| Schema migration | The migration table and the release notes | The old code reads the new data as incomplete. R-26 |
| Data backfill or delete | The job log and the row counts | The code is correct and the data is wrong. SRE Ch. 26, p. 421 |
| Capacity change | The fleet inventory and the maintenance calendar | Release It! Ch. 16, p. 260. Two servers left the pool |
| Dependency version | The build manifest for the artifact | Your code did not change. Your dependency did |
| Operator action | The incident timeline. SRE App. D, pp. 582–583 | You search for a deploy that never happened |
| Demand change | The marketing calendar | Release It! Ch. 16, p. 261. The load driver is outside engineering |

**Log every deploy and every configuration change, at every layer of the stack.** SRE Ch. 12,
p. 177 requires this record from the server binary down to the packages on each node.
`02-release-engineering.md` gives the artifact identity that makes the record readable.

**Annotate the error graph with the start and the end of each deploy.** SRE Ch. 12, p. 178
and Figure 12-2. A dashboard with no deploy marker cannot answer the first question. That
defect is R-41 in `rollback-hazard-catalog.md`.

---

## 2. The three checks that run first

Run these three checks before you start any search. Each one is cheap.

1. **Read the deploy log for the fault window.** Signal to run it: the metric shows a step
   change at a known time. SRE Ch. 12, p. 177.
2. **Compare the deploy markers to the start of the symptom.** Signal: a candidate deploy
   exists. A deploy that ended after the symptom started is not the cause.
3. **Read the change list of the release.** Signal: the deploy is the candidate, and it holds
   more than one change. SRE Ch. 8, p. 123 and p. 126 require a report of all changes in a
   release, archived beside the build artifacts.

**Stop after check 3 when the list holds one change.** The search is finished. Revert it.

**Continue to Section 4 when the list holds many changes.** The search is now a binary
search over the list.

---

## 3. When no change is in the window

**The search must be able to say "no change" as confidently as it says "this change".**
SRE Ch. 12, p. 184 gives the worked case. The App Engine team eliminated change as the
cause. The performance changed on a Saturday. No application push and no production push
were in flight. The most recent code push and configuration push had completed days before.

**Stop the change search when the window holds no change.** Then read the other four causes.

| Cause when no change is present | Signal | Where to read |
|---|---|---|
| A load shift | Traffic, session count, or request mix moved | Release It! Ch. 16, p. 258 |
| A latent defect plus an enabling condition | The code is old. The trigger is new | SRE App. D, p. 579 |
| Bad data, not bad code | The symptom follows one key or one record | SRE Ch. 26, p. 421 |
| An external party | A vendor, a certificate, or a partner file changed | Release It! Ch. 16, p. 260 |

**A latent defect needs a trigger.** SRE Ch. 33, pp. 557–558 states the form. A latent error
plus an enabling condition produces a fault. In the Shakespeare incident of SRE App. D,
p. 579, a file descriptor leak on the no-results path was old. The trigger was a traffic
increase of 88 times from one social post.

---

## 4. Binary search over the change list

**Binary search is the method when the candidate list is long.** SRE Ch. 12, p. 176 gives
bisection over the components of a system. Split the system in half. Examine the
communication paths between the two sides. Determine which half works. Repeat.

**SRE Ch. 17, p. 232 gives the same method over versions.** A practical test environment
selects branch points among the versions and the merges. Each branch point resolves the
maximum dependent uncertainty for the fewest iterations. When one area resolves into a
fault, you select more branch points.

**`git bisect` performs the second form. This command is Modern.** The books predate the tool. The method is
the book's. `08-agent-and-operator-shortcuts.md` holds the exact commands.

**The arithmetic favors the search.** A range of 40 changes needs about 6 tests. A linear
walk over the same range needs up to 40 tests.

### The four preconditions

| Precondition | The failure when it is absent | Source |
|---|---|---|
| The test gives the same answer every time | The search reports a random commit | SRE Ch. 17, p. 236 |
| Every revision in the range builds | A broken revision stops the search | SRE Ch. 17, p. 231 |
| The artifact rebuilds at the old revision | You cannot test the mid point at all | SRE Ch. 8, p. 122. R-06 |
| The range has one good end and one bad end | The search has no bounds | Bisection needs a working side and a failing side. SRE Ch. 12, p. 176 |

**Rebuild an old revision with the toolchain that the revision pins.** SRE Ch. 8, p. 122
versions the build tools by the source revision of the project. A rebuild with today's
compiler tests a different artifact. That defect is R-07.

### The manual search

Use this procedure when no bisect tool applies. It works for a configuration list, a data
job list, or a list of vendor versions.

1. Write the ordered list of candidate changes. One line per change.
2. Name the test. Write the exact command and the exact pass condition.
3. Confirm the test at the good end. It must pass.
4. Confirm the test at the bad end. It must fail.
5. Apply the first half of the list. Run the test.
6. Keep the half that fails. Discard the other half.
7. Repeat step 5 until one change remains.
8. Confirm the result. Remove that one change alone. The test must pass.

**Step 8 is not optional.** A search over a range with two independent faults returns one
commit that is not sufficient on its own. SRE Ch. 12, fn. 62, p. 187 names the case. Hickam's
dictum states that several common low-grade problems can explain the symptoms better than one
rare problem.

**Record the state before every test.** SRE Ch. 12, p. 180 requires systematic and documented
changes, so you can return the system to the state before the test. Without that record the
system reaches an unknown mixed configuration. That defect is R-14.

---

## 5. Search when the signal is not clean

Match the shape of the signal to the method. Read the caution before you start.

| Signal shape | Method | Caution |
|---|---|---|
| A step change at a known time | Read the deploy log for that window | Annotate the graph with the deploy start and end. SRE Ch. 12, pp. 177–178 |
| A step change, many candidates | Binary search over the change list | The test must give one answer. SRE Ch. 17, p. 236 |
| A probabilistic fault | Search on a rate over a fixed sample | Measure the flake rate first. SRE Ch. 17, pp. 236–237 |
| Each test costs hours | Reduce the set first, then search | Order the survivors by likelihood. SRE Ch. 12, p. 179 |
| Visible only under production load | Canary each candidate in turn | A canary is structured user acceptance. SRE Ch. 17, p. 229 |
| Corruption found weeks later | Search the log back to the retention limit | Low-grade deletion defects appear months later. SRE Ch. 26, p. 421 |
| Two metrics moved together | Design a test with exclusive outcomes | Correlation is not causation. SRE Ch. 12, p. 171 |
| No change in the window | Stop the change search | The cause is load, data, or a latent defect. SRE Ch. 12, p. 184 |

### The probabilistic case

**A flaky signal returns a wrong commit.** SRE Ch. 17, p. 236 defines a flaky test as one
whose repeated and identical runs change the result.

**Measure the accuracy that the search volume demands.** SRE Ch. 17, p. 237 gives the
arithmetic for one service. A comparison across 21,000 tests produces 42,000 results. To
reject only 1 patch in 100 good patches, each test must run correctly over 99.9999% of the
time.

**The number of runs per search step is a local decision.** Neither book gives a formula for
a production search. Both books give the input. Measure the failure rate at the known bad
end. Then choose a sample size that separates that rate from the rate at the known good end.

**Most defects scale with traffic.** SRE Ch. 17, p. 229 states that most bugs are of order
one. Their rate grows linearly with the amount of user traffic. For an order-one bug you can
convert logs of requests with unusual responses into new regression tests. That strategy does
not work for a higher order bug.

An automatic revert wired to a flaky signal reverts good changes and thrashes the fleet. That
defect is R-43.

### The production-only case

**Canary each candidate when the fault appears only under real traffic.** SRE Ch. 17, p. 229
warns that a canary is not a test. It is structured user acceptance. It exposes code to live
traffic that you cannot predict. It does not always catch a new fault.

**The rollout stages give a search surface at no cost.** SRE App. B, p. 572 requires staged
rollouts with an observation window per stage. The last clean stage bounds the candidate set.
`03-progressive-rollout-and-canary.md` holds the ladder.

---

## 6. Test design, for a reversal decision

`production-troubleshooting/references/04-test-and-treat.md` owns the six-step loop, the
hypothesis rules, and the negative-result rule. Three of those rules decide whether a reversal
is safe to run. Keep these three.

1. **An ideal test has mutually exclusive alternatives.** It confirms one group of hypotheses
   and eliminates another group. SRE Ch. 12, p. 179. Run this test before you reverse anything
   whose reverse action is expensive.
2. **Record the state before every test.** SRE Ch. 12, p. 180 requires systematic and
   documented changes, so you can return the system to the state before the test. Without that
   record the system reaches an unknown mixed configuration. That defect is R-14.
3. **A change that you eliminated is a result. Record it.** SRE Ch. 12, p. 180–182 calls a
   negative result conclusive. Line 5 of Section 10 carries it into the postmortem.

---

## 7. Correlation is not causation

`production-troubleshooting` owns this rule and the App Engine case behind it. SRE Ch. 12,
p. 171 and p. 184–185. Two consequences belong to a reversal decision.

**A revert chosen on a correlation reverses the wrong thing.** Before you revert on a
correlation, name the observation that the theory forbids. Then look for that observation. In
the App Engine case the theory forbade slow static content. The team found slow static
content. SRE Ch. 12, p. 185.

**Coincidence grows with the number of metrics.** SRE Ch. 12, p. 171 and fn. 64, p. 187. As a
system grows and monitoring adds metrics, a purely coincidental correlation becomes
inevitable.

That defect is R-42 in `rollback-hazard-catalog.md`.

---

## 8. Forty changes in one release train

**Signal: the deploy is the candidate, and it carries many unrelated changes.**

**Revert the whole release first. Search afterward.** SRE App. B, p. 572 puts the revert
before the diagnosis, to reduce the mean time to recovery. Continue the search on a branch,
away from production.

**A large batch is a defect, not a constraint.** SRE Ch. 9, p. 135 states the cost. Release
100 unrelated changes at the same time, and attributing a performance regression takes
considerable effort or extra instrumentation. Smaller batches let each change be understood
in isolation.

**Reduce the candidate set before you search.** Run these four steps in order.

1. Remove every change that cannot reach the failing path. Read the difference, not the
   commit message.
2. Keep every change that touches the code path, the configuration, or the data that the
   symptom names.
3. Order the survivors by likelihood. SRE Ch. 12, p. 179.
4. Run the binary search over the survivors only.

**A release built from a moving head has no candidate set.** SRE Ch. 8, pp. 123–124 requires
a branch at a specific revision. Never merge that branch back to the mainline. Fix on the
mainline, then cherry pick to the release branch. That defect is R-12.

**Cherry pick to test one change alone.** SRE Ch. 8, p. 122 rebuilds at the same revision as
the original build and adds only the specific changes. Each cherry pick request is separately
approved or rejected. SRE Ch. 8, p. 126.

**Eliminate a whole tier in one step when the response carries its own provenance.**
SRE Ch. 12, pp. 176–178 gives the mechanism. The `X-Request-Trace` header lists the backend
servers that answered the request. Its presence proved the request reached the backends, so
the frontend and the load balancers left the candidate set at once.

---

## 9. War stories

`production-troubleshooting` tells each of these cases in full. This table keeps only the
change-attribution lesson.

| Case | Source | Where the full account lives | Change-attribution lesson |
|---|---|---|---|
| The App Engine latency mystery | SRE Ch. 12, p. 182–186 | `production-troubleshooting/references/03-examine-and-diagnose.md` | The team eliminated change as the cause, because no push was in flight and the shift began on a Saturday. State absence with the same confidence as presence |
| Black Friday at the retailer | Release It! Ch. 16, p. 256–263 | `production-troubleshooting/references/03-examine-and-diagnose.md` | The change list that mattered held a maintenance window and a newspaper advertisement. Neither one lived in a repository |
| Voodoo operations | Release It! Sec. 17.4, p. 281–283 | `production-troubleshooting/references/05-negative-results-and-bias.md` | A remediation that nobody proved to be causal becomes lore. The team ran a weekly database failover for six months |
| The global crash-loop | SRE Ch. 13, p. 191–194 | `incident-response/references/02-emergency-response.md` | The push engineer named the change within five minutes, because he watched the chat channel. The book calls that luck. Build the deploy marker instead |

---

## 10. What the search must record

Write these seven lines when the search ends. They are the input to the postmortem and to
`verify`.

1. The symptom, and the exact time it started.
2. The window that the search covered, and its two ends.
3. The candidate set, and the reason each change entered it.
4. The test, its exact pass condition, and its measured flake rate.
5. The changes that the search eliminated. SRE Ch. 12, pp. 180–182.
6. The confirmed change, and the confirmation from step 8 of Section 4.
7. The result when the revert did not remove the symptom.

**A revert with no record is a defect.** R-44. SRE Ch. 15, p. 209 makes a release rollback a
postmortem trigger. Without line 7 the next responder repeats the same failed revert.

---

## 11. Cross-references

| You need | Read |
|---|---|
| The reverse action for the change you found | `01-the-reversibility-contract.md` |
| Artifact identity, the change list, cherry pick | `02-release-engineering.md` |
| The exposure ladder that bounds the candidate set | `03-progressive-rollout-and-canary.md` |
| The order of moves while the outage runs | `04-change-induced-emergency.md` |
| The search across a retention window | `06-data-and-schema-reversibility.md` |
| The exact bisect and checkpoint commands | `08-agent-and-operator-shortcuts.md` |
| The hazard codes named above | `rollback-hazard-catalog.md` |
| The six-step troubleshooting method and its war stories | `production-troubleshooting` |
| The command structure that runs above this search | `incident-response` |
| The deploy marker and the gating metric | `observability` |

Hazards that this file uses: R-06, R-07, R-09, R-10, R-12, R-14, R-26, R-41, R-42, R-43,
R-44.

---

## 12. Proportionality

| Situation | What this file requires |
|---|---|
| One change is in the window, and it is the candidate | Section 2 only. Revert it |
| The deploy holds 2 to 5 changes | Section 2, then step 1 and step 2 of Section 8 |
| The deploy holds many changes | The full binary search of Section 4 |
| The signal is probabilistic or slow | Section 5 before any search |
| No change is in the window | Section 3. Then stop |
| The symptom follows one row or one key | Nothing here. Read `06-data-and-schema-reversibility.md` |

**"Not applicable" is a valid result.** A local defect in code that never shipped needs no
search. A fault with a named stack trace and one owner needs no search. Use this file when
the candidate set is larger than one, or when the first revert did not remove the symptom.
