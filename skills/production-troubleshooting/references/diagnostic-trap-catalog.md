# Diagnostic Trap Catalog

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 6, Ch. 7, Ch. 10, Ch. 11, Ch. 12, Ch. 13, Ch. 22. *Release It! Design and Deploy
Production-Ready Software* — Michael T. Nygard (Pragmatic Bookshelf, 2007). Ch. 7, Ch. 9,
Ch. 16, Ch. 17.

A catalog of named traps that lengthen a production fault, or that send a diagnosis to a
wrong cause. Each trap is a defect in the order of work, in the reasoning, in the telemetry,
or in the log. The other traps are defects in the test, in the tools, or in the record. None
of them is a defect in a person.

Each entry has three parts:

- **Signature** — what the process, the dashboard, the log, or the ticket looks like. This
  is the text that you search for.
- **Consequence** — what goes wrong, and when.
- **Fix** — the required remedy.

**How to use it.** `production-troubleshooting` runs the catalog at gate G6, before it
accepts a cause. `code-review` runs it as a diagnosability pass on a difference. `verify`
runs it against the observability that a phase specified. `planner` and `detail-planning`
run it against a design. Report only the traps that apply. "Not applicable. This service has
a change log, a latency histogram, and an external probe." is a valid result. That result is
better than an invented finding.

**What this catalog is not.** It names defects that lengthen a diagnosis. `observability`
owns the signal-gap catalog with the prefix `M-`. `incident-response` owns the
incident-failure catalog with the prefix `N-`. `self-healing-apis` owns the
integration-fault catalog with the prefix `I-`. A defect that belongs to one of those goes
there, not here.

**Severity.** 🔴 causes data loss, corruption, or an unrecoverable state. It blocks the
merge. 🟡 causes an outage or a wrong result under load. 🔵 is a risk to operation or to
maintenance.

**Every claim names a chapter and a page, or it carries Modern.** Neither book contains a
container orchestrator, a service mesh, an instrumentation library, or a continuous
profiler. Never attribute a modern practice to a book.

---

## A. Traps in the order of work

### 🟡 D-01 — Diagnosis before mitigation
**Signature:** the incident timeline shows the first mitigation action after the first five
investigation actions. The error rate stays at its peak across those actions. The runbook
for the alert holds no mitigation step above its diagnosis steps.
**Consequence:** the system stays down while the responder reads logs. A correct diagnosis
helps no user during that time.
**Fix:** make the system serve as well as it can. Then find the cause. Put one mitigation
step at the top of every paging runbook. `SRE Ch. 12, p. 173`.

### 🔴 D-02 — A mitigation that erases the evidence
**Signature:** the runbook orders a restart, a redeploy, or a cache flush. No step above it
captures a thread dump, a memory sample, or a copy of the log.
**Consequence:** the fault stops and the cause becomes unknowable. The same fault returns,
and the team starts again from nothing.
**Fix:** put the capture step above the destructive step. Rapid triage does not remove the
duty to preserve the evidence. Both must happen (`SRE Ch. 12, p. 173`). The thread dump is
the evidence that a restart destroys (`Release It! Sec. 16.7, p. 259`).

### 🟡 D-03 — A restart before the source is named
**Signature:** the timeline shows more than two restarts of the same component, with no
localization step between them.
**Consequence:** the restart moves the load instead of removing the fault. A restart against
a cold cache makes the fault worse, because every request then costs a miss.
**Fix:** name the source of the fault first. Confirm that the restart does not only move the
load. Canary the restart and run it slowly (`SRE Ch. 22, p. 340`). Note the cases where a
restart is correct. They are a death spiral in garbage collection, a deadlock, and in-flight
requests that hold threads with no deadline (`SRE Ch. 22, p. 340`).

