# Vendor SLAs, SLA Inversion and Degraded Modes

**Sources:** *Release It!* (Nygard, 2007), Sec. 4.10 "SLA Inversion", pp. 102–105, and Ch. 13
"Availability", Sec. 13.1, pp. 229–230, and Sec. 13.2, pp. 230–232. Also Sec. 4.5, p. 85,
Sec. 5.2, p. 117, Sec. 14.1, p. 243, Ch. 15, p. 250, and Ch. 16, pp. 256–263. *Site
Reliability Engineering* (Beyer, Jones, Petoff, Murphy, 2016), Ch. 4 "Service Level
Objectives", pp. 62–73. Also Ch. 3, pp. 48–61, Ch. 6, pp. 87–89, Ch. 7, p. 118, Ch. 21,
pp. 297–307, and Ch. 22, pp. 323–337.

This reference answers two questions. What availability may you promise for a feature that
calls a vendor? What does that feature return when the vendor does not answer?

**`slo-engineering` owns the reliability target.** It owns SLI, SLO, SLA, the error budget,
availability math, and the dependency ceiling. Read
`slo-engineering/references/03-availability-math.md` and
`slo-engineering/references/05-gathering-availability-requirements.md` first. This file
applies that arithmetic to one vendor and chooses the degraded mode.

**The two questions are one question.** The number you may promise depends on the answer you
serve when the vendor fails. Change the degraded mode and you change the arithmetic.

---

## 1. The arithmetic of SLA inversion

Nygard names the antipattern. A system that must meet a high availability SLA depends on
systems of lower availability. (Release It! Sec. 4.10, p. 104.)

**Your availability for a feature is the product of the availabilities in that feature.** A
single failure in one dependency fails the whole feature. That is why the terms multiply.
Nygard writes the formula as `P(feature up) = (1 - P(internal failure)) * P(dep 1 up) *
P(dep 2 up) * ...`. (Release It! Sec. 4.10, p. 103.)

**Five external services at 99.9% each cap the feature at 99.5%.** This is Nygard's own
worked number. (Release It! Sec. 4.10, p. 103.)

**The ceiling is the worst provider.** Unless every dependency is engineered for the SLA you
must provide, the best you can do is the SLA of the worst provider. (Release It! Sec. 4.10,
p. 103.)

**A dependency with no SLA removes your SLA.** Nygard states that a system with two such
dependencies cannot offer an availability SLA. (Release It! Sec. 4.10, p. 103.)

**Each dependency fails in three independent layers.** The layers are transport availability,
the naming service (DNS), and the application-level protocol. Any one layer can fail for any
one connection. (Release It! Sec. 4.10, p. 103.)

**Decoupling moves the lower bound.** A feature that is fully decoupled from external systems
has the bound `P(internal failure)`. Most features sit between the two bounds. (Release It!
Sec. 4.10, p. 104.)

**Service levels only decrease when you call a third party.** Nygard states this as a rule.
(Release It! Sec. 4.10, pp. 104–105.)

---

## 2. Count every dependency, not only the vendor

Nygard's dependency list for Project Frammitz is longer than the integration list the team
had drawn. Figure 4.15 records the numbers. (Release It! Sec. 4.10, p. 104.)

| Dependency | Stated availability |
|---|---|
| Frammitz, the promise it must meet | 99.99% |
| Corporate MTA | 99.999% |
| Message queues | 99.99% |
| Corporate DNS | 99.9% |
| Inventory | 99.9% |
| SpamCannon's applications | 99% |
| Partner 1's DNS | 99% |
| Message broker | 99% |
| SpamCannon's DNS | 98.5% |
| Partner 1's application | No SLA |
| Pricing and promotions | No SLA |

**Two of the eleven rows publish no number.** Those two rows decide the whole answer.

**Ask the infrastructure questions as well.** Nygard names the corporate DNS cluster, the
SMTP service, message queues and brokers, and the enterprise SAN. (Sec. 4.10, p. 105.)

