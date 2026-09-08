# Designing a System That Can Be Diagnosed

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 12, "Making Troubleshooting Easier", p. 186, with the case study at p. 183–186. Also
Ch. 6, p. 82–96, Ch. 10, p. 141–157, Ch. 13, p. 189–199. *Release It!* — Michael T. Nygard
(Pragmatic Bookshelf, 2007). Ch. 17, "Transparency", p. 265–309. Ch. 16, p. 252–264.

**Read this file when you write a design.** Read it again when you close a fault and you
list the telemetry that you did not have. Section 10 of the Fault Record feeds this file.

---

## 1. The rule that governs this file

**Telemetry that does not exist when the fault starts arrives too late to shorten that
fault.** In the App Engine case, the application developers added instrumentation only
after SRE mitigated the fault. The diagnosis then took days (`SRE Ch. 12, p. 185–186`).

**Transparency comes from deliberate design.** Nygard states that adding transparency late
in development is about as effective as adding quality. It can be done. It costs more effort
and more money than building it in from the start (`Release It! Sec. 17.2, p. 275`).

**A system with no transparency cannot survive long in production.** Administrators cannot
tune it. Developers cannot raise its reliability. Sponsors cannot see the revenue. The system
decays a little with each release (`Release It! Ch. 17, p. 266`).

**Signal that this file applies.** You write a design. Or a Fault Record names a missing
signal. Or one alert produced a third incident with no new evidence.

---

## 2. The two fundamentals

`SRE Ch. 12, p. 186` names the most fundamental ways to speed troubleshooting. Section 9
covers the change process, which the same page names as a fourth item.

| Fundamental | What it means | What it removes from the incident |
|---|---|---|
| **Observability built into each component from the ground up** | Every component exports white-box metrics and structured logs at the time it is written (`SRE Ch. 12, p. 186`, `p. 174`) | The step where a responder guesses what a process is doing |
| **Well-understood and observable interfaces between components** | A caller can read the state of the boundary that it crosses (`SRE Ch. 12, p. 186`) | The step where a responder cannot say which side of a boundary failed |
| **Information available in a consistent way** | One unique request identifier travels the whole span of calls (`SRE Ch. 12, p. 186`) | The step where a responder matches log entries by time and by guess |

**The third item is not a separate idea.** It makes the first two usable. Without one
identifier you cannot join an upstream entry to a downstream entry (`SRE Ch. 12, p. 186`).
See D-19 in `diagnostic-trap-catalog.md`. **Modern:** a trace context header that an
instrumentation library propagates performs the same job. The book names the identifier.

---

## 3. Why an opaque component forces you to guess

**A process on a server is opaque by nature.** Nygard states that a process reveals almost
nothing about itself unless a debugger is attached. It may work correctly. It may run on its
last thread. It may spin and do nothing. The first job of the design is to move information
across that process boundary. A black-box technology sits outside the process and reads what
is externally observable. Operations can add it after delivery. A white-box technology runs
inside the observed thing. Development must integrate it (`Release It! Sec. 17.3, p. 276`).

| Property | Opaque component | Transparent component |
|---|---|---|
| How you learn that it is slow | A user telephones you (`Release It! Ch. 17, p. 266`) | A latency histogram and a probe (`SRE Ch. 6, p. 88–90`) |
| How you locate the fault inside it | You restart it and you watch | You read the busy threads and the pool state (`Release It! Sec. 17.6, p. 297–298`) |
| How you separate it from its dependency | You cannot | You compare both sides of the boundary (`SRE Ch. 6, p. 87`) |
| What a diagnosis costs | Repeated restarts. This is toil. (`SRE Ch. 5, p. 75`) | One reading of the exposed state |
| What the fault teaches you | Nothing that you can record | A cause, plus the alert that catches the next one |

**Guessing is not a personal defect.** Guessing is what an opaque interface leaves a
responder. Nygard compares an opaque system to a sick goldfish. Nothing that you do helps, so
you wait (`Release It! Ch. 17, p. 266`). Treat a guess as a missing signal, not as a mistake.

---

## 4. Which perspective a responder reads

Nygard defines transparency as the qualities that let operators, developers, and sponsors
understand four things (`Release It! Ch. 17, p. 265`). They are historical trends, present
conditions, instantaneous state, and future projections. `observability` Section 6 owns the
four perspectives, the technology that serves each one, and the dashboard rules. Read it
there.

