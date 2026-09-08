# Risk and the Error Budget

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 3 "Embracing Risk", p. 48–61. Supporting citations name their own chapter.

Read this file when you set a reliability target. Read it when the team argues about the
rate of releases. This file gives the target, the budget, and the policy that the budget
drives. It does not give the indicator specification. Read `02-slis-slos-and-slas.md` for
that.

---

## 1. One hundred percent is the wrong target

**Past a certain point, more reliability is worse for the service and for its users.**
The book states this as the opening claim of the chapter (SRE Ch. 3, p. 48). Extreme
reliability limits the speed of new features. It raises cost. It reduces the number of
features a team can afford.

**The user cannot perceive the top nines.** The user experience follows the least reliable
component in the path. A cellular network or a device dominates it. A user on a 99% reliable
smartphone cannot separate 99.99% from 99.999% (SRE Ch. 3, p. 48).

**100% is probably never the right reliability target** (SRE Ch. 3, p. 61). Keep the hedge.
The book writes "probably never", not "never". Ch. 1 names the exceptions. A pacemaker and
an anti-lock brake sit outside this rule (SRE Ch. 1, p. 26).

**The availability target is a business decision, not a technical one.** The business or the
product establishes it (SRE Ch. 1, p. 27). Where the business has not decided, say so and
stop. Do not invent a number.

**The target is both a minimum and a maximum** (SRE Ch. 3, p. 49). Exceed the target, but do
not exceed it by much. A large overshoot wastes chances to add features, to remove technical
debt, or to reduce operating cost.

**Signal that this rule applies:** a requirement states `100% uptime`, `zero downtime`, or
`it must never go down`. Entry L-01 in `slo-defect-catalog.md` covers the scan.

---

## 2. Managing risk costs money in two ways

Cost does not rise in a straight line as reliability rises. An incremental improvement in
reliability may cost 100 times more than the previous increment (SRE Ch. 3, p. 48). Keep the
modal. The book writes "may".

| Cost dimension | What it pays for | Source |
|---|---|---|
| The cost of redundant machine/compute resources | Equipment that permits a machine to leave service for maintenance. Storage for parity code blocks that hold a durability guarantee. | SRE Ch. 3, p. 48 |
| The opportunity cost | Engineers who build risk-reducing systems instead of features that the user sees | SRE Ch. 3, p. 49 |

**Risk is a continuum, not a switch.** The team places each service on that continuum with a
cost and benefit analysis. Search, Ads, Gmail, and Photos sit at different points. The goal
is explicit risk taking. Align the risk that a service takes with the risk that the business
accepts (SRE Ch. 3, p. 49).

---

## 3. Measuring service risk

A service failure has many effects. The book names user dissatisfaction, harm, loss of
trust, direct or indirect revenue loss, brand impact, and undesirable press coverage
(SRE Ch. 3, p. 49). Some of these effects are very hard to measure.

**Google reduces the problem to one tractable proxy — unplanned downtime**
(SRE Ch. 3, p. 49). The proxy stays consistent across many types of system. Unplanned
downtime is expressed in nines. Each additional nine is an order of magnitude improvement
toward 100% availability (SRE Ch. 3, p. 50). A scheduled maintenance window is planned
downtime. It is not unplanned downtime, and it does not enter this metric
(SRE Ch. 3, p. 53).

---

## 4. The two availability formulas

Choose one formula and record the choice. `03-availability-math.md` holds the conversion
table and the arithmetic. This section states which formula to select.

| Situation | Formula | Reason | Source |
|---|---|---|---|
| One instance. The system is wholly up or wholly down. | Time-based. Uptime divided by total time over the period. | Two states are the only states | SRE Ch. 3, p. 50 |
| A globally distributed service. Some part serves traffic at all times. | Aggregate. Successful requests divided by all requests, over a rolling window. | Fault isolation keeps the service at least partly up, so the time formula is usually not meaningful | SRE Ch. 3, p. 50 |
| A batch job, a pipeline, a storage system, or a transactional system | Aggregate. Records processed successfully divided by records received. | There is no continuous request stream | SRE Ch. 3, p. 51 |

**Worked number for the time formula.** A system with a 99.99% target can be down for up to
52.56 minutes in a year and still meet the target (SRE Ch. 3, p. 50). **Worked number for the
aggregate formula.** A system serves 2.5M requests in a day at a 99.99% daily target. It can
serve up to 250 errors and still meet the target for that day (SRE Ch. 3, p. 50).

**Not all requests are equal.** A failed registration request is not the same as a failed
background poll for new email. The book accepts the success rate across all requests as a
reasonable approximation of unplanned downtime, seen from the user
(SRE Ch. 3, p. 50). Keep that word. It is an approximation.