**A firewall rule is an integration point.** Nygard treats the firewall rule set as an index
of the systems you call. Use it to find the dependencies nobody recorded. (Sec. 14.1, p. 243.)
`01-integration-points.md` holds the inventory record.

---

## 3. War story — Project Frammitz

Nygard describes a mission-critical website with redundancy at every level. The team built
redundant power, network, storage, server hardware, and applications, and chose a
shared-nothing architecture. The business required 99.99% availability, which allows slightly
more than four minutes of downtime each month. The site then called settlement, accounting,
fulfillment, inventory, fraud detection, a channel partner, geocoding, address verification,
and credit card authorization. The published numbers ran from 99.999% down to 98.5%, and two
dependencies published nothing. Nygard's verdict is that Frammitz can meet the promise only by
luck. The engineering was correct. The promise was arithmetic that does not hold.
(Release It! Sec. 4.10, pp. 102–105.)

---

## 4. Gather the availability requirement per feature

**Never gather the requirement for "the system".** Nygard gives the reason. When the system
uses stability patterns such as Circuit Breaker, the system as a whole can serve requests
while one feature does not work. (Release It! Sec. 13.2, p. 230.)

Run these five steps with the business sponsor. (Release It! Sec. 13.1, pp. 229–230.)

1. Name the feature. Do not name the system.
2. Convert the candidate percentage into downtime minutes per month.
3. Multiply the downtime by peak-hour revenue to get the worst-case loss.
4. Price the added implementation cost and operation cost across the system life span.
5. Compare the avoided loss with the added cost, and decide with the sponsor.

| Availability | Downtime allowed |
|---|---|
| 98% | 864 minutes per month (Release It! Sec. 13.1, p. 229) |
| 99.99% | 4 minutes per month (Release It! Sec. 13.1, p. 230) |
| 99.99% | at most 52.56 minutes per year (SRE Ch. 3, p. 50) |

**Each extra nine costs about ten times the implementation and about two times the annual
operation.** Nygard gives this as a rule of thumb. (Release It! Sec. 13.1, p. 230.)

**SRE prices the nine from the revenue side.** An improvement from 99.9% to 99.99% adds 0.09%
of availability. On a service that earns one million dollars, that increment is worth $900.
Invest only when the improvement costs less than $900. (SRE Ch. 3, p. 54.)

**When revenue does not translate, compare against the background error rate.** SRE measured
the typical background error rate of ISPs at 0.01% to 1%. An error rate below that band sits
inside the noise of the user's own connection. (SRE Ch. 3, p. 54.)

**The user cannot perceive the top nines.** A user on a 99% reliable smartphone cannot tell
99.99% from 99.999%. (SRE Ch. 3, p. 48.)

**100% is probably never the right reliability target.** (SRE Ch. 3, p. 61.)

---

## 5. What you may promise, per feature

| Feature class | External dependencies | The number you may promise |
|---|---|---|
| Feature with no external call | None | Your maximum internal number (Release It! Sec. 4.10, p. 105) |
| Feature with vendors that publish numbers | One or more, all numbered | At or below the product, degraded by your own failure rate (Release It! Sec. 4.10, p. 105) |
| Feature with one vendor that publishes nothing | One or more, one unnumbered | No number. State the degraded mode instead. (Release It! Sec. 4.10, p. 103) |
| Feature that a third party performs end to end | The vendor only | A pass-through of the vendor number, at best (Release It! Sec. 13.2, p. 231) |
| Feature that is decoupled and degrades | Any, all decoupled | Your internal number, plus a stated degraded behavior (Release It! Sec. 4.10, p. 104) |

**Write exclusions for loss of availability caused by external systems.** (Release It!
Ch. 15, p. 250.)

**The last row is the only row you can improve by engineering.** Decoupling is the remedy
Nygard lists first. Make the application work without the remote system. Degrade it in a way
you chose in advance. (Sec. 4.10, p. 104.) `03-stability-patterns.md` holds Circuit Breaker.

---

## 6. SLI, SLO and SLA are three different words

