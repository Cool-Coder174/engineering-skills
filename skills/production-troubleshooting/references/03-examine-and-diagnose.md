# Examine and Diagnose

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 12, the sections "Examine" and "Diagnose", p. 174–178. Supporting pages: Ch. 6,
p. 87–90. Ch. 10, p. 144–145 and p. 155–156. Ch. 12, p. 183–186. Ch. 22, p. 317–333.
*Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007). Ch. 16, Sec. 16.3–16.11,
p. 254–264.

**Read this file when you hold observations and no cause.** Step 2 of the loop gave you a
mitigation. Step 3 collects observations. Step 4 turns them into hypotheses.
`04-test-and-treat.md` then separates the hypotheses.

Examine and diagnose are two steps. Examine reads the system. Diagnose reasons about what
it read. A responder who joins the two steps adopts the first plausible story.

---

## 1. What examine means

**Examine what each component does, to decide whether it behaves correctly**
(`SRE Ch. 12, p. 174`). The step produces a list of components that behave wrongly. It
does not produce a cause.

The step uses three sources of data (`SRE Ch. 12, p. 174–175`).

1. Metrics. Graph the time series. Apply operations to the time series.
2. Logs. Export one record per operation, and records of system state.
3. Traces. Follow one request through the whole stack.

**A component that reports healthy can still fail the user.** Every component in the
environment can run correctly on its own while the user sees a fault. This happens with
blocked threads and with cascading failure (`Release It! Sec. 17.5, p. 287`).

**Read the system as it runs, not as the diagram draws it.** A server endpoint that lists
its recent calls tells you how the server communicates, without an architecture diagram
(`SRE Ch. 12, p. 175`). See D-11 in `diagnostic-trap-catalog.md`.

---

## 2. The four examine questions

Ask these four questions of every component that you suspect. Each row names the telemetry
that answers the question, and the trap that appears when the telemetry is absent.

| Question | Telemetry that answers it | Trap |
|---|---|---|
| What does the component serve? | Traffic rate. Error rate by class. Latency histogram. Aborted request count. (`SRE Ch. 6, p. 88`) | D-12, D-13 |
| What does it ask of its dependencies? | Per-dependency call count, latency, timeout count, error class, and endpoint address (`Release It! Sec. 17.6, p. 298`) | D-14, D-18 |
| What does it consume? | Processor, memory, threads busy over five seconds, queue depth, pool checkouts, high-water mark (`Release It! Sec. 17.1, p. 270–271`) | D-17 |
| What does the user see? | An external probe from the user vantage point. The probe stores the failing body. (`SRE Ch. 10, p. 155–156`) | D-15, D-20 |

**The four questions collect the four golden signals.** They are latency, traffic, errors,
and saturation (`SRE Ch. 6, p. 88`). `observability` Section 3 owns the definition of each
signal, the three error classes, the failed-request latency rule, and the histogram
boundaries. Do not restate them here. Read them there when a signal is absent or unreadable.

**Instrument both sides of every boundary.** Without both sides you cannot separate a slow
dependency from a slow network (`SRE Ch. 6, p. 87`). This rule decides which of two
components you examine next, so it belongs to the examine step.

---

## 3. Logging: four capabilities that shorten the step

Text logs help reactive debugging in real time. Structured binary logs let you build tools
for retrospective analysis (`SRE Ch. 12, p. 174`). Design the logging system so that you can
enable it as needed, quickly and selectively (`SRE Ch. 12, p. 175`).

| Capability | The signal that you need it | Source |
|---|---|---|
| Several verbosity levels, changeable while the process runs | You need detail on one operation and you cannot restart the process | `SRE Ch. 12, p. 174` |
| Statistical sampling | The traffic volume is high. Full logging costs too much. Show one of every 1,000 operations. | `SRE Ch. 12, p. 174` |
| A selection language | You must isolate one shape of operation, such as a call slower than 10 ms, or a payload below 1,024 bytes | `SRE Ch. 12, p. 174–175` |
| One record per line, with a severity column and a message code | A person must scan the file, and `grep` must parse it | `Release It! Sec. 17.4, p. 280–281` |

**Print one request identifier on every line.** A shared identifier joins upstream and
downstream records without guesswork (`SRE Ch. 12, p. 186`). Nygard gives the same rule.
The identifier can be a user id, a session id, or a number that the entry point assigns
(`Release It! Sec. 17.4, p. 283`). **Modern:** an instrumentation library can propagate a
trace context header and do the same job. Neither book describes one.

**Dead processes log no tales. Hung processes log no tales either**
(`Release It! Sec. 17.5, p. 283`). A hang needs a thread dump, not a log search.

---

## 4. Exposed state: the third tool

**Expose current state on an endpoint of the running server** (`SRE Ch. 12, p. 175`). Google
servers expose these items.

