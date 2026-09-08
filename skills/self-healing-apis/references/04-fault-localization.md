# Localizing a Fault Across a Dependency

**Sources:** *Site Reliability Engineering* Ch. 12, "Effective Troubleshooting", pp. 169–188.
*Release It!* Ch. 16, "Phenomenal Cosmic Powers, Itty-Bitty Living Space", pp. 252–264.
Supporting pages: SRE Ch. 6, pp. 82–90. SRE Ch. 13, pp. 189–199. SRE Ch. 22, pp. 313–343.
Release It! Ch. 2, pp. 21–34. Sec. 4.1, pp. 46–60. Ch. 13, pp. 229–232. Ch. 17, pp. 265–309.

Read this file when a dependency looks faulty and you must prove where the fault is.

**`production-troubleshooting` owns the general method.** Read
`production-troubleshooting/references/01-the-troubleshooting-model.md` for the six steps, the
diagnostic techniques, and the fault record. This file states only what changes when the
suspect party is a vendor we do not operate. `observability` owns signal design and alert
rules. `incident-response` owns the roles and the written incident state.

---

## 1. The question this file answers

A fault crossed a boundary. Four locations are possible.

1. **Us.** Our process, our resource pool, our client code, our host, or our change.
2. **The network.** The route to the vendor, and every middlebox in that route.
3. **The vendor.** Their application, their load balancer, or their capacity.
4. **A shared dependency.** A component that we and another consumer both call.

**A fault is localized when you can name the location and name the evidence.** "The vendor
is down" is not a location. It stays a guess until a probe supports it.

**Name the layer first. Name the cause second.** A thread dump names the layer that holds
the threads. It does not name the reason that layer is slow. (Release It! Ch. 2, p. 29.)
One person's symptom is another person's cause. A slow read is a symptom to the vendor and a
cause to us. (SRE Ch. 6, p. 87.)

---

## 2. Restore service before you localize the fault

**Triage precedes root-cause analysis.** SRE states the instinct and rejects it. Your first
response in a major outage is to make the system work as well as it can. (SRE Ch. 12,
p. 173.)

**Stopping the bleeding is the first priority.** You do not help users if the system dies
while you analyze it. (SRE Ch. 12, p. 173.) Nygard writes the same rule as "Restoring
service takes precedence over investigation." (Release It! Ch. 2, p. 24.)

**Fly the airplane.** A pilot flies the airplane before the pilot analyzes the emergency.
When a fault can corrupt data, freeze the system instead of letting it continue.
(SRE Ch. 12, p. 173.) Rapid triage still does not permit the loss of evidence. Preserve the
logs and the dumps as you restore service. (SRE Ch. 12, p. 173.)

| Triage action | Signal that triggers it | What it costs you | Source |
|---|---|---|---|
| Divert traffic to a location that can serve | One location fails an independent probe | Latency for the moved users | SRE Ch. 12, p. 173 |
| Drop traffic wholesale | A cascade is in progress | Every request in the dropped class | SRE Ch. 12, p. 173 |
| Disable a subsystem | The subsystem drives the load | One feature | SRE Ch. 12, p. 173 |
| Throttle one dependency pool | Threads block on that pool | Part of one feature | Release It! Ch. 16, p. 262 |
| Freeze the system | A fault can corrupt data | Everything, for a short period | SRE Ch. 12, p. 173 |

**Collect the diagnostic bundle before you restart.** Nygard's team ran a script that took
thread dumps of every Java application and snapshots of the databases. The script does not
prolong the outage, and it feeds the post-mortem. (Release It! Ch. 2, p. 24.) Read
`07-automated-remediation-safety.md` before you automate any of these actions.

---

## 3. The troubleshooting loop, applied to one integration point

SRE names six steps. Problem Report, Triage, Examine, Diagnose, Test and Treat, Cure. The
loop returns from Test and Treat to Examine. (SRE Ch. 12, pp. 170–182.)