| Term | SRE definition | Who it binds | Cost of a miss |
|---|---|---|---|
| SLI | A defined quantitative measure of one aspect of the service (SRE Ch. 4, p. 62) | Nobody. It is a number. | None |
| SLO | A target value or a range of values for a level that an SLI measures (SRE Ch. 4, p. 63) | Your team | An internal decision |
| SLA | A contract with users that includes consequences for meeting or missing the objectives (SRE Ch. 4, p. 65) | Your company | The stated consequence |

**The test that separates the two.** Ask what happens when the objectives are not met. No
explicit consequence means you hold an SLO, not an SLA. (SRE Ch. 4, p. 65.)

Apply SRE's five lessons when you choose the target. (SRE Ch. 4, p. 71.)

1. Do not choose the target from current performance.
2. Keep the aggregation simple.
3. Avoid absolutes such as "always" and "infinite".
4. Keep as few objectives as possible.
5. Start with a loose target and tighten it later.

**Keep a safety margin.** Hold an internal objective tighter than the published one. The
margin gives you room to answer a chronic fault before a user sees it. (SRE Ch. 4,
pp. 72–73.)

**Be conservative in what you advertise.** The broader the audience for a number, the harder
it is to change that number later. (SRE Ch. 4, p. 73.)

---

## 7. Measure the vendor with your own SLI

**A vendor number is a claim. Your probe is a measurement.**

Nygard states the mechanism. An automated system checks the availability of a feature by
running synthetic transactions against it. A human with a mouse is not the mechanism. A help
desk report is not the mechanism. (Release It! Sec. 13.2, p. 231.)

**Give the probe a designated user ID.** The synthetic transaction emulates a real user. The
marker keeps it out of the production data. (Release It! Sec. 13.2, p. 231.)

**Run the probe from outside your data center.** Nygard prescribes a mock client elsewhere
that runs synthetic transactions on a schedule. The alarm condition is that the client cannot
process the transaction, whether or not the server process runs. (Release It! Sec. 4.5, p. 82.)

Nygard names three observations that an aggregate check calls healthy. (Release It!
Sec. 13.2, p. 231.)

| What the feature does | Why a weak check passes it | What your definition must fix |
|---|---|---|
| It answers after 27.5 minutes | A response arrived | The maximum response time for each step |
| It answers in 50 ms with an error for everyone | The response was fast | The codes or text patterns for success and for failure |
| It flaps, and it looks healthy at each check | The sample missed the fault | The probe frequency, and the number of locations |

**A latency metric that omits timed-out calls is wrong.** Requests that never complete never
enter the average. On Black Friday the average page latency looked acceptable while the site
served nothing. (Release It! Ch. 16, p. 259.)

**A retry inside the vendor client hides the vendor fault.** SRE prescribes white-box
monitoring for failures that retries mask. A black-box check alone will not see them. (SRE
Ch. 6, p. 87.)

**Page on the four golden signals.** They are latency, traffic, errors, and saturation. Page a
human when one signal is problematic, or, for saturation, nearly problematic. (SRE Ch. 6,
pp. 88–89.)

**Modern** — a vendor status page is a claim the vendor publishes about itself. Neither book
discusses status pages. Apply the book rule instead. Your own synthetic transaction decides
availability. `04-fault-localization.md` separates a fault in us from a fault in them.

---

## 8. The eight variables that define availability

Nygard lists eight questions. An availability definition that leaves one of them open is not
a definition. (Release It! Sec. 13.2, pp. 231–232.)

1. How often does the monitoring device run its synthetic transaction?
2. What is the maximum acceptable response time for each step of the transaction?
3. Which response codes or text patterns indicate success?
4. Which response codes or text patterns indicate failure?
5. How frequently should the synthetic transaction run?
6. From how many locations?
7. Where does the device record the data?
8. Which formula computes the percentage? Time, or the number of samples?

Add three items to the eight.

- Which devices monitor the feature, and how each device reports a problem. (Sec. 13.2, p. 231.)
- The exclusions for loss of availability caused by external systems. (Ch. 15, p. 250.)
- The designated user ID that marks the synthetic transaction. (Sec. 13.2, p. 231.)