Two of the four decide whether a fault is diagnosable. This file states only those two.

- **Instantaneous behavior answers "what is it doing right now".** A responder reads it from
  monitoring, thread dumps, and logs (`Release It! Sec. 17.1, p. 273–275`). Without it, step 3
  of the loop has no input.
- **Present status answers "is this nominal".** Nygard defines nominal for a continuous metric
  as the mean for the time period plus or minus two standard deviations. The period that
  correlates best for traffic-driven metrics is the hour of the week (`Sec. 17.1, p. 271`).
  Without a stated normal, a responder cannot say that a reading is abnormal.

**A required event must appear by its absence.** A daily feed that did not arrive is a fault.
Nygard traces a startling number of business-level faults to batch jobs that failed invisibly
for 33 days (`Sec. 17.1, p. 272`). This is the one gap that no golden signal reports.

---

## 5. Telemetry that must exist before the fault

### 5.1 The four golden signals

`SRE Ch. 6, p. 88–89` names the four golden signals. They are latency, traffic, errors, and
saturation. `observability` Section 3 owns the definition of each one, the three error
classes, the failed-request latency rule, and the histogram boundaries. Do not restate that
table here. Item 4 and item 5 of the checklist in Section 11 require it.

**The one consequence that belongs to this file.** A mean hides a bimodal distribution, so a
responder reads a normal latency panel during an outage. A service with a 100 ms mean at
1,000 requests per second can easily have 1% of requests take 5 seconds
(`SRE Ch. 6, p. 89–90`). That is trap D-12.

### 5.2 What to expose inside the application

You are likely to guess the key metrics wrong. Even a correct guess decays, because the key
metrics change. Expose every state variable, counter, and metric
(`Release It! Sec. 17.6, p. 297`).

| Family | Items the book names | Why the responder needs it |
|---|---|---|
| Traffic indicators | Page requests total, page requests, transaction counts, concurrent sessions | Separates a demand change from a capacity change |
| Resource pool health | Enabled state, total resources, resources checked out, high-water mark, resources created, resources destroyed, times checked out, threads blocked for a resource, times a thread blocked | Names the pool that holds the blocked threads |
| Database connection health | Number of exceptions thrown, number of queries, average query response time | Separates the database from the network to it |
| Integration point health | State of the Circuit Breaker, its manual override, its failed call count, its last successful call time, its state transition count (`p. 271`), timeouts, requests, average response time, good responses, network errors, protocol errors, application errors, the actual address of the remote endpoint, concurrent requests, concurrent request high-water mark | Locates the vendor fault in one reading |
| Cache health | Items in cache, memory used, hit rate, items flushed by the garbage collector, configured upper limit, time spent creating items | Finds a cold cache and a cache that other nodes invalidate |

Source for the five families: `Release It! Sec. 17.6, p. 297–298`. Every counter carries an
implied time component. Read each one as "in the last n minutes" (`p. 298`).

**Present status adds the thread view** (`Release It! Sec. 17.1, p. 270–271`). For each thread
pool record the thread count, the threads busy, and the **threads busy more than five
seconds**. Record also the high-water mark, the low-water mark, the times a thread was
unavailable, and the request backlog. The five-second threshold is the one that the Black
Friday team read (`Sec. 16.6, p. 258`). Any other threshold is a local decision.

`observability` Section 7 owns the same expose lists as a design rule. This file states why
each family shortens a diagnosis.

### 5.3 The signals that the collection itself produces

A target that stops answering produces a gap in the time series. A gap fires no alert. See
D-17. Borgmon records four synthetic variables per target (`SRE Ch. 10, p. 144`).

1. Whether the name resolved to a host and a port.
2. Whether the target answered the collection.
3. Whether the target answered the health check.
4. What time the collection finished.

**Use the collection failure itself as a signal** (`SRE Ch. 10, p. 145`). The last variable
gives the freshness clock that a staleness alert needs. Prometheus, Riemann, Heka, and Bosun
are named as external systems with this design (`SRE Ch. 10, p. 142`).

### 5.4 Both sides, and the outside view

**Instrument both sides of every boundary.** You need how fast the caller believes the
dependency to be, and how fast the dependency believes itself to be. Without both you cannot
separate a slow dependency from a slow network (`SRE Ch. 6, p. 87`). See D-14.