### 🟡 D-04 — A fleet-wide remedy with no single-node test
**Signature:** the remediation command carries an "all hosts" or "all instances" flag on its
first run.
**Consequence:** a remedy that does not work removes the healthy capacity that remains. The
team cannot then tell whether the remedy failed or the diagnosis failed.
**Fix:** apply the remedy to one node. Confirm the effect. Then apply it to the fleet. Read
the verdict from the fleet. One repaired node behind a load balancer receives all the
traffic, and it can fail for that reason alone (`Release It! Sec. 16.10, p. 262–263`).
`SRE Ch. 22, p. 340`.

### 🔴 D-05 — A corrupting fault left running
**Signature:** a fault that writes wrong values or wrong permissions continues while the
team investigates. The runbook holds no stop switch.
**Consequence:** the volume of corrupt data grows for the length of the investigation. The
cost of recovery grows with it.
**Fix:** freeze the system when the fault can cause data corruption that you cannot recover.
A frozen system is better than a system that corrupts more data. `SRE Ch. 12, p. 173`.

---

## B. Traps in reasoning

### 🟡 D-06 — A correlation adopted as the cause
**Signature:** the incident document names a metric that correlates with the fault. A fix is
already in progress. No test separates the two metrics.
**Consequence:** the team builds the wrong fix. The App Engine team started to build
composite indices for a datastore call that had no causal role. A trace of static content,
which never touches the datastore, showed the same slowness and removed the theory.
**Fix:** state the mechanism that connects the two metrics. Then run a test that a
coincidence would fail. Expect coincidental correlation to grow as the metric count grows.
`SRE Ch. 12, p. 171, p. 184–185`.

### 🟡 D-07 — Confirmation bias on a repeat alert
**Signature:** the same alert name fired three or more times in one week. The newest
incident document reuses the previous cause and adds no new evidence.
**Consequence:** the responder spends the incident on a line of reasoning that was wrong
from the start.
**Fix:** state the assumption. Then test it. Treat the previous cause as one hypothesis
among several (`SRE Ch. 11, p. 165`). Stress makes this trap more likely, because it
replaces deliberate thought with habit (`SRE Ch. 11, p. 164–165`).

### 🔵 D-08 — The rare cause chosen before the common one
**Signature:** the first hypothesis names a rare component or an exotic failure. Nobody
checked a common cause.
**Consequence:** time goes to a search that the base rate does not justify.
**Fix:** prefer the common cause and the simpler explanation (`SRE Ch. 12, p. 171`). Keep
the counterweight. A set of common low-grade faults can explain the symptoms better than one
rare fault (`SRE Ch. 12, p. 187`).

### 🔵 D-09 — Exactly one root cause recorded
**Signature:** the postmortem template holds one root cause field and one action item.
**Consequence:** the other defects that produced the incident stay in production. The
incident repeats through a second path.
**Fix:** record every defect that, if repaired, gives confidence that the event does not
happen again in the same way. Repair each one. `SRE Ch. 6, p. 83`. `SRE Ch. 12, p. 182`.

### 🟡 D-10 — An amplifier named as the origin
**Signature:** the diagnosis names the retry rate, the health check failure count, or the
restart count as the root cause.
**Consequence:** the team removes the amplifier and the original fault returns. A retry
graph can indicate bad retry behavior. It can also be a compounding cause and not the
origin.
**Fix:** separate the term that started the fault from the term that grows it. State which
one you removed (`SRE Ch. 22, p. 327`). A cascading failure grows through positive feedback,
so it always holds both terms (`SRE Ch. 22, p. 313`).

### 🟡 D-11 — The system as designed, not as it runs
**Signature:** the diagnosis cites an architecture diagram, a design document, or a code
comment. It cites no telemetry from the running process.
**Consequence:** the responder reasons about a system that does not exist. Every deduction
after that point is unsound.
**Fix:** read the exported state of the running server. A server endpoint that lists its
recent calls tells you how that server communicates, with no diagram. `SRE Ch. 12, p. 175`.

---

## C. Traps in the telemetry