1. **Problem Report.** Record the expected behavior, the actual behavior, and a way to
   repeat the fault. For a vendor call, the expected behavior is the success pattern of your
   own probe. (SRE Ch. 12, p. 172.)
2. **Triage.** Apply Section 2 of this file. Size the response to the impact.
   (SRE Ch. 12, p. 173.)
3. **Examine.** Read the signals for this dependency. Read the error rate and the latency
   distribution for each operation. Read the breaker state, the pool state, and the thread
   dump. (SRE Ch. 12, pp. 174–175. Release It! Ch. 17, p. 298.)
4. **Diagnose.** Apply the four techniques of Section 6. (SRE Ch. 12, pp. 176–178.)
5. **Test and Treat.** Design one probe that separates two hypotheses. Run it. Record the
   result. Return to step 3. (SRE Ch. 12, pp. 179–180.)
6. **Cure.** Prove the cause as far as production permits. Write the post-mortem.
   (SRE Ch. 12, p. 182.)

**Google opens a bug for every issue, including one that arrives by mail or chat.** Route
the report to the engineer who is on duty, not to a named person. A direct report costs a
transcription step, and it hides the problem from the team. (SRE Ch. 12, p. 172.)

---

## 4. The four locations and the evidence that separates them

Read one row. Run its probe. Then read the table again.

| Location | Evidence that confirms it | Evidence that eliminates it | Probe to run | Source |
|---|---|---|---|---|
| **Us** | Our threads block on pool checkout. Our probe from a second host succeeds. Our change sits inside the fault window. | The same probe fails from a host that runs none of our code | Read the thread dump. Read the pool metrics. Read the change log. | Release It! Ch. 2, p. 29. SRE Ch. 12, p. 177 |
| **The network** | The connect attempt hangs and no reset packet arrives. Packets stop at one side of a middlebox. | A reset packet arrived, because the route carried it | Capture packets on both sides of the middlebox. Probe from a second route. | Release It! Sec. 4.1, pp. 51–53 |
| **The vendor** | Our threads block in a socket read after a finished handshake. Our probe fails from two separate locations. | Our probe succeeds from a second location on the same account | Send the known-good probe from outside our network. | Release It! Ch. 2, p. 29. Release It! Ch. 13, p. 231 |
| **A shared dependency** | Two consumers that share no code failed at almost the same time. | Only one consumer failed, and the other holds the same dependency | List the dependencies both consumers hold. Test the common one. | Release It! Ch. 2, pp. 25–26 |

**Instrument both sides of the boundary.** You must know how fast the caller believes the
vendor to be, and how fast the vendor believes itself to be. Without both numbers you cannot
separate a slow vendor from a slow network. (SRE Ch. 6, p. 87.) Catalog entry `I-34` names
the defect. A probe that fails to reproduce the fault is still a result. Record it.
(SRE Ch. 12, pp. 180–182.)

---

## 5. What each symptom class implies

The column "Mechanism" states why the signature looks the way it looks. Read it before you
accept the location, because the same signature can have a second mechanism.

| Signature | Mechanism | Likely location | Confirming probe |
|---|---|---|---|
| An immediate exception, from every caller | The vendor host sent a reset packet. The route carried it. | The vendor process, or their load balancer | Call the vendor from a second network location |
| The connect attempt hangs, with no reset packet | The listen queue is full, or a middlebox drops the packet | The network, or the vendor's accept path | Capture packets on both sides of the middlebox |
| Threads block in a socket read | The handshake finished. The vendor accepted and did not answer. | The vendor application | Send the known-good probe from outside our network |
| Threads block on pool checkout | Every connection is checked out and none returns | Our pool, driven by a slow dependency below it | Compare the in-flight count against the pool maximum |
| Low CPU and most threads busy | The threads wait on a call that does not return | A blocked integration point below us | Repeat both measurements one hop toward the vendor |
| Errors stop and no answers arrive | The threads are exhausted. No thread remains to raise an error. | Our own process | Read the thread dump |
| Latency rises for one operation only | The shared client code and the shared route are healthy | The vendor, for that operation | Compare latency per operation |
| About half of the probes succeed, with no pattern | Some vendor instances serve and some do not | A partial fault at the vendor | Probe repeatedly. Record every failing answer. |
| Latency is bimodal and some calls never finish | A subset of the vendor's shards is sick | The vendor, for a subset of keys | Read the latency distribution, not the mean |
| Errors start at the same time every day | A middlebox expired the idle entries of the pool | The network path | Add a quiet period to the load profile |
| The fault started inside our push window | A change acted on a system that had inertia | Our change | Annotate the error graph with the push window |

