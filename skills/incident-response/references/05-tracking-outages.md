# Tracking Outages

**Sources:** *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy), Ch. 16 "Tracking
Outages", p. 216–221. Supporting pages: Ch. 6, p. 88. Ch. 11, p. 166–167. Ch. 15,
p. 208–212. Ch. 30, p. 505. Ch. 31, p. 514–516. App. B, p. 573–576.

A postmortem explains one outage. An outage tracker explains the shape of the year. This
file gives the tracker: what it receives, how it groups, how a team tags it, and what a
responder asks it.

**A raw count means nothing without a baseline.** The chapter opens on this rule. You improve
reliability only when you start from a known baseline and can track progress (SRE Ch. 16,
p. 216).

Read `04-postmortem-culture.md` first if you need the record of one event. Read this file
when you need the trend across many events.

---

## 1. Why a postmortem archive is not enough

**The postmortem archive holds only the large events.** A team writes a postmortem for an
incident with large impact. Issues that are individually small, frequent, and widespread do
not reach that bar (SRE Ch. 16, p. 216).

**The archive also reasons one service at a time.** A postmortem improves one service or one
set of services. It can miss a fix with a small effect per event and a large effect across
many events (SRE Ch. 16, p. 216, and footnote 84, p. 221).

**Consequence.** The team fixes the loud outages and never funds the cross-cutting work.
Catalog entry `N-36` names this coverage gap. See `incident-failure-catalog.md`.

**The tracker closes the gap because it receives every alert, not every postmortem.** It
answers questions that no single record answers (SRE Ch. 16, p. 216):

1. How many alerts does this team receive per on-call shift?
2. What is the ratio of actionable alerts to nonactionable alerts over the last quarter?
3. Which service that this team manages creates the most toil?

**Signal that you need a tracker.** A responder says "it happened again" and nobody can state
the count. That sentence is the trigger.

---

## 2. Escalator and Outalator

The book names two systems. They sit at two layers.

| | Escalator | Outalator |
|---|---|---|
| Layer | The single notification | The outage |
| Input | Every alert notification for SRE | Every alert that the monitoring systems send |
| Job | Track whether a person acknowledged receipt | Annotate, group, and analyze the alerts |
| Action on a timer | Escalate to the next destination after a configured interval | None. The tracker is passive. |
| Example escalation | Primary on-call, then secondary on-call | — |
| Page | SRE Ch. 16, p. 216 | SRE Ch. 16, p. 216–217 |

**Escalator escalates. It does not decide.** If no acknowledgment arrives after the
configured interval, the system escalates to the next configured destination (SRE Ch. 16,
p. 216). The book states no value for that interval. The interval is a local decision.

**Escalator succeeded because it changed nobody's behavior.** The team built it as a
transparent tool that received copies of the email sent to the on-call aliases. It joined the
existing workflow with no change to the user and no change to the monitoring system
(SRE Ch. 16, p. 216).

**Build the tracker on the pipeline that already exists.** The book repeats the same move for
Outalator. The team added features to infrastructure that was already in place
(SRE Ch. 16, p. 217).

**Modern.** Escalator and Outalator are Google-internal tools. The book names Slack, Hipchat,
and IRC as systems that a team can connect to a tracker of this shape (SRE Ch. 16, p. 218).
Any paging vendor or incident tool that you choose today is **Modern**. Do not attribute it
to the book. Judge it against the four duties in Section 3.

---

## 3. The four duties of the tracker

| Duty | The tracker must | Page |
|---|---|---|
| Receive | Accept every alert from the monitoring systems, passively | p. 216 |
| Store | Keep a copy of the original notification, and of every email reply | p. 217 |
| Group | Combine several alerts into one incident | p. 217 |
| Tag | Accept free-form tags at any level | p. 219 |

**The tracker shows several queues in one time-ordered list.** A responder reads one view
instead of switching between queues by hand (SRE Ch. 16, p. 217).

**The queue view matters because one team fronts many services.** A single SRE team is often
the primary contact for services whose secondary escalation targets differ. Those targets are
usually the developer teams (SRE Ch. 16, p. 217).

**Mark the useful annotations as important.** The interface then collapses the rest of the
message. The book names the case that this rule removes: a reply to all that adds names to
the copy list and nothing else (SRE Ch. 16, p. 217).

**An annotated incident carries more context than an email thread.** The thread fragments.
The record does not (SRE Ch. 16, p. 217).

---

## 4. Aggregation: one event, many alerts

**One event triggers many alerts. Accept that.** A network failure produces timeouts and
unreachable backend services for everyone. Every affected team receives its own alerts. The
owners of the backend services receive theirs. The network operations center receives its own
(SRE Ch. 16, p. 218–219).