### 🟡 D-12 — Mean latency over a bimodal distribution
**Signature:** the dashboard panel shows `avg` latency or `mean` latency. No p99 panel and
no histogram exists for that call.
**Consequence:** a fault where 5% of requests hang for the full deadline can produce an
80.4% error rate at the frontend. The mean latency stays normal through it.
**Fix:** record request counts in latency buckets. Space the bucket boundaries about
exponentially. Read the distribution, not the mean. `SRE Ch. 22, p. 330–331`.
`SRE Ch. 6, p. 89–90`.

### 🟡 D-13 — Failed requests absent from the latency metric
**Signature:** the code writes the latency metric when the response completes. No counter
exists for an aborted request or for a timeout.
**Consequence:** a request that never completes never enters the average. A total outage
reads as a mild slowdown. In the Black Friday incident the latency was effectively infinite,
and the statistics showed the average of the requests that completed.
**Fix:** count aborted requests and timeouts as separate series. Track the latency of failed
requests. A slow error is worse than a fast error. `Release It! Sec. 16.6, p. 258–259`.
`SRE Ch. 6, p. 88`.

### 🟡 D-14 — One side of a boundary instrumented
**Signature:** the caller records the latency of a dependency. The dependency reports no
latency of its own, or nobody collects it.
**Consequence:** you cannot separate a slow dependency from a slow network between you and
it. The two faults need different fixes.
**Fix:** record how fast the caller believes the dependency to be, and how fast the
dependency believes itself to be (`SRE Ch. 6, p. 87`). Expose per dependency the Circuit
Breaker state, the timeout count, the error counts by class, and the address of the remote
endpoint (`Release It! Sec. 17.6, p. 298`).

### 🟡 D-15 — White-box signals only
**Signature:** no probe sends a request from the user vantage point. Every dashboard reads
an internal counter.
**Consequence:** you see only the requests that arrive. A request lost to a name resolution
fault is invisible. You can alert only on the faults that you expected. A dead process and a
hung process write no log line at all.
**Fix:** add a black-box probe against the user-facing address. Add a second probe behind
the load balancer, so you can locate the failing hop. `SRE Ch. 10, p. 155–156`.
`Release It! Sec. 17.5, p. 283`.

### 🟡 D-16 — An up-or-down check only
**Signature:** the dependency has one health check that returns healthy or unhealthy. No
error rate series and no latency series exists for it.
**Consequence:** retries hide the degradation. The check stays green while the users wait.
Slow response is worse than no response.
**Fix:** add white-box signals per dependency. White-box monitoring detects the faults that
retries mask. `SRE Ch. 6, p. 87`. `Release It! Sec. 8.3, p. 165`.

### 🟡 D-17 — A gap in the time series read as health
**Signature:** an alert rule that needs data points to fire. No synthetic series records
whether the collection succeeded.
**Consequence:** a target that stops answering produces a gap, and the gap fires no alert.
The fault is silent. A collection agent dies with its host and says nothing.
**Fix:** record four synthetic series per target. The first says whether the name resolved.
The second says whether the target answered the collection. The third says whether it
answered the health check. The fourth gives the time that the collection finished. Use the
collection failure itself as a signal (`SRE Ch. 10, p. 144–145`). A heartbeat detects a dead
agent (`Release It! Sec. 17.5, p. 285`).

### 🔵 D-18 — Tracing that cannot see in-process time
**Signature:** a trace with a gap between the request start and the first outgoing call. The
tracing library records remote calls only.
**Consequence:** work inside the process is invisible. A distributed trace reports that no
dependency is slow while the caller holds the fault. The App Engine case had a gap of about
250 ms with no call in it.
**Fix:** add application-level instrumentation for the path inside the process
(`SRE Ch. 12, p. 185–186`). **Modern:** a continuous profiler in production covers the same
gap. Neither book has one.

### 🟡 D-19 — No request identifier across the hops
**Signature:** log lines from two services that handled one request carry no shared
identifier. The log format has no trace field.
**Consequence:** the responder matches an upstream entry to a downstream entry by time and
by guess. After an outage, 10,000 log lines hold no search string.
**Fix:** propagate one unique request identifier through the whole span of calls. Print it
on every line. `SRE Ch. 12, p. 186`. `Release It! Sec. 17.4, p. 283`. **Modern:** a trace
context header that an instrumentation library carries does the same job.