- A sample of the calls that the server recently sent and received.
- Histograms of error rate and latency for each type of call.
- The current configuration, and in some servers the data itself.
- In Borgmon, the monitoring rules, and a trace of one computation back to its source
  metrics.

The first item removes the need for an architecture diagram. The second item tells you
which dependency is unhealthy in one read.

**Instrument a client as the last tool of examine.** Build or modify a client to discover
what a component returns in response to requests (`SRE Ch. 12, p. 175`).

---

## 5. Nygard's clinical order

Nygard runs the same step under clinical names. The order is fixed. Do not change it.

| Nygard's step | What the responder does | Source |
|---|---|---|
| Take the pulse | Learn the normal rhythm of the system before the fault | `Release It! Sec. 16.3, p. 254–255` |
| Vital signs | List every measurement that you already hold, at one moment | `Release It! Sec. 16.6, p. 257–259` |
| Diagnostic tests | Run a test that produces new data, such as a thread dump | `Release It! Sec. 16.7, p. 259` |
| Call a specialist | Page the owner of the dependency that the tests named | `Release It! Sec. 16.8, p. 260–261` |
| Compare treatment options | List the moves. Reject any move whose effect nobody knows. | `Release It! Sec. 16.9, p. 262` |
| Does the condition respond? | Apply the move to one node. Read the result. | `Release It! Sec. 16.10, p. 262–263` |

**Learn normal before the fault, not during it.** Nygard's team sampled latency, free heap,
active request threads, and active sessions on a loop. In the time since the site launched,
the team learned what was normal for noon on Tuesday in July
(`Release It! Sec. 16.3, p. 255`). The book gives no sampling duration.

**A graphical console does not scale to the step.** A console explores a system when you do
not know what you search for. It becomes impractical at thirty or forty servers
(`Release It! Sec. 16.3, p. 255`). See D-36 in `diagnostic-trap-catalog.md`.

**Write the vital signs as a list, before you reason.** The list from the Black Friday
incident held six items at twenty minutes (`Release It! Sec. 16.6, p. 258`).

1. Session counts were high, higher than the day before.
2. Network bandwidth use was high but below its limit.
3. Application server page latency was high.
4. Processor use was low on the web, application, and database hosts.
5. The search servers, the usual suspect, responded well.
6. Request threads were almost all busy, many for more than five seconds.

**Two of those six items form a fingerprint.** Busy threads plus low processor use means
that the threads block. They do not compute (`Release It! Sec. 16.7, p. 259`).

---

## 6. Diagnose: choose the technique

A design that you understand helps you form a hypothesis. The techniques below work without
that understanding (`SRE Ch. 12, p. 176`).

| Signal | Technique | Cost |
|---|---|---|
| The stack has few layers and known interfaces | Divide and conquer. Traverse the stack from one end to the other end. (`p. 176`) | Linear in the layer count |
| The stack is large. A linear traversal is too slow. | Bisection. Split the system. Examine the boundary. Repeat. (`p. 176`) | Logarithmic in the component count |
| Components have defined inputs and outputs | Simplify and reduce. Inject known data at each hop. Read the output. (`p. 176`) | You must build a test input |
| The system does something, but not the wanted thing | Ask what, then where, then why (`p. 176–177`) | Low. It needs a profiler. |
| The fault started at a point in time | What touched it last. Read the change log. (`p. 177`) | It needs a change log. See D-31. |
| This service faults often | Build a diagnostic tool for this service (`p. 178`) | High once. Low after that. |
| Every component reports healthy | Read the user path from outside (`SRE Ch. 10, p. 156`) | It needs an external probe |

You may combine the techniques. The list is not an order.

---

## 7. Simplify and reduce

**Treat the system as components with defined inputs and outputs.** A component performs a
known transformation from its input to its output (`SRE Ch. 12, p. 176`). Then you can read
the data between two components and decide whether the first one works.

1. Select the hop that you suspect.
2. Inject known test data at that hop.
3. Read the output and compare it against the expected output.
4. Inject data that probes a suspected cause of the error.
5. Repeat at the next hop.

**A reproducible test case makes the diagnosis much faster** (`SRE Ch. 12, p. 176`). A
reproducible case can also run in a non-production environment. There you may use a
technique that is too invasive for production.

---

## 8. Divide and conquer, and bisection

**Divide and conquer.** Begin at one end of the stack. Examine each component in turn until
you reach the other end. The technique suits a data processing pipeline
(`SRE Ch. 12, p. 176`).

**Bisection.** Split the system in half. Examine the communication paths between the two
halves. Decide which half works. Repeat until one component remains
(`SRE Ch. 12, p. 176`).