**Even a single-service fault fires several alerts**, because the monitoring diagnoses several
error conditions (SRE Ch. 16, p. 219).

**You can reduce the count. You cannot reach one alert per event.** The book states that
multiple alerts are unavoidable in most trade-offs between false positives and false negatives
(SRE Ch. 16, p. 219). Ch. 11 still sets the target at one alert per incident
(SRE Ch. 11, p. 167). Hold both statements. Aim at 1:1. Do not expect it.

**Grouping is the answer to duplication. Email is not.** The book calls the ability to group
alerts into one incident critical (SRE Ch. 16, p. 219). One email that says "this is the same
thing as that other thing" works for one alert. It does not scale inside a team, between
teams, or across time (SRE Ch. 16, p. 219).

**Group three kinds of notification into one incident** (SRE Ch. 16, p. 217):

1. Alerts that describe the same incident.
2. Auditable events that are unrelated, such as privileged database access.
3. Spurious monitoring failures.

**Grouping gives you two separate numbers.** Count incidents per day. Count alerts per day.
Never merge them (SRE Ch. 16, p. 218). The ratio of the two numbers is the alert fan-out. It
is the number that Section 7 acts on.

Catalog entry `N-25` fails a tracker that has no grouping. See `incident-failure-catalog.md`.

---

## 5. Tagging

**Tagging is the most useful feature of the tracker.** The book calls it trivial in appearance
and probably the most useful unique feature of Outalator (SRE Ch. 16, p. 219–220).

**A tag is a free-form single word. A colon separates the levels.** The tracker reads the
colon as a semantic separator. That reading promotes a hierarchical namespace and permits
automatic treatment (SRE Ch. 16, p. 219).

| Namespace | Use | Book example |
|---|---|---|
| `cause:` | What produced the incident | `cause:network`, `cause:network:switch`, `cause:network:cable` |
| `action:` | What a responder did | Suggested prefix, p. 219 |
| `bug:` | The tracked follow-up | `bug:76543`, parsed into a link to the bug tracker |
| `customer:` | The affected counterparty | `customer:132456` |
| `bogus` | A false positive | `bogus`, widely used |

**Do not fix the tag list in advance.** The book states the opposite rule. A team that finds
its own preferences and standards produces a better tool and better data (SRE Ch. 16, p. 219).

**Accept the bad tags as a cost.** The book names two kinds. A typo such as `cause:netwrok`.
An unhelpful tag such as `problem-went-away` (SRE Ch. 16, p. 219). Neither justifies a fixed
list.

**Let the depth follow the team.** `cause:network` is enough for one team. Another team needs
`cause:network:switch` against `cause:network:cable` (SRE Ch. 16, p. 219).

**Generate the suggested prefixes from history.** The tracker suggests the prefixes that this
team already used. `customer:` appears for the teams that use it, and not for the others
(SRE Ch. 16, p. 219).

**Tag the non-incidents too.** A false positive is an alerting event and not an incident. So
is a test event. So is an email that a person sent to the wrong address. The tracker does not
tell these apart by itself. The tag does (SRE Ch. 16, p. 219).

---

## 6. Analysis: three layers

Enabling analysis is one of the most important functions of the tool (SRE Ch. 16, p. 220).
The book gives three layers in order.

| Layer | What it produces | The question it answers | Page |
|---|---|---|---|
| 1. Counting | Incidents per week, per month, per quarter. Alerts per incident. | How much? | p. 220 |
| 2. Comparison | The same counts across teams, across services, and across time | Is this load normal? | p. 220 |
| 3. Semantic analysis | The component behind the most incidents. Shared causes across teams. | What must we fix? | p. 220 |

**Layer 1 alone cannot decide anything.** A count with no baseline carries no meaning. The
book puts the sentence "That's the third time this week" against one question. Did the event
use to happen five times per day, or five times per month? (SRE Ch. 16, p. 220).

**Layer 2 is the layer that most teams skip, and it is easy to provide.** It lets a team judge
its own alert load against its own record and against other services (SRE Ch. 16, p. 220).

**Layer 3 needs the incident records next to the counts.** The book states the assumption
plainly. Semantic analysis works only when the tool presents this information alongside the
incident records (SRE Ch. 16, p. 220).

**Historical data also helps during a live incident.** The question "what did we do last time"
is always a good starting point (SRE Ch. 16, p. 220). It is a starting point and not an
answer. Read `01-incident-command.md` for the rule that a responder tests the old cause and
never assumes it.

---

## 7. Find the alert that always fires and never matters