**The transition from errors to silence is a marker.** In the airline outage the first
twenty callers received an exception. After that the calls stopped raising errors and
started to block. Error-rate monitoring goes quiet exactly when the system gets worse.
(Release It! Sec. 3.3, p. 40.) A slow answer is worse than no answer, because it holds a thread
on both sides. (Release It! Sec. 4.1, p. 50.) See `01-integration-points.md`.

---

## 6. Bisect the request path

**Simplify and reduce.** Each component with a defined interface performs a known
transformation. Inject known test data at each step and check the output. Build a test case
that repeats. (SRE Ch. 12, p. 176.)

**Divide and conquer.** Start at one end of the stack and examine each component in order.
(SRE Ch. 12, p. 176.)

**Bisection.** Use it when the stack is too large for a linear walk. Split the path in half.
Examine the communication between the two halves. Decide which half is healthy. Repeat until
one component remains. (SRE Ch. 12, p. 176.)

**Ask what, where, and why.** A faulty system usually still does something. Determine what
it does. Then ask why it does that, and where its resources go. (SRE Ch. 12, pp. 176–177.)

**What touched it last.** A working system stays working until an external force acts on it.
A configuration change is such a force. Recent changes are a productive place to start.
(SRE Ch. 12, p. 177.) Annotate the error graph with the start and the end of each push.
(SRE Ch. 12, p. 178.) Catalog entry `I-37` names the missing change log.

**The change log must also answer "no change".** The App Engine team eliminated change as a
cause, because the latency rose on a Saturday and the last pushes finished days earlier.
(SRE Ch. 12, p. 184.)

### A response that exonerates a tier

Make each answer carry its own provenance. Google's `X-Request-Trace` header lists the
addresses of the backends that answered. The presence of the header proved that the request
reached the backends. That single fact discounted the frontend and the load balancers.
(SRE Ch. 12, pp. 176, 178.) Ask the vendor for the same. Generate one identifier at the edge
and propagate it through every hop. (SRE Ch. 12, p. 186.) Catalog entry `I-36` names the
defect.

### The hops of a typical vendor call

| Hop | Probe that tests this hop | What a pass proves |
|---|---|---|
| Our handler | Call the client with a stub in place of the vendor | Our code produces the request |
| Our client and pool | Read the pool metrics and the thread dump | The pool has free capacity |
| Our egress route | Probe the vendor from the same host | The route accepts the connection |
| The middlebox | Capture packets on both sides | The packets cross |
| The vendor edge | Probe from a second network location | The vendor answers someone |
| The vendor operation | Probe each operation separately | That operation works |

**Tracing has a named limit.** Dapper traces remote calls only. A 250-millisecond stall
inside a process produced no trace data. A method that reads only inter-service traces will
report "no dependency is slow" while the caller is the fault. (SRE Ch. 12, p. 185.)

---

## 7. The known-good probe

A known-good probe is a request whose correct answer you already know. Nygard calls this a
synthetic transaction. (Release It! Ch. 13, p. 231.)

**Define the probe before the outage.** Release It! lists what the definition must state.
(Release It! Ch. 13, pp. 231–232.)