**Add a black-box probe from the user vantage point.** Combine heavy white-box monitoring
with modest and critical black-box monitoring (`SRE Ch. 6, p. 87`). White-box monitoring
detects the faults that retries mask. Place a second probe behind the load balancer, so you
can isolate the failing edge (`SRE Ch. 10, p. 156`). The probe validates the response
contents, not only the status code (`SRE Ch. 10, p. 156`). It also stores the status, the
headers, and the body of a failed attempt. The Shakespeare prober did not, and the responder
paid one manual reproduction cycle for that (`SRE Ch. 12, p. 175–176`). See D-15 and D-20.
`observability` Section 5 owns the white-box and black-box split.

**A component can report healthy while the user sees a fault.** Nygard states that every
component can be up and the end user can still receive bad results. This happens with
blocked threads and with cascading failure (`Release It! Sec. 17.5, p. 287`). See D-16.

---

## 6. Interfaces that you can observe and operate

An interface that you can only read is half of the requirement. The Black Friday team
resolved that incident through the interface, not through a deploy.

| Requirement | Rule | Source |
|---|---|---|
| Component-level restart | Provide a control that restarts one component, not only the process or the host | `Release It! Sec. 16.11, p. 263–264` |
| A scriptable administration interface | One command must set a property, stop the service, and start it again | `Release It! Sec. 17.6, p. 296–297` |
| A runtime switch per integration point | An `enabled` property that a person sets to false without a deploy | `Release It! Sec. 16.9, p. 262` |
| Known degraded behavior | State in advance what the caller does when the dependency returns nothing | `Release It! Sec. 16.9, p. 262` |
| A logged state transition | Log every interesting state transition, even when you also send a notification | `Release It! Sec. 17.4, p. 283` |

**The restart rule is a named principle.** The ability to restart components, instead of
entire servers, is a key concept of recovery-oriented computing (`Sec. 16.10, p. 263`).
A dynamic reconfiguration plus a component restart took less than five minutes. A
configuration file change plus a full restart would have taken more than six hours under
that load (`Sec. 16.10, p. 263`). A graphical console is impractical at thirty or forty
servers (`Sec. 16.3, p. 255`). See D-36 and D-37.

**Modern:** a feature-flag platform provides the runtime switch. The book describes a
runtime `enabled` property on the component (`Release It! Sec. 16.9, p. 262`).

**The diagnosis tools must not depend on the failing service.** Keep a communication path
and a command-line path outside the serving path (`SRE Ch. 13, p. 192–194`). See D-34.

---

## 7. Logs that a person and a tool can both read

A log file is a human-computer interface (`Release It! Sec. 17.4, p. 280`).

`observability` Section 8 owns the log contract and the level rules. Four of its rules decide
whether a log is readable during an incident. Those four are listed here with the reason that
a responder feels.

| Rule | The cost during a fault | Source |
|---|---|---|
| Write one record per line, in space-padded columns, with a severity character | A two-line format defeats the reader and defeats `grep` | `Release It! Sec. 17.4, p. 280–281` |
| Add a message code field | Operations looks the code up in a run book | `Release It! Sec. 17.4, p. 278–279` |
| Carry a trace identifier on every line | You will read 10,000 lines after an outage and you need a search string | `Release It! Sec. 17.4, p. 283` |
| Remove any configuration that enables debug or trace | The real fault hides under method traces and checkpoint lines | `Release It! Sec. 17.4, p. 278` |

**Structured logs are the book's own recommendation.** `SRE Ch. 12, p. 174` names text logs
and structured binary logs, and `p. 186` requires structured logs in each component. Nygard
requires a build step that removes any configuration which enables a debug or a trace level
(`Sec. 17.4, p. 278`). **Modern:** shipping structured logs to a log platform serves the same
purpose. **Modern:** run that debug-level check against the container image in the pipeline.

**A dead process writes no log.** Nygard states that dead processes and hung processes both
write nothing (`Release It! Sec. 17.5, p. 283`). Logging alone cannot cover a hang. This is
why Section 5.4 requires an outside view.

---

## 8. Keep the policy outside the application

`observability` Section 9 owns the operations database, the expectation ladder, and the rules
that retire a signal. This section states only the part that a responder can change during a
fault.

**The monitoring system is an exoskeleton around the system, not a weave through it**
(`Release It! Sec. 17.2, p. 275`). Three decisions must live outside the application code.
Which metrics trigger an alert. Where the thresholds sit. How state variables combine into
one health status. They are policy decisions, and they change at a different rate than the
application code changes (`Sec. 17.2, p. 275`).