This is the highest-value query in the tracker. Run it every quarter.

**Signal that triggers this procedure.** The on-call describes an alert as transient. Or the
tracker shows a high alert count and a low incident count for one rule.

1. **Query layer 1.** List every alert rule by firing count over the quarter.
2. **Divide the count by the incident count for the same rule.** A large ratio means fan-out,
   and Section 4 handles it. A large count with almost no incidents means noise.
3. **Read the tags.** Count the `bogus` tags. Count the alerts that carry no `action:` tag.
   An alert with many firings and no `action:` tag is the candidate.
4. **Compute the ratio of actionable alerts to nonactionable alerts.** The book names this
   question for the last quarter (SRE Ch. 16, p. 216).
5. **Send each candidate to `observability` for a disposition.** That skill owns the decision.
6. **Record the disposition as a tracked bug with a named owner.** A silent threshold change
   leaves no trail. Catalog entry `N-29` fails that change.

**Disposition.** `observability` owns the disposition of an alert rule. Its Table 1 chooses the
output. Its Table 13 chooses which rule to remove. This file does not restate those tables. It
supplies the counts that the disposition needs.

The production meeting asks two questions of every paging event. Should it have paged in that
way? Should it have paged at all? (SRE Ch. 31, p. 515). The nonpaging bucket holds three cases.
An issue that should have paged. An issue that needs attention and no page. An issue that needs
neither (SRE Ch. 31, p. 516). One event that produces several alerts needs grouping, not a
disposition (SRE Ch. 11, p. 167).

**Every paging alert must be actionable.** A paging alert must also align with a symptom that
threatens the SLO of the service (SRE Ch. 11, p. 166). An alert that maps to none of the four
golden signals needs a stated reason. `observability` owns those signals.

**A low-priority alert that fires every hour costs more than it looks.** It disrupts
productivity. The fatigue that it produces makes the team treat a serious alert with less
attention than that alert needs (SRE Ch. 11, p. 166). Catalog entry `N-24` names this.

**There are two options for an undiagnosed recurring alert. There is no third.** "Either
investigate such alerts fully, or fix the alerting rules." (SRE Ch. 30, p. 505). The book
lists this alert as an emergency waiting to happen. Catalog entry `N-28` names it.

**Never route the leftovers to email.** An alert that is worth interrupting a person must page,
or it must become a tracked bug. The book calls an email destination the moral equivalent of
`/dev/null` (SRE App. B, p. 574). Catalog entry `N-26` fails that destination.

**Silencing is a tool during an incident, not a disposition.** Silence a duplicate or
uninformative alert so the on-call can attend to the incident (SRE Ch. 11, p. 167). Then ask
for the disposition after the incident closes.

**The numeric targets that the book states.** Fewer than 5 daily tickets. Fewer than 2 paging
events per shift (SRE Ch. 11, p. 166). No more than 2 events per 12-hour on-call shift
(SRE App. B, p. 576). One alert per incident as the target ratio (SRE Ch. 11, p. 167). The
book states no threshold for "too noisy" as a firing count. That number is a local decision.

---

## 8. Reporting

**Two report paths exist. They serve two different readers.**

| Path | Reader | Content | Cadence | Page |
|---|---|---|---|---|
| Shift handoff email | The next on-call engineer, plus a copy list | The subjects, the tags, and the important annotations of the selected outages | Every shift change | p. 220–221 |
| Report mode | The team at the production review | The main list, with the important annotations expanded inline | Weekly for most teams | p. 221 |

**The handoff email selects zero or more outages.** The responder passes recent state to the
next shift (SRE Ch. 16, p. 220–221). Zero is a valid selection.

**Report mode gives a quick overview of the lowlights** (SRE Ch. 16, p. 221). The important
annotations from Section 3 are the input. An unmarked annotation stays collapsed.

**The production meeting is the place for the report.** Most SRE teams hold it weekly, and it
lasts 30 to 60 minutes (SRE Ch. 31, p. 514). Its agenda already holds an outages item and a
paging events item (SRE Ch. 31, p. 515).

**Alert the owner of the dependency when their alerting stayed silent.** The tracker makes that
gap visible across teams. See the war story in Section 10 (SRE Ch. 16, p. 221).

---

## 9. Counts that mislead

Read this section before you present any number from the tracker.