---

## D. Traps in the log and the alert

### 🔵 D-20 — The probe does not store the failing response
**Signature:** the probe writes a success result or a failure result. It stores no status
code, no headers, and no body from the failed attempt.
**Consequence:** the responder must reproduce the fault by hand to learn that the service
returned a 502 status with no payload. That cycle is pure waste. A monitoring system reports
only the condition that a person told it to detect
(`Release It! Sec. 17.5, p. 284–285`).
**Fix:** store the status, the headers, and the body of every failed probe
(`SRE Ch. 12, p. 175–176`). Validate the contents of the payload, not only the status code
(`SRE Ch. 10, p. 156`).

### 🔵 D-21 — Debug logging enabled in production
**Signature:** a logging configuration file in the production profile that sets a level of
`DEBUG` or `TRACE`. Search the deployed configuration, not the repository default.
**Consequence:** a real fault hides under method traces and checkpoint lines. One debug
message caused a weekly database failover for six months, because a responder read it as an
instruction.
**Fix:** add a build step that removes any configuration which enables a debug level or a
trace level. `Release It! Sec. 17.4, p. 278, p. 281–283`. **Modern:** the same check runs
against the container image in the pipeline.

### 🔵 D-22 — A log format that a search tool cannot parse
**Signature:** a log record that spans two lines. No severity column. No message code field.
**Consequence:** a person cannot scan the file, and `grep` cannot parse it. The format
defeats the person and the tool at the same time.
**Fix:** write one record per line, in space-padded columns, with a severity indicator and a
message code. Give operations the catalog of message codes.
`Release It! Sec. 17.4, p. 278–281`.

### 🟡 D-23 — More than one alert for one incident
**Signature:** the paging channel holds several alert names with the same start time. The
alerting configuration has no inhibition rule and no grouping rule.
**Consequence:** the responder triages duplicates instead of the fault. In one incident the
alerts spammed the normal channel and the emergency channel, and made communication harder.
**Fix:** group the related alerts. Silence a duplicate alert during an incident. Tune the
configuration toward one alert per incident. `SRE Ch. 11, p. 167`. `SRE Ch. 13, p. 193–194`.

### 🟡 D-24 — An alarm that fires during normal operation
**Signature:** an alert with a low acknowledgement rate, or a team that reports routine
transient pages for the same condition.
**Consequence:** the false alarms train the responders to ignore the condition. The one real
alarm is then indistinguishable from the noise. In the Black Friday incident an engineer did
not answer a page for 100% processor use, because that group received such pages routinely.
**Fix:** set the expectation from the historical data of the same hour of the week. Start
loose, then tighten (`Release It! Sec. 16.8, p. 261`, `Sec. 17.7, p. 303–304`). Retire a
report or an alert that nobody reads (`Release It! Sec. 17.8, p. 305`).

### 🔴 D-25 — A remedy that removes the monitoring
**Signature:** an automation step, a turndown script, or a drain procedure that deletes or
disables the monitoring configuration for the target.
**Consequence:** the team loses sight of the damage while the damage grows. In one incident
the on-call engineers had to reverse the monitoring changes before they could measure the
blast radius.
**Fix:** exclude the monitoring configuration from any automated turndown. Keep the reverse
operation for a monitoring change ready. `SRE Ch. 13, p. 196`.

---

## E. Traps in the test

### 🟡 D-26 — The test runs from the wrong vantage point
**Signature:** a probe or a manual command that runs on a workstation, while the real caller
is a server in another network.
**Consequence:** a firewall rule that permits one source address makes the test fail,
although the real caller would succeed. The responder then removes a correct hypothesis.
**Fix:** run the test from the same vantage point as the real caller. State the vantage
point in the record. `SRE Ch. 12, p. 179`.