**Why this matters during a fault.** A threshold that lives outside the code is a threshold
that a responder can change without a deploy. Nygard's team met the same rule on the other
side. A configuration file change plus a full restart would have taken more than six hours,
and the runtime change took less than five minutes (`Sec. 16.10, p. 263`). This is the same
lever as the runtime `enabled` property in Section 6.

**The observation store must not endanger the service.** A fault in the operations database
must have no noticeable effect on the primary function (`Sec. 17.7, p. 302`). A responder who
cannot read the history has lost the answer to "is this normal". See D-24.

---

## 9. Simplify, control, and log every change

`SRE Ch. 12, p. 186` names the third fundamental. A change that misrepresents the state of
reality creates the need to troubleshoot. Simplify it. Control it. Log it.

| Layer | What the log must hold | Trap if it is absent |
|---|---|---|
| Code | The version identifier and the deploy start and end time | D-31, D-33 |
| Configuration | The value before, the value after, and the actor | D-31 |
| Infrastructure | The maintenance window of every dependency | D-31 |
| Demand | The marketing campaign and its start time | D-31 |

**Annotate the error graph with the deploy marks.** `SRE Ch. 12, p. 178` shows error rates
with deployment start and end times on the same graph. Google treats a change to a
dependency as a special case of a runtime configuration change (`SRE Ch. 7, p. 101`). Every
production change needs an audit trail. The Admin Server logs the requestor, the parameters,
and the result (`SRE Ch. 7, p. 113`).

---

## 10. War stories

**The whitelist cache (`SRE Ch. 12, p. 183–186`).** An App Engine customer reported latency
up nearly an order of magnitude, with no code change and no traffic rise. A correlation with
`merge_join` datastore calls sent the team toward composite indices. Tracing then showed that
static content, which never touches the datastore, was also slow, and that removed the
theory. Tracing also showed a window of about 250 ms between request start and the first
call, with nothing in it. The tracing system records only remote calls. SRE mitigated. The
developers then added their own instrumentation and found a whitelist check on every request,
backed by an in-memory cache. This proves that a component with no in-process instrumentation
is invisible to a remote-call tracing system.

**Black Friday (`Release It! Ch. 16, p. 256–263`, `Ch. 17, p. 265–266`).** A retail site went
down and lost orders at about a million dollars an hour. Rolling restarts failed. Thread
dumps showed 3,000 front-end threads blocked on a connection pool with no timeout. That pool
waited on an order management system. Its 450 threads waited on a scheduling system that
served about 25 concurrent requests and received about 90. Nygard states that the team relied
on component-level visibility, and that the visibility was no accident. This proves that the
telemetry which resolves an incident is telemetry that somebody designed earlier.

**The caches that invalidated each other (`Release It! Sec. 17.2, p. 275`).** Every display
of an item accidentally updated it, which sent a cache invalidation notice to every other
server. Per-server visibility hid the fault. As soon as the statistics of all the caches
appeared on one page, the fault was obvious. Without that page the team would have added
servers, and each new server would have made the fault worse. This proves that a scaling
effect is visible only in a view that aggregates across nodes.

**The nightly batch that met its goal (`Release It! Sec. 17.2, p. 275`).** A retailer ran a
project to make items appear on the site sooner. The batch jobs finished two hours earlier,
and the project met its goal. Items still did not appear until a long parallel process
finished at 5 or 6 a.m. Strictly local visibility leads to strictly local optimization. This
proves that a design must expose the outcome at the system boundary.

**"Reset required" (`Release It! Sec. 17.4, p. 281–283`).** An administrator received a page
and immediately started a database failover. The triggering line was a debug message about an
encrypted channel, and the application reset that channel by itself. Six months earlier that
line had happened to be the last thing logged before a database crash. The wording never said
who must reset, and the debug level was still enabled. The result was weekly database
failovers at peak hours for six months. This proves three rules. Remove debug configuration.
Name the actor in every message. Never let an unproven remedy become a run book entry.

**The report that nobody read (`Release It! Sec. 17.8, p. 305`).** A reporting system mailed
a report to a distribution list. Half the recipients had a mail rule that deleted it. The
report cost money, it created a false sense of security, and nobody read the one edition that
showed something serious. This proves that telemetry without a feedback process is worse than
no telemetry.

---

## 11. The build-time checklist, and how much of it each system needs

Run this list against a design, against a new service, and against a Fault Record Section 10.