| Count | Why it misleads | Source |
|---|---|---|
| "Most incidents caused" per component | It can be an artifact of over-sensitive monitoring | Ch. 16, footnote 85, p. 221 |
| "Most incidents caused" per component | It can be a small set of client systems that misbehave, or that run outside the agreed service level | Ch. 16, footnote 85, p. 221 |
| Incident count alone | It indicates nothing about the difficulty of the fix, and nothing about the severity of the impact | Ch. 16, footnote 85, p. 221 |
| Cost against benefit, one incident at a time | It under-values a mitigation that applies across many events | Ch. 16, footnote 84, p. 221 |
| An alert count with no baseline | The same count can be good or bad | Ch. 16, p. 220 |

**A component that causes the most incidents is a starting point.** The book states that it is
a good place to start work on the alert count and the system. It also states the three
objections above (SRE Ch. 16, footnote 85, p. 221).

**A condition inside the SLO can still be a defect.** The alert conditions of two teams can sit
inside the nominal service level objective and still fail the higher expectations of the users
(SRE Ch. 16, p. 220). Read `03-on-call.md` for the error budget that this comparison consumes.

**The correct fix can be to reduce reliability.** The book names the introduction of more
artificial failures as a way to stop a service from over-performing (SRE Ch. 16, p. 220).

---

## 10. War stories

### The network failure that paged four teams
A network failure produces timeouts and unreachable backend services for every caller. Each
affected team receives its own alerts. The owners of the backend services receive their own
alerts. The network operations center hears its own klaxons. Nothing in this picture is a
monitoring defect. It is the unavoidable result of the trade-off between false positives and
false negatives. What it proves: the fix is grouping inside the tracker, and not a further
reduction of the alert rules (SRE Ch. 16, p. 218–219).

### The Bigtable team that nobody paged
A service suffers a disruption that looks like a Bigtable incident. The responder opens the
tracker and sees that the Bigtable SRE team received no alert. The right move is to alert that
team by hand. What it proves: a dependency fault is often invisible to the team that owns the
dependency. Only a shared cross-team view makes that gap detectable. The book states that
improved cross-team visibility does make a large difference to incident resolution, or at least
to incident mitigation (SRE Ch. 16, p. 221).

### Stale data and high latency, one cause
One team alerts on stale data. Another team alerts on high latency. Network congestion delays
database replication, and that single condition produces both symptoms. Each team debugs its
own symptom and never names the shared cause. What it proves: only semantic analysis across
the records of several teams joins two differently named symptoms to one systemic problem
(SRE Ch. 16, p. 220).

### The Bigtable change that pays only in aggregate
A particular change to Bigtable needs significant engineering effort and gives only a small
mitigating effect for one outage. Judged against that one outage, the change loses. Judged
across many events where the same mitigation applies, the effort may well be worthwhile. What
it proves: per-incident cost analysis systematically under-values a cross-cutting fix, and the
tracker is the instrument that reveals the aggregate (SRE Ch. 16, footnote 84, p. 221).

### The schema-change job with no witness
Some teams configure a dummy escalator destination. No person receives the notifications. The
notifications still appear in the tracker, where a responder tags, annotates, and reviews them.
One named use is a periodic job that may not be idempotent. The book's example is the automatic
application of schema changes from version control to a database. Another named use is a
technical audit of privileged account use and role account use. What it proves: the alert
pipeline doubles as a record of risky automation. A later incident can then be joined to the
exact automated run that preceded it (SRE Ch. 16, p. 221).

---

## 11. How to use this file in a review

Ask five questions of a team, a runbook, or an alert configuration.

1. **Does one system receive every alert?** If the answer is no, the team has no baseline, and
   `N-36` applies.
2. **Can the team state incidents per day and alerts per day as two numbers?** If the answer is
   no, the tracker has no grouping, and `N-25` applies.
3. **Does every alert rule carry a firing count and an action count for the last quarter?** If
   the answer is no, the team cannot run Section 7, and `N-24` and `N-28` stay invisible.
4. **Does a threshold change carry an owner and a bug number?** If the answer is no, `N-29`
   applies.
5. **Does the weekly production review read the tracker?** If the answer is no, the analysis
   exists and nobody acts on it.

**When this file does not apply.** A single live fault needs `01-incident-command.md`, not this
file. A single closed event needs `04-postmortem-culture.md`. A rotation design needs
`03-on-call.md`. A team that produces nothing needs `06-interrupts-and-overload.md`. "Not
applicable" is a valid result, and it is better than an invented trend.

**Related files.** `02-emergency-response.md` supplies the three emergency classes that the
tracker later counts. `incident-failure-catalog.md` holds the codes that this file cites.
They are `N-24`, `N-25`, `N-26`, `N-28`, `N-29`, and `N-36`. `../SKILL.md` Table 10 gives the
alert handling inside an open incident, and Table 12 gives the tag namespace. `observability`
owns every disposition of an alert rule.