1. What the probe sends.
2. The maximum acceptable response time for each step.
3. The response codes or the text patterns that mean success.
4. The response codes or the text patterns that mean failure.
5. How often the probe runs.
6. How many locations the probe runs from.
7. Where the results are recorded.
8. The formula that computes the availability percentage.

**Store the failing answer, and give the probe a designated monitoring user.** SRE's own
prober recorded success and failure and nothing else. The responder had to repeat the call
by hand to learn that the answer was a 502 with no body. A remediation loop cannot classify
a failure that it never captured. (SRE Ch. 12, pp. 175–176.) A probe with no designated user
pollutes production data. (Release It! Ch. 13, p. 231.) Catalog entry `I-25` names the
defect.

**Run the probe from the vantage point of the real caller.** A firewall rule that permits
one address makes a probe from your workstation fail, although the same probe from the
application host would succeed. (SRE Ch. 12, p. 179.)

**Run the probe on the path that serves work, and give it its own timeout.** A check served
by a separate thread pool stays green through a total outage. (Release It! Ch. 2, p. 31.) A
check with no timeout hangs exactly when it matters. (Release It! Sec. 4.1, p. 55.) Catalog
entry `I-24` names the defect.

**A partial fault has its own signature.** About half of the probes succeeded over ten
minutes, with no discernible pattern. That is a partial vendor fault. (SRE Ch. 12, p. 175.)

---

## 8. The vendor status page against your own signal

| Signal | What it proves | What it does not prove |
|---|---|---|
| The process is running | The process exists | Nothing about transactions. Release It! Sec. 4.5, p. 82 |
| A port accepts a connection | A listener exists | Nothing about the work path |
| A static status endpoint returns 200 | One thread pool answers | Nothing about the pool that serves work. Release It! Ch. 2, p. 31 |
| The vendor's status page is green | The vendor believes it is healthy | Nothing about our traffic, our route, or our account. **Modern** |
| The vendor's alerts are quiet | Nothing, when their alarms are untuned | Nothing. Release It! Ch. 16, p. 261 |
| Our probe succeeds from two locations | The whole path worked for that request | Nothing about the tail |
| Per-operation error rate and latency | The shape of the fault | Nothing about the cause. SRE Ch. 12, p. 175 |
| A thread dump | The layer that holds the threads | Nothing about why that layer is slow. Release It! Ch. 2, p. 29 |
| Packet capture on both sides | Whether the packets crossed | Nothing about application state. Release It! Sec. 4.1, p. 52 |

**The status page is corroboration, never evidence.** Vendor status pages are **Modern**.
Neither book names them. Measure the vendor with your own synthetic transaction, which
Release It! does prescribe. (Release It! Ch. 13, pp. 231-232.) Use their page to confirm what
you already measured. Catalog entry `I-26` names the defect. A green page and a failing
probe leave the fault unresolved. Both sides remain possible.

**Vendor alarms are not your signal.** In the Black Friday outage the vendor's on-call engineer
ignored pages about a 100% CPU condition. That group receives false alarms for transient CPU
spikes. (Release It! Ch. 16, p. 261.)

**A latency metric that omits timed-out calls is wrong.** Requests that never finished never
entered the average. A total outage then reads as a mild slowdown. (Release It! Ch. 16,
p. 259.) Catalog entry `I-33` names the defect.

**Every component can be up while the user result is wrong.** This happens with blocked
threads and with cascading failure. A monitoring system reports the system's view of itself.
(Release It! Ch. 17, p. 287.) Read `09-vendor-slas-and-degradation.md` for the availability
arithmetic across a chain of dependencies.

---

## 9. Design the test that separates two hypotheses

SRE gives the worked pair. To separate a failed network from a database that refuses
connections, connect with the application host's credentials, and ping the database host.
(SRE Ch. 12, p. 179.)