1. Every component exports white-box metrics and structured logs (`SRE Ch. 12, p. 186`).
2. Every boundary is instrumented on both sides (`SRE Ch. 6, p. 87`).
3. One request identifier travels the whole span of calls (`SRE Ch. 12, p. 186`).
4. The four golden signals exist for every user-facing path (`SRE Ch. 6, p. 88–89`).
5. Latency is a histogram, and failed requests have their own counters (`SRE Ch. 6, p. 88–90`).
6. Each dependency exposes the integration point list of Section 5.2 (`Release It! Sec. 17.6, p. 298`).
7. Each pool exposes the resource pool list of Section 5.2 (`Release It! Sec. 17.6, p. 297`).
8. A black-box probe runs from the user vantage point and stores the failing body (`SRE Ch. 10, p. 156`).
9. A second probe runs behind the load balancer (`SRE Ch. 10, p. 156`).
10. The collector records the four synthetic variables per target (`SRE Ch. 10, p. 144`).
11. Every component has a restart control of its own (`Release It! Sec. 16.11, p. 263`).
12. Every integration point has a runtime switch and a known degraded path (`Release It! Sec. 16.9, p. 262`).
13. The administration interface is scriptable (`Release It! Sec. 17.6, p. 296–297`).
14. The build removes any configuration that enables debug or trace (`Release It! Sec. 17.4, p. 278`).
15. The change log covers code, configuration, infrastructure, and demand (`SRE Ch. 22, p. 335`).
16. Thresholds and the rules that combine state variables live outside the application (`Release It! Sec. 17.2, p. 275`).
17. The diagnosis tools do not depend on the service that they observe (`SRE Ch. 13, p. 192–194`).
18. A person reviews the data on a stated cadence (`Release It! Sec. 17.8, p. 307–308`).

**Item 18 is not optional.** Nygard states that the best data in the world cannot help when
nobody is looking (`Release It! Sec. 17.8, p. 305`).

### How much of the list each system needs

Match the telemetry to the exposure. "Not applicable" is a valid result for this file.

| System | Required before it serves users | You may defer |
|---|---|---|
| A script with no user and no production data | Nothing from this file | Everything |
| An internal tool with one team of users | Items 1, 4, and 14 | Probes, the OpsDB, the runtime switch |
| A user-facing service with no external dependency | Items 1 to 5, 8, 10, 11, 14, 15, 16 | The predictive perspective |
| A service that calls a vendor | Add items 6, 7, 12, and 13 | Nothing above |
| A service that moves money or holds records | The whole list. Add the audit trail of `SRE Ch. 7, p. 113`. | Nothing |

**The alert rule.** An alert that fires often needs a repair, not a louder pager. When the
repair is impossible, automate the response (`SRE Ch. 6, p. 93`). A service that spends its
error budget halts releases and invests in testing and development instead
(`SRE Ch. 3, p. 60`). `slo-engineering` owns that policy. `observability` owns the alert rule.

---

## 12. Cross-references

| Where to go | For what |
|---|---|
| `../SKILL.md`, Section 4 | The four examine questions, and the telemetry that answers each |
| `03-examine-and-diagnose.md` | How to read this telemetry during a fault |
| `01-the-troubleshooting-model.md` | The loop that this telemetry serves |
| `05-negative-results-and-bias.md` | Why a correlation needs a mechanism and a test |
| `diagnostic-trap-catalog.md`, groups C, D, F, and G | D-12 to D-25, D-31, D-33, D-34, D-36, D-37 |
| `self-healing-apis/SKILL.md` | Per-dependency telemetry, timeouts, the Circuit Breaker, bulkheads, and load shedding |
| `detail-planning/SKILL.md` | The observability specification. This file lists the requirement. That skill writes the specification. |
| `verify/SKILL.md` | The check that the specified telemetry exists in the code |
| `reverse-branching/SKILL.md` | The change log, the deploy marks, and the revert mechanism |
| `observability/SKILL.md` | The signal design, the alert rule, the log contract, and the gap scan |
| `capacity-engineering/SKILL.md` | The saturated resource, pool sizing, and the load test |

**One owner per overlap.** `observability` owns the design of the signal, the alert, and the
log. This file states only which of those signals a responder needed and could not read.
Every gap in Section 11 is a requirement that `observability` turns into a rule and
`detail-planning` turns into a specification. Where the two files repeat a book page, the
citation is the same and `observability` is the owner.