---

## 9. Choose the degraded mode for each integration point

Choose one row for each integration point. Record the choice before the outage.
(Release It! Sec. 5.2, p. 117.)

| Degraded mode | Use it when | What the user sees | Build this first |
|---|---|---|---|
| Cached answer | The answer stays valid for a known period | An answer with a stated age | A cache with a size bound and an invalidation rule |
| Queued answer | The work can finish later | A confirmation now, and a result later | A durable queue, and a path for the late answer |
| Default answer | A safe constant exists | A conservative result | A written decision on the constant, approved by the sponsor |
| Partial answer | The feature is one part of a larger answer | The page or the record without that part | A renderer that tolerates the missing part |
| Refusal | No safe answer exists | An immediate, clear refusal | A message, and a documented error code |

**The cache row carries a trap.** SRE separates a latency cache from a capacity cache. The
service can serve the expected load with an empty latency cache. It cannot serve the expected
load with an empty capacity cache. A cold capacity cache turns a restart into an outage. Make
every new cache a latency cache, or engineer it well enough to serve as a capacity cache.
(SRE Ch. 22, p. 333.)

**The refusal row is a legitimate answer.** The business sponsor decides it, not the engineer.
Nygard frames the decision as a question. Should the site crash when it cannot check
availability for in-store pickup? (Release It! Sec. 4.5, p. 85.)

**SRE orders the responses.** Redirect the request when that is possible. Serve a degraded
result when that is necessary. Handle the resource error transparently when nothing else
remains. (SRE Ch. 21, p. 297.)

**SRE defines a degraded response.** It is less accurate, or it holds less data, and it is
cheaper to compute. Two examples are a search across a small share of the candidate set, and
a read from a local copy that may be stale. (SRE Ch. 21, p. 297.)

**A layer that cannot serve a request returns one of two things.** It returns an "overloaded,
do not retry" error, or a degraded response. The layer above holds the same two options.
(SRE Ch. 21, p. 307.) `05-overload-and-load-shedding.md` holds the retry rules.

---

## 10. Operate the degraded mode

SRE gives the operating rules for graceful degradation. (SRE Ch. 22, pp. 323–324.)

| Question | The decision | Source |
|---|---|---|
| Which signal triggers the mode? | Choose from CPU usage, latency, queue length, or thread count | SRE Ch. 22, p. 324 |
| Who triggers it? | Decide automatic or manual before the outage | SRE Ch. 22, p. 324 |
| How often should it trigger? | Not often. Frequent entry means a capacity planning fault or a load shift. | SRE Ch. 22, p. 324 |
| How do we stop it? | Build a switch that disables the mode quickly, and that tunes its parameters | SRE Ch. 22, p. 324 |
| How do we keep the path working? | Run a small set of servers near overload on a schedule | SRE Ch. 22, p. 324 |
| Can it exit without a human? | Answer this before you ship the mode | SRE Ch. 22, p. 337 |

**A degradation switch is an automated remediation.** It must be idempotent, rate limited,
observable, and reversible. Each property carries its own source. Idempotent and rate limited,
SRE Ch. 7, p. 118. Observable, SRE Ch. 7, p. 113. Reversible, SRE Ch. 13, pp. 190-191.
`07-automated-remediation-safety.md` holds the rest.

---

## 11. Record the decision before the outage

Write these fields for each integration point. The record belongs with the integration
inventory, not in an incident channel.

- The chosen degraded mode, from the table in Section 9.
- The business sponsor who approved it, and the date.
- The exact user-visible behavior, as text or as a described screen.
- The switch that enables the mode, and who may operate it.
- The method that exercises the mode, such as a harness port or a scheduled drill.
- The availability arithmetic for the feature, and the promise you publish.

**Record it at the start, not after the first incident.** A discussion about SLA definitions
one year after launch is probably a heated discussion after an incident. At that point it is
too late. Nygard calls the result blamestorming. (Release It! Sec. 13.2, p. 230.)