**One response field can discount two tiers in one step.** A response header that lists the
backend servers which served the request makes one claim likely. The request reached the
backends and failed there, because the response probably would not carry that header
otherwise. That one fact discounts the frontend server and the load balancers
(`SRE Ch. 12, p. 176, p. 178`). Design responses so that they carry their own provenance.
This is the cheapest bisection, and it costs nothing during the incident.

---

## 9. Ask what, then where, then why

**A malfunctioning system is often still doing something. It does not do the thing that you
want** (`SRE Ch. 12, p. 177`). So ask three questions in order.

1. **What** does the system do now?
2. **Where** do its resources go, or where does its output go?
3. **Why** does it do that?

The Spanner chain shows the shape (`SRE Ch. 12, p. 177`).

| Question | Answer in the Spanner case |
|---|---|
| Symptom | The cluster has high latency. Calls to its servers time out. |
| Why? | The server tasks use all their processor time. |
| Where does the processor time go? | The profiler shows a sort of entries in logs checkpointed to disk. |
| Where in the sort code? | The evaluation of a regular expression against log file paths. |

The chain ends at a specific defect and a specific repair. Rewrite the expression so that it
does not backtrack. Search the codebase for the same pattern. Consider RE2, which does not
backtrack and guarantees linear runtime growth with input size (`SRE Ch. 12, p. 177`).

**The technique needs a profiler.** Without one, step 2 has no answer.

---

## 10. What touched it last

**Systems have inertia. A working system stays in motion until an external force acts on
it.** That force is often a configuration change or a shift in the type of load served
(`SRE Ch. 12, p. 177`).

Recent changes are a productive place to begin. The technique needs a change log that records
new version deployments and configuration changes at all layers of the stack. That range runs
from the server binaries down to the packages on individual nodes (`SRE Ch. 12, p. 177`).

**Annotate the error-rate graph with the start time and the end time of each deployment**
(`SRE Ch. 12, p. 178`). Then a responder reads the order of events instead of guessing it.

**The change log must state absence as confidently as it states presence.** The App Engine
team removed change as a cause. The fault started on a Saturday, and no application push and
no production push was in flight (`SRE Ch. 12, p. 184`).

**A change is not only a commit.** The Black Friday outage needed two non-code changes. A
vendor took two of four scheduling servers down for maintenance. Marketing ran a newspaper
insert that offered free home delivery (`Release It! Sec. 16.8, p. 261`). See D-31 and D-32
in `diagnostic-trap-catalog.md`.

`reverse-branching` owns the decision to reverse a change. This step only produces the
change window.

---

## 11. Specific diagnoses

**Build a diagnostic tool for a service that faults often** (`SRE Ch. 12, p. 178`). Google
SREs spend much of their time on this work. The signal that justifies the cost is
repetition. One fault does not justify a tool. The third occurrence of one alert does.

**Search for commonalities between services and teams first.** A tool that two teams share
costs less than two tools (`SRE Ch. 12, p. 178`).

---

## 12. Symptom to first hypothesis

Each row states a hypothesis, not a cause. `04-test-and-treat.md` tells you how to test it.

| Observation | First hypothesis | Source |
|---|---|---|
| Threads are busy. Processor use is low. | A caller blocks on a dependency, or on a pool with no timeout | `Release It! Sec. 16.6–16.7, p. 258–259` |
| Mean latency is normal. The frontend error rate is high. | A small share of requests hangs for the full deadline | `SRE Ch. 22, p. 330–331` |
| Latency rises after a restart or after a new cluster starts | A cold cache that the service needs for its capacity | `SRE Ch. 22, p. 331–333` |
| Latency rises with no code change and no traffic rise | Per-request work that grows with the stored data | `SRE Ch. 12, p. 185–186` |
| Processor use is at 100% on one small dependency | That dependency is the constraint. Every tier above it queues. | `Release It! Sec. 16.8, p. 261` |
| Health checks fail and the process still runs | Overload starves the health check path | `SRE Ch. 22, p. 317` |
| A trace shows a gap before the first outgoing call | The caller does the slow work in the process | `SRE Ch. 12, p. 185` |
| A metric correlates with the fault | Test the correlation before you build the fix | `SRE Ch. 12, p. 184–185` |

**Prefer the common cause.** When you hear hoofbeats, think of horses, not zebras
(`SRE Ch. 12, p. 171`). Note the counterweight. A set of common low-grade faults can explain
the symptoms better than one rare fault (`SRE Ch. 12, fn. 62, p. 187`).

---

## 13. War stories

**The Shakespeare search fault** (`SRE Ch. 12, p. 173, p. 175–176, p. 178`). A black-box
probe reported no search results for five minutes, and the alerting system filed a bug with
the probe results and the playbook entry. The probe sent one request each minute and
expected a 200 status with an exact payload. About half the probes succeeded over ten
minutes, with no discernible pattern. The probe did not store the failing response, so the
responder reproduced the fault by hand and saw a 502 status with no payload. The response
carried a header that listed the backend servers which handled the request. That header
proved the request reached the backends, so the responder discounted the frontend and the
load balancers and examined the backends alone. This story proves two rules. Self-describing
responses remove whole tiers in one step. A probe that hides the failing response costs one
manual cycle.