| Rule | Reason | Source |
|---|---|---|
| An ideal test has mutually exclusive alternatives | It admits one group and eliminates another | SRE Ch. 12, p. 179 |
| Run the tests in decreasing order of likelihood | Consider the obvious first | SRE Ch. 12, p. 179 |
| Weigh the risk each test creates for the system | An active test changes production | SRE Ch. 12, p. 179 |
| Search for the confounding factor | A firewall rule can invalidate the result | SRE Ch. 12, p. 179 |
| Treat an active test as a change | More CPU speeds work and raises the data-race rate | SRE Ch. 12, pp. 179–180 |
| Accept a suggestive result when a definite one is impossible | A race resists a timely repeat | SRE Ch. 12, p. 180 |
| Record every idea, every test, and every result | You can restore the pre-test state | SRE Ch. 12, p. 180 |

**Verbose logging is an active test.** It can worsen the latency problem that you measure.
After that you cannot tell whether the fault grew on its own. (SRE Ch. 12, pp. 179–180.)
Read `08-fault-injection-and-test-harness.md` before you inject a fault into production.

---

## 10. Two traps

**Prefer the common cause.** SRE quotes the medical rule. "When you hear hoofbeats, think of
horses not zebras." (SRE Ch. 12, p. 171.) Occam's Razor gives the same direction.

**Then accept several small causes.** Hickam's dictum is the counterweight. A set of common
low-grade faults can explain the symptoms better than one rare fault. (SRE Ch. 12, fn. 62,
p. 187.) A single cause may not exist. (SRE Ch. 12, p. 182.)

**Correlation is not causation.** Two metrics can move together because they share a cause.
As the metric count grows, coincidental correlations become certain. (SRE Ch. 12, p. 171.)
A remediation built on a temporal correlation becomes superstition at machine speed. Catalog
entry `I-48` names the defect.

**Identify the source before you restart a process.** A restart can amplify a fault. A cold
cache, a crash loop, and a poisoned dependency all get worse. (SRE Ch. 22, p. 340.) Catalog
entry `I-46` names the defect.

---

## 11. The war stories

**The airline grounded by one exception (Release It! Ch. 2, pp. 21–34).** Every check-in
kiosk in the country went red at 2:30 a.m., and the voice response servers followed.
Suspicion settled on a database failover two hours earlier. That failover was the trigger
and not the cause. Two thread dumps localized the fault. The kiosk dump showed forty threads
inside a socket read, waiting for an answer that never arrived. The dump of the shared
service showed every thread blocked on a database connection checkout. Monitoring stayed
green, because the monitor probed a status page that a separate thread pool served. The case
proves three rules. Blocked in a socket read means the layer below does not answer. Blocked
on pool checkout means the pool at this layer is exhausted. A health check on the wrong pool
proves nothing.

**Black Friday and the third hop (Release It! Ch. 16, pp. 256–263).** A retailer lost about
a million dollars an hour while rolling restarts failed. The vital signs were low CPU and
most threads busy, on the front end and on the back end. That identical pair, one hop apart,
located the fault. The front end held 3,000 request threads blocked on a connection pool
with no timeout. The order system held 450 threads blocked inside calls to a scheduling
system that served about 25 concurrent requests and received about 90. The fix was a
throttle on the pool for that one vendor, not a deploy. The average latency had hidden the
outage, because timed-out requests never entered it.

**The 5 a.m. problem (Release It! Sec. 4.1, pp. 50–56).** Thirty application server
instances hung within a five-minute window at almost exactly 5 a.m. each day. Thread dumps
put every request thread inside the database driver. Packet capture then showed almost
nothing on the wire in either direction, while the database was demonstrably healthy. The
absence of packets was the finding. The connection pool served the quiet night from one
connection, and thirty-nine sat idle past the firewall's one-hour idle timeout. The firewall
discarded those entries and sent no reset packet. The stack then retransmitted into a black
hole. The case proves that a middlebox can invalidate a connection without either endpoint
learning of it.