**A pre-agreed definition moves the argument to the data.** Nygard warns that paper is a thin
shield during an incident. The definition still helps. It keeps attention on the measurement
rather than on a person. (Release It! Sec. 13.2, p. 232.)

---

## 12. War story — Black Friday, and the mode nobody had recorded

An online retailer lost about one million dollars an hour on Black Friday morning. Thread
dumps showed 3,000 front-end request threads blocked on a connection pool that had no
timeout. The pool waited on an order management system whose 450 threads waited on an
external home-delivery scheduling service. That service handled about 25 concurrent requests
and received about 90. Two of its four servers were down for holiday maintenance, a third
misbehaved, and marketing had published a free-delivery offer. The responders rejected several
remedies, because nobody knew how the application behaved in the state each remedy would
create. They found an engineer who could confirm the behavior. The pool returned null, and the
code then showed a message that delivery scheduling was not available. The team set the pool
maximum to zero on one server, recycled the pool component, and then changed every server. The
site recovered in about ninety seconds. Through the rest of the weekend the team reduced the
maximum to one, not to zero, because zero disabled home delivery completely. The remedy was a
degraded mode. The team found it during the outage instead of choosing it before.
(Release It! Ch. 16, pp. 256–263.)

---

## 13. War story — the Global Chubby planned outage

Chubby is Google's lock service. The global instance failed so rarely that service owners
added dependencies on it and assumed it would never fail. Its high reliability produced a
false sense of security, and its rare failures then produced user-visible outages across many
services. SRE's remedy is to make global Chubby meet its objective and not exceed it by much.
In a quarter where real failures have not consumed the budget, SRE takes the system down on
purpose. That outage exposes an unreasonable dependency soon after somebody adds it.
(SRE Ch. 4, pp. 64–65.)

**The lesson for a vendor integration.** A vendor that has never failed in your experience is
a vendor whose degraded mode you have never run. Treat the absence of past failures as a gap
in evidence, not as a low probability.

---

## 14. Exercise the degraded path

**The code path you never run is the code path that often does not work.**

| Signal that you must exercise the path | The action |
|---|---|
| A new integration point ships | Build a test harness for it, and produce the six failure classes. `08-fault-injection-and-test-harness.md` |
| A degraded branch has no test that reaches it | Add the harness port, then the test |
| The vendor has not failed for a long period | Schedule a controlled outage of the dependency (SRE Ch. 4, pp. 64–65) |
| The degraded path serves no traffic | Send a small share of traffic through it on a schedule (SRE Ch. 22, p. 324) |
| Too many callers enter the degraded mode | Alert on the rate of entry, and treat it as a capacity signal (SRE Ch. 22, p. 324) |

**A mock cannot produce these faults.** A mock returns what you told it to return. A test
harness produces an accepted connection that never answers, a slow answer, and a half-open
socket. (Release It! Sec. 5.7, p. 138.)

---

## 15. The fault codes this reference covers

| Code | Trigger to check it | Where the rule lives here |
|---|---|---|
| I-38 | You wrote an availability number for a feature that calls a vendor | Sections 1, 2 and 5 |
| I-39 | An integration point has no recorded degraded mode | Sections 9 and 11 |
| I-40 | A degraded branch has no test and no drill | Section 14 |
| I-41 | An outbound host or SDK has no entry in the inventory | Section 2 |

`integration-fault-catalog.md` holds the full entries.

---

## 16. When this reference does not apply

State "not applicable" when it is true. That answer is better than an invented finding.

- The change touches no outbound call and no availability promise.
- The change is internal to one process and calls no vendor.
- The feature already has a recorded degraded mode, and the change does not alter it.
- The task asks which component to build. Send that question to `system-design`.
- The task asks whether a retry of a write is safe. Send it to `data-systems-design`.

**One proportionality rule.** A prototype that no user depends on needs the arithmetic and one
sentence for the degraded mode. It does not need the eight variables, the drill schedule, or
a signed approval. Write the full record when the feature carries a promise to a user.