**Set the target per quarter. Track it weekly, or even daily** (SRE Ch. 3, p. 51). The short
tracking period is what lets the team locate and correct a meaningful deviation.

---

## 5. Risk tolerance of a consumer service

The reliability team works with the product owner. The two parties turn business goals into
explicit objectives (SRE Ch. 3, p. 51). A consumer service usually has a product team. That
team is the best source for the reliability requirement (SRE Ch. 3, p. 52).

**Where no product team exists, the engineers play that role, knowingly or unknowingly**
(SRE Ch. 3, p. 52). Name the person who decides. An unnamed decider is still a decider.

Ask these four questions (SRE Ch. 3, p. 52).

1. What level of availability is required?
2. Do different types of failure have different effects on the service?
3. How can the cost of the service locate it on the risk continuum?
4. What other service metrics are important?

For question 1, ask these five sub-questions (SRE Ch. 3, p. 52).

| Sub-question | What the answer changes |
|---|---|
| What level of service will the users expect? | The floor of the target |
| Does the service tie directly to revenue, ours or a customer's? | Whether the cost test of Section 7 applies |
| Is the service paid, or is it free? | The strength of the expectation |
| What level do competitors provide? | The market floor |
| Is the service for consumers, or for enterprises? | The blast radius of one outage |

**War story — Google Apps for Work and YouTube (SRE Ch. 3, p. 52–53).** Most users of Google
Apps for Work are enterprise users. Their employees use Gmail, Calendar, Drive, and Docs for
daily work. An outage for Google is therefore an outage for every enterprise that depends on
the service. For a typical Google Apps for Work service, Google might set an external
quarterly availability target of 99.9%. Google backs that target with a stronger internal
target and with a contract that stipulates penalties. YouTube gives the contrast. In 2006
YouTube served consumers and was still changing and growing fast. Google set a lower
availability target for YouTube, because rapid feature development mattered
correspondingly more. The story proves that the target follows the market position and the
business lifecycle, not the technology.

---

## 6. The shape of the failure changes the response

Two failures can produce the same absolute error count and a very different business impact
(SRE Ch. 3, p. 53). Ask which is worse for this service. A constant low rate of failures, or
an occasional full outage.

| Failure shape | Example | Required response | Source |
|---|---|---|---|
| Partial and cosmetic | Profile pictures do not render | Remediate quickly | SRE Ch. 3, p. 53 |
| Exposure of private data | One user sees another user's private contacts | Stop the service during the debugging phase and the cleanup phase | SRE Ch. 3, p. 53 |
| Regular scheduled outage | The Ads Frontend outside business hours | Schedule it in advance. Count it as planned downtime. | SRE Ch. 3, p. 53 |

**War story — profile pictures against private contacts (SRE Ch. 3, p. 53).** The book uses
one contact management application to hold both cases. Intermittent failures that stop a
profile picture from rendering give a poor user experience, and the reliability team corrects
them quickly. A failure that shows one user's private contacts to another user can undermine
basic user trust in a significant way. The prescribed response for the second case is
different in kind. Google stops the service entirely during the debugging phase and the
potential cleanup phase. The story proves that an error count alone cannot set the response.

**War story — the Ads Frontend maintenance window (SRE Ch. 3, p. 53).** Advertisers and
website publishers use the Ads Frontend to configure, run, and monitor advertising
campaigns. Most of that work happens during normal business hours. Google therefore decided
that occasional, regular, scheduled outages in the form of maintenance windows were
acceptable. Google counted those outages as planned downtime, not as unplanned downtime.
The story proves that a usage pattern can move some downtime out of the risk metric
completely.

**The book requires the schedule. This skill also requires the announcement.** The three words
the book uses are "occasional, regular, scheduled" (SRE Ch. 3, p. 53). It names no
announcement. A schedule that sits in a document nobody outside the team reads gives the user
no notice. Announce the window as well. Entry L-44 in
`slo-defect-catalog.md` covers the scan.

---

## 7. The cost of one more nine

**Signal to run this test:** somebody proposes a higher target, and the service ties to
revenue.

Ads can make this trade because a request success or a request failure translates directly
into revenue gained or lost (SRE Ch. 3, p. 54).

| Step | Question or calculation | Worked number from the book |
|---|---|---|
| 1 | What is the proposed improvement in the target? | 99.9% → 99.99% |
| 2 | What increase in availability does that give? | 0.09% |
| 3 | What revenue does the service carry? | $1M |
| 4 | Multiply step 2 by step 3. | $1M × 0.0009 = $900 |
| 5 | Compare the cost of the improvement against that value. | Invest below $900. Do not invest above $900. |