**Shakespeare search (SRE Ch. 12, pp. 173–178).** A black-box probe reported no search results
for five minutes. The alert filed a bug with links to the probe results and to the playbook
entry. Over ten minutes about half of the once-a-minute probes succeeded, with no discernible
pattern. The prober did not store what the failed calls returned, so the responder repeated one
call by hand and saw a 502 with no body. The answer carried an `X-Request-Trace` header that
named the backends. The presence of that header proved the request had reached the backends,
which discounted the frontend and the load balancers in one step. The case proves that a probe
must store the failing answer, and that an answer which names its own path exonerates whole
tiers.

**The App Engine latency mystery (SRE Ch. 12, pp. 182–186).** A customer reported latency up
by nearly an order of magnitude, with no code change and no traffic increase. The team found
a correlation with a rise in a datastore call that usually indicates poor indexing, and
started to build composite indices. Tracing then showed that requests for static content,
which never touch the datastore, were also much slower. That single fact exposed the
correlation as spurious and the index work as wasted. The real cause was an access-control
defect that stored one object per access. The case proves that a plausible mechanism does
not make a correlation causal.

**Voodoo operations (Release It! Ch. 17, pp. 281–283).** Nygard watched an administrator
receive a page and immediately start a production database failover. The message that
triggered it was a debug line that Nygard had written himself. It reported that an encrypted
channel to a vendor needed a reset, and the application reset the channel by itself. The
message had nothing to do with the database. Six months earlier that line had happened to be
the last thing logged before a database crash. There was a temporal connection and no causal
connection. The team then ran weekly production failovers during peak hours for six months.
The case proves that every automated trigger needs a stated mechanism.

---

## 12. What to record while you work

Fill the Localization Aids section of the Integration Record with these five items.

1. **The correlation identifier.** The field name, where we set it, and where we log it.
2. **The known-good probe.** What it calls, and the locations it runs from.
3. **The change log.** Where it lives, and what it records at each layer.
4. **The two latency numbers.** Caller-observed latency and vendor-reported latency.
5. **The evidence that names the network.** The capture point, or the probe on a second
   route.

The signal set per dependency is fixed. Expose the breaker state, the timeout count, the request
count, and the average response time. Expose the good-answer count, the network-error count, the
protocol-error count, and the application-error count. Expose the address of the remote
endpoint, the concurrent request count, and its high-water mark. (Release It! Ch. 17, p. 298.)
Catalog entry `I-35` names the missing set.

**Three error classes, not one.** Network errors, protocol errors, and application errors
are separate counters. That split is the axis that localizes the fault. (Release It! Ch. 17,
p. 298.) The observability plane must never stop the serving plane. (Ch. 17, p. 302.)

**Mitigation before diagnosis is legitimate, and definite proof is often unavailable.** SRE
moved the application to a larger machine, reached acceptable latency, and continued the
analysis afterward. (SRE Ch. 12, p. 185.) Accept a probable causal factor when production
cannot repeat the fault. Then write the post-mortem, and act on the items it names.
(SRE Ch. 12, p. 182. SRE Ch. 13, p. 198.)

---

## Cross-references

| Question | File |
|---|---|
| What the six failure classes of a call look like | `01-integration-points.md` |
| The name of the failure you observe | `02-stability-antipatterns.md` |
| The defense to apply after you localize the fault | `03-stability-patterns.md` |
| Quotas, throttles, criticality, and retry budgets | `05-overload-and-load-shedding.md` |
| The fault crossed a layer boundary | `06-cascading-failure.md` |
| The rules for acting without a human | `07-automated-remediation-safety.md` |
| Producing these faults on demand | `08-fault-injection-and-test-harness.md` |
| The availability arithmetic and the degraded mode | `09-vendor-slas-and-degradation.md` |
| The `I-` codes named in this file | `integration-fault-catalog.md` |