### 🟡 D-27 — The test changes what it measures
**Signature:** verbose logging enabled, a processor added, or a cache cleared during the
investigation, with no note in the incident record.
**Consequence:** you can no longer tell whether the fault grew on its own or grew because of
your instrumentation. More processors make a request faster and make a data race more
likely.
**Fix:** treat an active test as a change to the system. Record it. Reverse it.
`SRE Ch. 12, p. 179–180`.

### 🔵 D-28 — A test that does not separate the hypotheses
**Signature:** a test whose pass result and whose fail result both fit two hypotheses that
are still open.
**Consequence:** the test consumes time and removes nothing. The list of hypotheses does not
become shorter.
**Fix:** design the test so that it rules one group of hypotheses in and another group out.
Run the tests in decreasing order of likelihood. Weigh the risk that each test poses to the
system. `SRE Ch. 12, p. 179`.

### 🔴 D-29 — The system left in an unknown state
**Signature:** a production flag differs from source control after the incident. The
incident record lists no change that the team made during the investigation.
**Consequence:** the system runs in an undocumented configuration. Nobody can restore the
state before the test. The next fault then holds a new variable.
**Fix:** make each change in a systematic and documented way. Then you can return the system
to its state before the test. `SRE Ch. 12, p. 180`.

### 🟡 D-30 — A suggestive result reported as proof
**Signature:** a race condition or a deadlock claimed as proven, from one run that nobody
reproduced. A runbook entry that states a remedy and names no cause.
**Consequence:** the team ships a fix for a cause that it did not establish. The real fault
stays. An unproven remedy becomes team lore, and then a documented procedure.
**Fix:** state the strength of the evidence. Write "probable cause" when the fault resists
reproduction (`SRE Ch. 12, p. 180, p. 182`). Never let a remedy that appeared to work become
a procedure without a proven cause (`Release It! Sec. 17.4, p. 281–283`).

---

## F. Traps in the change hypothesis

### 🟡 D-31 — The change log covers code only
**Signature:** the deploy log lists commits and images. It lists no configuration change, no
infrastructure change, no vendor maintenance window, and no demand campaign.
**Consequence:** the true trigger stays invisible. One outage came from a vendor maintenance
window and a newspaper insert that drove demand at the weakest dependency. Neither one was a
commit.
**Fix:** log every new version deployment and every configuration change, at every layer of
the stack. Include the third-party maintenance calendar and the demand campaigns.
`SRE Ch. 12, p. 177`. `SRE Ch. 22, p. 335`. `Release It! Sec. 16.8, p. 261`.

### 🟡 D-32 — The last change named without a test
**Signature:** the diagnosis names the most recent deploy. The fault started before the
deploy started, or no test excludes the change.
**Consequence:** the team reverses a change that did not cause the fault, and the fault
stays. The heuristic that finds most causes also anchors you on the wrong one.
**Fix:** compare the fault start time against the change window. State that no change is in
the window when that is true. Then test the change like any other hypothesis.
`SRE Ch. 12, p. 177–178, p. 184`.

### 🔵 D-33 — An error graph with no deploy marks
**Signature:** the error-rate dashboard has no annotation layer for the deploy start time
and the deploy end time.
**Consequence:** the responder aligns two graphs by eye, and gets the order of events wrong.
**Fix:** annotate the error-rate graph with the start time and the end time of each
deployment. `SRE Ch. 12, p. 178`.

---

## G. Traps in the tools and the levers

### 🔴 D-34 — The diagnosis tools sit behind the fault
**Signature:** the dashboard, the chat system, or the deploy tool runs on the same cluster
as the fault, or depends on the same service. The monitoring traffic crosses the same
network segment as the production traffic.
**Consequence:** the fault removes the tools that you need to diagnose the fault. In one
outage most of the troubleshooting stack and the communication stack sat behind the
crash-looping jobs.
**Fix:** keep a low-overhead communication system outside the serving path, and use it
regularly. Keep command-line tools and alternative access that work when the normal
interfaces do not (`SRE Ch. 13, p. 192–194`). Keep the monitoring traffic off the segments
that carry public traffic (`Release It! Sec. 17.5, p. 285`).