The five numbers above are the book's own example. It assumes each request has equal value
(SRE Ch. 3, p. 54). Your revenue figure is a local decision. The method is not. Nygard
prices the same decision from the operating side. Read
`05-gathering-availability-requirements.md` for that second worked case.

**Where reliability does not translate into revenue, use the background error rate of ISPs.**
Google measured the typical background rate between 0.01% and 1% (SRE Ch. 3, p. 54). Drive
the measured error rate below that band and the errors stay inside the noise of the user's
own internet connection. Measure from the user for this comparison to hold.

---

## 8. Other service metrics buy degrees of freedom

Knowing which metrics do not matter is as valuable as knowing which metrics do
(SRE Ch. 3, p. 55). A metric that nobody needs is a constraint you can trade.

**War story — AdWords against AdSense (SRE Ch. 3, p. 55).** Speed was a distinguishing
feature of Web Search at launch. When Google introduced AdWords, a key requirement was that
the ads must not slow the search experience. Every generation of AdWords systems treats that
requirement as an invariant. AdSense serves contextual ads into a third-party page, so its
latency goal is only to avoid slowing the render of that page. The specific target therefore
depends on the speed of the publisher's page. AdSense ads can serve hundreds of milliseconds
slower than AdWords ads. That looser requirement let Google consolidate serving into fewer
geographical locations and save substantial cost. The story proves that a named
insensitivity is a budget you can spend.

---

## 9. Risk tolerance of an infrastructure service

An infrastructure component has multiple clients by definition, and those clients often have
different needs (SRE Ch. 3, p. 55). **Do not engineer every infrastructure service to be
ultra-reliable.** That approach is usually far too expensive in practice, because
infrastructure also aggregates large amounts of resources (SRE Ch. 3, p. 56).

**The strategy is explicitly delineated levels of service** (SRE Ch. 3, p. 57). Partition the
infrastructure. Offer it at multiple independent levels. Externalize the cost difference to
the client. The client then selects the lowest-cost level that meets its need. The next table
gives the two Bigtable levels (SRE Ch. 3, p. 56).

| Client population | Desired queue state | Provisioning | Relative cost |
|---|---|---|---|
| Low-latency client | Request queues almost always empty | Slack capacity, reduced contention, more redundancy, stronger client isolation | The reference cost |
| Throughput client | Request queues never empty | Runs very hot, less redundancy | 10% to 50% of the low-latency cluster |

**War story — the two Bigtable client populations (SRE Ch. 3, p. 55–56).** Some consumer
services read from Bigtable inside the path of a user request. Those services need low
latency and high reliability. Other teams use Bigtable as a repository for offline analysis
and care about throughput. The low-latency user wants the request queues almost always
empty, because inefficient queuing is often a cause of high tail latency. The analysis user
wants the queues never empty, so the system never idles. Success for one population is
failure for the other. Google resolved the conflict by building two kinds of cluster instead
of one compromise. The story proves that a single objective across a shared component
serves neither client.

**War story — Google+ data placement (SRE Ch. 3, p. 57).** Google+ places data that is
critical to enforcing user privacy in a high-availability, globally consistent datastore,
such as a globally replicated SQL-like system. It places optional data that only improves
the user experience in a cheaper store that is less reliable, less fresh, and eventually
consistent. Google can run several classes of service on identical hardware and identical
software. It varies the quantity of resources, the degree of redundancy, the geographical
constraints, and the configuration of the infrastructure software. The story proves that a
visible price is what makes a client choose the correct level.

---

## 10. The edge cannot hide unreliability

**War story — Google frontend infrastructure (SRE Ch. 3, p. 57).** The frontend
infrastructure is the set of reverse proxy systems and load balancing systems near the edge
of the network. Those systems terminate the TCP connection from the user's browser. A
consumer service can often limit the visibility of unreliability in its backends. The edge
cannot. If a request never reaches the application service frontend server, the request is
lost. Google therefore engineers these systems to an extremely high level of reliability.
The story proves that the position of a component in the request path sets its risk
tolerance, and not only its function.

**The rule for your design.** Set a higher target for a component whose failure the system
cannot mask. Set a lower target for a component behind a fallback path.

---

## 11. Why the error budget exists

The product development team and the reliability team are evaluated on different metrics.
Product development is evaluated largely on product velocity. Reliability is evaluated on
the reliability of the service (SRE Ch. 3, p. 58). The two incentives point in opposite
directions. **Information asymmetry amplifies the tension** (SRE Ch. 3, p. 58). The
developers see the time and the effort inside a release. The reliability engineers see the
state of production.

The book names four typical tensions (SRE Ch. 3, p. 58).