**The App Engine latency mystery** (`SRE Ch. 12, p. 182–186`). A customer reported latency
up by nearly an order of magnitude. Processor time and serving processes were nearly
quadrupled, with no code change and no traffic rise. The team removed change as a cause,
because the shift happened on a Saturday with no push in flight. The developers found a
correlation with a rise in one datastore call, and started to build composite indices.
Tracing then showed that requests for static content were also slow. Static content never
touches the datastore, so the correlation was spurious and the index work was wasted. The
same tracing showed fast dependency calls and a gap of about 250 ms before the first call.
The tracing tool records only remote calls, so the in-process work was invisible. SRE
mitigated first, and moved the application to the instance type with the most processor
capacity. The developers then instrumented the application and found a whitelist check on
every request, backed by an in-memory cache. A long-standing access-control defect created
one whitelist object for each access to one path. A security scanner then produced thousands
of them in half an hour. This story proves three rules. A correlation with a plausible
mechanism can still be a coincidence. Remote-call tracing is blind to in-process time.
Mitigation may precede diagnosis.

**Black Friday at the online retailer** (`Release It! Sec. 16.5–16.11, p. 256–264`). The
site went down on the morning of Black Friday, and lost orders at about one million dollars
an hour. Rolling restarts failed at once. The vital signs showed high session counts, high
page latency, low processor use, and request threads busy for more than five seconds. The
recorded latency was an average of the requests that completed, so the requests that timed
out never entered it. Thread dumps on the 3,000 front-end threads showed most threads
blocked on a connection pool with no timeout. Thread dumps one hop out showed the 450
threads of the order management system blocked on an external scheduling system. That
scheduling system ran on one working server out of four, at 100% processor use. It received
about 90 requests against a capacity near 25 concurrent requests. Two servers were down for
holiday maintenance and a third was broken. Marketing had run a newspaper insert that drove
the demand. The team throttled at the connection pool for that one dependency and tested the
move on one node. It learned that the maximum applies only at pool start, so it recycled the
pool component and then applied the move to the fleet. The external probe turned green about
ninety seconds later. This story proves four rules. Busy threads plus low processor use
means blocking. An average latency hides an infinite latency. A per-dependency pool is the
throttle that the incident needs. A component-level restart takes minutes where a full
restart takes hours.

---

## 14. Traps that appear at this step

Run these entries from `diagnostic-trap-catalog.md` before you accept a hypothesis.

| Group | Entries | Why they belong to this step |
|---|---|---|
| Reasoning | D-06, D-07, D-08, D-10, D-11 | They corrupt the move from observation to hypothesis |
| Telemetry | D-12, D-13, D-14, D-15, D-16, D-17, D-18, D-19 | They make the observation wrong or absent |
| Log and probe | D-20, D-21, D-22 | They hide the evidence that examine needs |
| Change hypothesis | D-31, D-32, D-33 | They break "what touched it last" |

**"Not applicable" is a valid scan result.** Report only the traps that you can name with
evidence.

---

## 15. When to leave this step

Leave step 3 and step 4 when all three conditions hold.

1. You can name the component that behaves wrongly, and the telemetry that shows it.
2. You hold two or more hypotheses that a test can separate.
3. Each hypothesis cites an observation from the running system, not a design document.

If condition 1 fails, the telemetry is missing. Record the gap in Section 10 of the Fault
Record, and give it to `detail-planning`. If condition 2 fails with one hypothesis, you have
guessed. Read `05-negative-results-and-bias.md`.

---

## 16. Cross-references

| File | Read it when |
|---|---|
| `01-the-troubleshooting-model.md` | You need the six steps and the loop rule |
| `02-triage-first-diagnose-second.md` | The service is down and you must act now |
| `04-test-and-treat.md` | You hold two or more hypotheses |
| `05-negative-results-and-bias.md` | The diagnosis feels certain |
| `06-making-troubleshooting-easier.md` | The telemetry that you needed was absent |
| `diagnostic-trap-catalog.md` | Every fault, before you accept a cause |
| `observability/SKILL.md` | A signal is absent, or you must define the one that was missing |
| `capacity-engineering/SKILL.md` | The examine step named a saturated resource |
| `data-systems-design/references/hazard-catalog.md` | The cause is a replication or concurrency anomaly |
| `systems-programming/references/failure-catalog.md` | The cause is a blocked call or a lost durable write |
| `system-design/references/02-estimation.md` | You must estimate the constraint of a tier |
| `security-engineering/SKILL.md` | The trigger is untrusted input or a privilege change |