### 🔴 D-35 — A rollback path that was never tested
**Signature:** the release process holds no rollback rehearsal. Nobody tested the rollback
procedure in a test environment.
**Consequence:** the rollback fails at the moment you need it, and the outage grows. One
team attempted a rollback of a permissions change, failed, and lengthened the outage.
**Fix:** test the rollback procedure before the change that may need it
(`SRE Ch. 13, p. 190–191`). `reverse-branching` owns the rollback mechanism. This catalog
only names its absence.

### 🔵 D-36 — Remediation that needs a console
**Signature:** the remediation step tells a person to open a graphical console. No script
and no command-line interface exists for the same action.
**Consequence:** the remedy is slow, and it differs across hosts. A console is impractical
at thirty or forty servers.
**Fix:** make the administration interface scriptable. Then one command sets the property,
stops the component, and starts it again. `Release It! Sec. 17.6, p. 296–297`.
`Release It! Sec. 16.3, p. 254–255`.

### 🔵 D-37 — Restart is available only for the whole server
**Signature:** the only restart control is a process restart or a host restart. No control
restarts one component.
**Consequence:** the cheapest safe remedy is unavailable. One dynamic reconfiguration and
one component restart took less than five minutes. A change to a configuration file and a
full restart would have taken more than six hours under that load.
**Fix:** provide a restart at the level of one component. It is a key concept of
recovery-oriented computing (`Release It! Sec. 16.10–16.11, p. 263–264`). An `enabled`
property on an integration point gives the same control with less risk
(`Release It! Sec. 16.9, p. 262`).

---

## H. Traps in the record

### 🔵 D-38 — The report goes to a person, not to a queue
**Signature:** an incident that began in a direct message or in an email to one engineer,
and that never became a ticket.
**Consequence:** three harms follow. Somebody must transcribe the report later. The report
stays invisible to the team. The load concentrates on the engineers whom the reporters
happen to know, and not on the engineer who is on duty.
**Fix:** open a ticket for every issue, including one that arrives by email or by message.
Route it to the person on duty. `SRE Ch. 12, p. 172`.

### 🔵 D-39 — A problem report with no expected behavior
**Signature:** a ticket that states a symptom, and states no expected behavior and no
reproduction.
**Consequence:** the responder starts by rebuilding the report. The report has no consistent
form, so nobody can search it against the past reports.
**Fix:** require the expected behavior, the actual behavior, and the reproduction where one
exists. Use one form, in one searchable location. `SRE Ch. 12, p. 172`.

### 🟡 D-40 — A negative result that nobody published
**Signature:** an incident document, a design document, or a review that names no failed
test and no rejected approach.
**Consequence:** the next team repeats the same removed hypothesis. A document with no
failure in it is either filtered, or the author was not rigorous.
**Fix:** record every test that removed a hypothesis, and publish it. A negative result is
conclusive, and the tool that produced it outlives the experiment.
`SRE Ch. 12, p. 180–182`.

---

## Index by symptom

| Symptom | Check these entries |
|---|---|
| The fault stopped and nobody knows why | D-02, D-03, D-29, D-30 |
| Every component reports healthy and users see a fault | D-13, D-15, D-16, D-20 |
| The dashboard looks normal during the outage | D-12, D-13, D-17, D-33 |
| The same fault returns every week | D-07, D-09, D-24, D-40 |
| Nobody can say which change caused it | D-31, D-32, D-33 |
| The responders cannot communicate during the outage | D-23, D-34 |
| The team removed the wrong hypothesis | D-26, D-28 |
| The fix did not work | D-06, D-10, D-11, D-32 |
| The remedy took hours | D-35, D-36, D-37 |
| The incident had no ticket | D-38, D-39 |
| The blast radius grew during the response | D-01, D-04, D-05, D-25 |