| Tension | Too little gives | Too much gives |
|---|---|---|
| Software fault tolerance | A brittle, unusable product | A product nobody wants, that runs very stably |
| Testing | Outages, privacy data leaks, and press-worthy events | A lost market |
| Push frequency | Slow delivery | Risk on every push |
| Canary duration and size | An untested release at full traffic | A slow release cycle |

Canarying is the named practice of testing a new release on a small subset of a typical
workload (SRE Ch. 3, p. 59). `reverse-branching/references/03-progressive-rollout-and-canary.md`
owns the mechanism.

**An informal balance is not an answer.** A preexisting team usually has one. Nobody can
prove that balance is optimal, rather than a function of the negotiating skills of the
engineers involved (SRE Ch. 3, p. 59). **"Hope is not a strategy."** The book gives that line
as Google SRE's unofficial motto. It rejects decisions driven by politics, by fear, or by
hope (SRE Ch. 3, p. 59).

---

## 12. Forming the error budget

**Signal to run this procedure:** an objective exists, and no allowed failure amount exists.

The two teams jointly define a quarterly error budget from the service's objective. The
practice has four steps (SRE Ch. 3, p. 59).

1. Product Management defines an objective. The objective sets the expected uptime per
   quarter.
2. A neutral third party measures the actual uptime. That third party is the monitoring
   system.
3. The difference between the two numbers is the budget of remaining unreliability for the
   quarter.
4. New releases can be pushed while measured uptime stays above the objective.

**The arithmetic form.** Ch. 1 gives it in one line. The error budget is one minus the
availability target (SRE Ch. 1, p. 27). A 99.99% target leaves 0.01%.

**Worked example.** An objective of 99.999% of queries served successfully per quarter gives
a budget of a 0.001% failure rate. A problem that fails 0.0002% of the expected queries
spends 20% of the quarterly budget (SRE Ch. 3, p. 59).

**Step 2 is the load-bearing step.** The party that ships the change must not measure its own
budget. Entry L-17 in `slo-defect-catalog.md` treats that defect as a merge blocker.

---

## 13. Using the budget

**The budget is a common incentive.** It lets product development and reliability find the
balance between innovation and reliability together (SRE Ch. 3, p. 60).

| Budget state | Required action | Source |
|---|---|---|
| Objectives met, budget remains | Releases continue | SRE Ch. 3, p. 60 |
| Budget close to spent | Reduce the release rate, or revert the release | SRE Ch. 3, p. 60 |
| Violations spent the budget | Halt releases temporarily. Invest the freed resources in testing and in development. | SRE Ch. 3, p. 60 |
| The team cannot launch features at all | The team may elect to loosen the objective, which increases the budget | SRE Ch. 3, p. 60 |

**The on/off form is the crude form.** The book names it bang/bang control and states that
more subtle and effective approaches are available (SRE Ch. 3, p. 60, footnote 15). The named
alternatives are a reduced release rate and a revert while the budget is close to spent.

**A large budget permits more risk.** A nearly drained budget makes the product developers
themselves request more testing or a lower push rate. The book calls that outcome
self-policing (SRE Ch. 3, p. 60).

**Self-policing depends on one precondition.** The reliability team must hold the authority
to actually stop launches when the objective is broken (SRE Ch. 3, p. 60). A budget with no
stop authority is a report, not a control. Catalog entry L-21 covers it.

**A fault you did not cause still spends the budget.** A network outage or a datacenter
failure reduces the measured objective and spends the budget. The number of new pushes may
fall for the rest of the quarter. The whole team supports that reduction, because everyone
shares responsibility for uptime (SRE Ch. 3, p. 60).

**The budget prices an over-strict target.** It shows the cost of a high target in
inflexibility and in slow innovation (SRE Ch. 3, p. 60). Loosen the objective in writing when
that cost is too high. Do not ignore the gate.

---

## 14. Key insights of the chapter

The book closes Ch. 3 with three insights (SRE Ch. 3, p. 60–61).

1. Managing service reliability is largely managing risk, and managing risk can be costly.
2. 100% is probably never the right reliability target. Match the profile of the service to
   the risk that the business accepts.
3. An error budget aligns incentives and shows joint ownership. It makes the rate of
   releases decidable. It defuses discussions about outages with stakeholders. It lets
   several teams reach the same conclusion about production risk without rancor.

---

## 15. Where to read next

- Write the indicator and the objective — `02-slis-slos-and-slas.md`
- Convert the target into minutes, and compute a dependency ceiling — `03-availability-math.md`
- The team has no time to build anything — `04-the-toil-budget.md`
- Define "down" for one feature — `05-gathering-availability-requirements.md`
- Scan the record for named defects — `slo-defect-catalog.md`
- Perform the revert or the freeze — `reverse-branching/SKILL.md`
